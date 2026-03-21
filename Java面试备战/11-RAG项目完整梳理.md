# RAG 项目完整梳理 — 面试深挖专用

> 基于你的实际代码逐行分析，覆盖架构、每个组件的实现细节、设计决策和面试追问。
> 建议先通读一遍建立全局视角，再针对薄弱环节重点记忆。

---

## 一、30 秒项目介绍（开场白）

> 这个项目是电网预警规则的意图解析引擎。核心目标是把业务人员用自然语言描述的预警规则（比如"断路器SF6化学试验超期8年生成工单"），转换成结构化的业务 DSL，后续交给执行引擎去做设备预警和工单生成。
>
> 我负责整个后端的核心链路设计和开发，包括规则语料清洗、Slot 抽取、混合检索、DSL 生成和多层校验。技术栈是 Python + FastAPI + PostgreSQL(pgvector)。线上部署目标是 `192.168.0.219`，当前在线容器是 `rag-backend`、`rag-frontend`、`rag-embedding`、`rag-postgres`。

---

## 二、系统架构全景

```
用户输入："断路器SF6化学试验超期8年，35kV电压，生成工单"
                            │
                    ┌───────▼────────┐
                    │  Step 0: 澄清恢复 │ ← 如果上一轮有待澄清（选设备/选指标），先消费
                    └───────┬────────┘
                    ┌───────▼────────┐
                    │  Step 1: 文本标准化 │ ← 全角→半角、去噪声、场景特定清洗
                    └───────┬────────┘
                    ┌───────▼────────┐
                    │  Step 2: 多源抽取  │ ← 正则 + AC自动机 + 歧义消解
                    └───────┬────────┘
                    ┌───────▼────────┐
                    │  Step 3: LLM Slot │ ← LLM 汇总正则/词典结果，输出槽位
                    │         组装+校验  │    SlotValidator 检查完整性
                    └───────┬────────┘
                            │
                    ┌── 不完整？──→ 返回 need_clarification（选设备/选指标/不支持）
                    │
                    ┌───────▼────────┐
                    │  Step 4: 混合检索  │ ← BM25 + pgvector 并行 → RRF 融合
                    └───────┬────────┘
                    ┌───────▼────────┐
                    │  Step 5: DSL 生成  │ ← LLM + Few-shot 范文 → 业务 DSL
                    └───────┬────────┘
                    ┌───────▼────────┐
                    │  Step 6: DSL 校验  │ ← 结构/语义/引用完整性校验 + 自动修复
                    └───────┬────────┘
                    ┌───────▼────────┐
                    │  Step 7: 回译确认  │ ← DSL → 自然语言摘要，等用户确认
                    └───────┬────────┘
                            │
              用户确认 ──→ confirmed（保存）
              用户拒绝 ──→ revise_requested（负例 + 语料降权 → 重新生成）
```

**四个 API 端点：**

| 端点 | 方法 | 作用 |
|------|------|------|
| `/health` | GET | 健康检查（DB + LLM + Embedding 三路探活） |
| `/api/v1/kb/summary` | GET | 知识库统计（实体/场景/范文数量） |
| `/api/v1/intent` | POST | 核心：自然语言 → DSL（8步 Pipeline） |
| `/api/v1/confirm` | POST | 用户确认/拒绝 DSL |

### 2.1 仓库里能直接拿来当“项目证据”的点

- 编排入口就在 `backend/src/pipeline/orchestrator.py` 的 `PhaseAOrchestrator`，构造函数里直接挂了 `TextNormalizer`、`AliasScanner`、`RegexExtractor`、`EntityDisambiguator`、`LLMSlotAssembler`、`SlotValidator`、`VectorSearch`、`HybridRanker`、`DSLGenerator`、`DSLValidator`、`BackTranslator`、`ConfirmationHandler`
- 会话模型 `backend/src/models/session.py` 明确保留了 `pending_clarification`、`negative_examples`、`negative_constraints`、`deprioritized_corpus_ids`、`retry_count`、`max_retries`
- 部署文档写明当前目标服务器是 `192.168.0.219`，在线容器是 `rag-backend`、`rag-frontend`、`rag-embedding`、`rag-postgres`
- 回归命令不是手工试几个 case，而是 `make slot-regression-bundle`、`make slot-dsl-smoke`、`make slot-phase1-report`
- 当前回归口径是 `42` 条 existing cases 加 `26` 条 markdown cases 汇成 `68` 条基线
- `backend/tests/` 当前有 `49` 个 pytest 文件，覆盖 unit / integration / regression / smoke

---

## 三、知识库体系（5 层 KB）

```
KB1: 实体表 (KBEntityField)         — 17 种电力设备（断路器、主变、GIS...）
KB2: 场景表 (KBScene)               — 18 个监测场景（SF6试验、缺陷查询、大电流冲击...）
KB3: 场景表/关联表 (KBSceneTable/Join) — 每个场景对应的物理表和 JOIN 链
KB4: 语料库 (KB4Corpus)             — 规则范文（用户输入 + 业务DSL + 向量），用于 RAG 检索
KB5: 指标库 (KB5Metric/Stat/Entity/Scenario) — 100+ 指标定义，含统计类型和适用范围
```

**数据一致性保障：**
- `canonical_kb.py` 是种子数据（硬编码），作为 Single Source of Truth
- DB 可以新增数据，但种子数据优先
- `KBRepository._merge_*_with_seed()` 做合并去重

**面试回答要点**：为什么要种子数据 + DB 双源？
> 电网领域的核心实体和场景相对稳定，硬编码保证不会因为 DB 误操作导致系统崩溃。同时 DB 允许运维人员增量添加新指标和语料，不需要改代码重新部署。

---

## 四、Step 2 多源 Slot 抽取 — 最核心的设计

### 4.1 第一层：正则抽取（RegexExtractor）

从文本中抽取结构化信息，6 类模式：

| 类型 | 正则示例 | 输入 | 输出 |
|------|---------|------|------|
| 电压范围 | `(\d+)\s*kV(及以上\|及以下)` | "110kV及以上" | `{base:"110kV", bound:"gte", values:["110","220","500","1000"]}` |
| 电压 | `(\d+)\s*kV` | "35kV" | `{voltage:"35kV"}` |
| 次数 | `(\d+)\s*次` | "3次" | `{value:3, unit:"次"}` |
| 时长 | `(近\|超)?(\d+)(年\|个月\|天)` | "超8年" | `{value:8, unit:"年", qualifier:"超"}` |
| 倍数 | `(\d+)\s*倍(\d+次)?` | "5倍1次" | `{multiple:5, count:1}` |
| 数值阈值 | `(大于等于\|不低于\|...)(\d+)(℃\|%\|A)` | "大于等于90℃" | `{op:"大于等于", value:90, unit:"℃"}` |

**支持中文数字转换**："十五" → 15，"五十" → 50

### 4.2 第二层：别名词典扫描（AliasScanner）

基于字符串匹配（类似 AC 自动机思路），扫描三类别名：

```python
# 实体别名（来自 KB1）
"断路器" → entity: breaker
"开关"   → entity: breaker
"SF6开关" → entity: breaker（priority=5，比"开关"优先）
"GIS"    → entity: composite_apparatus
"CT"     → entity: current_transformer

# 场景提示词（来自 KB2 的 keywords）
"化学试验" → scenario_hint: sf6_test
"缺陷"    → scenario_hint: defect_query
"带电检测" → scenario_hint: gis_detection

# 动作别名
"工单"   → action: generate_work_order
"告警"   → action: generate_alarm
"预警"   → action: generate_warning
```

**排序规则**：按 `(start_position, -priority)` 排序，优先匹配更长的别名。

### 4.3 第三层：歧义消解（EntityDisambiguator）

当多个候选命中时，根据文本距离、共现模式、实体特异性选最优。

**例子**："开关柜SF6气体" —— "开关" 匹配 breaker，"开关柜" 匹配 switch_cabinet。
消歧器根据长度优先选 switch_cabinet。

### 4.4 第四层：LLM Slot 组装（LLMSlotAssembler）

把前三层结果打包发给 LLM 做最终判断：

```python
# System Prompt
"你是电力设备规则意图识别助手。请输出 JSON，字段固定为 entity, scenario, metrics, scope_hints, conditions, action。只保留有把握的信息。"

# User Prompt（JSON 格式）
{
  "user_input": "断路器SF6化学试验超期8年，35kV，生成工单",
  "ac_hits": [
    {"alias":"断路器", "type":"entity", "value":"breaker", "priority":3},
    {"alias":"化学试验", "type":"scenario_hint", "value":"sf6_test", "priority":4}
  ],
  "regex_hits": [
    {"type":"duration", "value":{"value":8, "unit":"年", "qualifier":"超"}},
    {"type":"voltage", "value":"35kV"}
  ],
  "disambiguation": {"entity":"breaker", "scenario":"sf6_test"}
}
```

- temperature=0（确定性输出）
- 超时 8 秒，超时 fallback 到纯规则结果

**LLM 输出后的关键处理：**

1. **指标标准化**：`_normalize_metrics()` 把 LLM 输出的指标名映射到标准知识库
2. **指标适用性过滤**：检查指标是否适用于当前实体+场景
3. **时间绑定**：`_apply_time_bindings()` 把正则抽出的时长绑定到对应指标的 time 字段
4. **多实体检测**：`MULTI_ENTITY_CONNECTORS = ("、", "和", "及", "与", "或")` 切分输入识别多设备

**面试追问：为什么不全交给 LLM？**
> 三个原因：
> 1. 正则和词典对结构化信息（数字、电压）的抽取准确率接近 100%，LLM 可能会算错数字
> 2. 分层后 LLM 只需要做语义判断（"8年"是修饰试验超期还是设备运行年限），降低了任务难度
> 3. 正则和词典的延迟是毫秒级，LLM 是秒级。如果 LLM 超时，还能 fallback 到纯规则结果

---

## 五、Step 3 Slot 校验与澄清机制

### 5.1 校验逻辑（SlotValidator）

按优先级依次检查：

```
1. 实体是否存在？
   ├── 有 multi_entity_candidates → 触发 entity_multi_select 澄清
   ├── 有 entity_candidates → 触发 entity_choice 澄清
   └── 都没有 → 触发 entity_fill 澄清（推荐兼容实体）

2. 场景是否合法？
   └── 缺失或不合法 → 错误

3. 实体-场景是否兼容？
   └── 不兼容 → 触发 entity_multi_select 或 entity_fill

4. 指标是否适用于实体+场景？
   └── 不适用 → 澄清

5. 指标是否为空？
   └── 空 → 触发 metric_fill 澄清（推荐指标卡片）

6. 是否有不支持的能力？（同比增长/环比等）
   └── 有 → 触发 unsupported_capability 澄清（建议改写）

7. 自动修复：
   - 缺少 action → 默认 generate_warning
   - 缺少 scope_hints → 默认添加 "运行状态=在运, 资产性质≠用户"
```

### 5.2 五种澄清类型

| 类型 | 触发条件 | 用户交互 | 示例 |
|------|---------|---------|------|
| `entity_choice` | 2个候选实体 | 单选 | "适用于断路器还是隔离开关？" |
| `entity_multi_select` | 多个候选实体 | 多选 | "适用于哪些设备？（可多选）" |
| `entity_fill` | 无实体识别 | 建议列表 | "请明确设备对象" |
| `metric_fill` | 有场景无指标 | 指标卡片 | "请选择要关注的指标" |
| `unsupported_capability` | 检测到同比/环比 | 改写建议 | "不支持同比增长，建议改为..." |

### 5.3 多轮对话状态管理（SessionContext）

```python
session_context = {
    "session_id": "uuid",
    "pending_clarification": {          # 上一轮待澄清的信息
        "clarification_type": "entity_multi_select",
        "source_user_input": "原始输入",
        "entity_candidates": ["breaker", "isolate_switch"],
        "slot_seed": {...},             # 锁定的 Slot 快照
        "scenario_ranked": [...]        # 场景候选排序
    },
    "deprioritized_corpus_ids": ["r1", "r2"],  # 被拒绝的语料 ID
    "negative_examples": [{"dsl":{...}, "user_feedback":"不对"}],
    "negative_constraints": [{"field":"action", "op":"neq", "value":"generate_work_order"}],
    "retry_count": 1,
    "max_retries": 2,
    "last_parse_record_id": "uuid",
    "last_feedback_record_id": "uuid"
}
```

**Step 0 澄清消费流程**（`_resolve_pending_entity_fill()`）：

1. 获取 `pending_clarification` 的类型
2. 对用户输入做清洗：去掉前缀（"我选"、"选择"、"就要"）和后缀（"。"、"呀"）
3. 按类型匹配：
   - entity_choice/fill：别名匹配用户输入 → 锁定实体
   - entity_multi_select：逗号/空格分割 → 多实体；"全选" → 所有候选
   - metric_fill：精确匹配 suggested_input → 激活 slot_seed
   - unsupported_capability：精确匹配改写建议 → 激活 slot_seed
4. 返回 `{entity, target_entities, slot_seed, source_user_input}`
5. 后续 Step 3 使用锁定的实体和 slot_seed，跳过歧义阶段

---

## 六、Step 4 混合检索 — 核心创新点

### 6.1 BM25 检索

```python
class BM25Retriever:
    def __init__(self, examples):
        # 每条范文拼接：rule_name + rule_text + tags
        corpus = [tokenize_zh(compose_doc(ex)) for ex in examples]
        self._bm25 = BM25Okapi(corpus)  # rank_bm25 库

def tokenize_zh(text):
    # 双层分词：词级 + 字级
    cleaned = re.sub(r"[^\w\u4e00-\u9fff]+", " ", text)
    tokens = cleaned.split()                              # 词级：["SF6", "泄漏", "预警"]
    char_tokens = [c for c in text if "\u4e00" <= c <= "\u9fff"]  # 字级：["泄","漏","预","警"]
    return tokens + char_tokens
```

**为什么要字级分词？**
> 电网术语没有标准分词词典，"化学试验"可能被分成"化学"+"试验"或不分。加上字级 token 保证即使分词不准，单字也能匹配上。

### 6.2 pgvector 向量检索

```sql
SELECT id, entity_type, scenario, business_dsl, user_input,
       1 - (embedding_vec <=> CAST(:embedding AS vector)) AS similarity
FROM kb4_corpus
WHERE is_active = TRUE
  AND entity_type = :entity_type    -- 精确过滤
  AND scenario = :scenario_filter    -- 精确过滤
ORDER BY embedding_vec <=> CAST(:embedding AS vector)
LIMIT :top_k;
```

- `<=>` 是 pgvector 的 cosine distance 运算符
- **先过滤后排序**：利用 SQL WHERE 条件精确缩小范围，再做向量排序
- 索引：`USING ivfflat (embedding_vec vector_cosine_ops)`

### 6.3 RRF 混合排序（HybridRanker）

```python
class HybridRanker:
    def __init__(self, k=60):
        self.k = k

    def rank(self, vector_results, bm25_results, deprioritized_ids=None, top_n=3):
        scores = {}
        # 两路分别计算 RRF 分数
        for rank, item in enumerate(vector_results):
            scores[item.rule_id] = scores.get(item.rule_id, 0) + 1.0 / (self.k + rank + 1)
        for rank, item in enumerate(bm25_results):
            scores[item.rule_id] = scores.get(item.rule_id, 0) + 1.0 / (self.k + rank + 1)

        # 负例降权
        for rule_id in (deprioritized_ids or []):
            if rule_id in scores:
                scores[rule_id] *= 0.3  # 降为 30%

        # 排序取 Top-N
        ranked = sorted(scores.items(), key=lambda x: x[1], reverse=True)
        return [record_map[rid] for rid, _ in ranked[:top_n]]
```

**计算示例：**
```
假设 k=60：
Vector 结果：[A(rank0), C(rank1), D(rank2)]
BM25 结果： [A(rank0), B(rank1), C(rank2)]

A 的分数：1/61 + 1/61 = 0.0328（两路都排第一，得分最高）
C 的分数：1/62 + 1/63 = 0.0320
B 的分数：0 + 1/62 = 0.0161（只在 BM25 中）
D 的分数：1/63 + 0 = 0.0159（只在 Vector 中）

最终排序：A > C > B > D
```

**负例降权机制：**
- 用户拒绝某个 DSL → 该 DSL 对应的语料范文 ID 加入 `deprioritized_corpus_ids`
- 下次检索时该 ID 的分数 × 0.3
- 效果：从 Top-3 掉到 Top-10 之外，不会再被选为 few-shot 范文

---

## 七、Step 5-6 DSL 生成与校验

### 7.1 DSL 生成

**Prompt 结构：**
```json
{
  "user_input": "断路器SF6化学试验超期8年，35kV，生成工单",
  "slots": {
    "entity": "breaker",
    "scenario": "sf6_test",
    "metrics": [{"name":"最近SF6化学试验时间", "stat":"max"}],
    "scope_hints": [{"field":"电压等级", "op":"eq", "value":"35kV"}],
    "action": "generate_work_order"
  },
  "corpus_examples": [
    {"rule_id":"ex1", "rule_name":"SF6化学试验超期工单", "dsl_json":{...}},
    {"rule_id":"ex2", "rule_name":"SF6压力低预警", "dsl_json":{...}}
  ],
  "scene_details": {"tables":[...], "joins":[...]},
  "constraints": {"only_business_dsl": true, "max_nesting_depth": 3}
}
```

- System Prompt：`"你是电力设备业务 DSL 生成器。请输出严格 JSON，不要输出 markdown。"`
- temperature=0.1
- 超时 12 秒

**Fallback 生成（LLM 失败时）：**
```python
def _fallback(slots, user_input, corpus_examples, scene):
    # 1. 从第一个范文借 action
    reference_action = corpus_examples[0].dsl_json["rules"][0]["then"]["action"]

    # 2. 从 slots 构建 metrics
    metrics = [{"id":"m1", "name": slot.name, "stat": slot.stat or "exists"} ...]

    # 3. 构建最小 rule
    rule = {
        "when": {"metric":"m1", "op":"exists"},
        "then": {"action": reference_action, "message": "基于用户输入触发规则"}
    }

    # 4. 组装完整 DSL
    return {"entity": slots.entity, "scenario": slots.scenario, "metrics": metrics, "rules": [rule]}
```

### 7.2 DSL 结构（BusinessDSL）

```json
{
  "entity": "breaker",                    // 或 target_entities: ["breaker","isolate_switch"]
  "scenario": "sf6_test",
  "scope_filters": [
    {"field": "电压等级", "op": "eq", "value": "35kV"},
    {"field": "运行状态", "op": "eq", "value": "在运"}
  ],
  "metrics": [
    {
      "id": "m1",
      "name": "最近SF6化学试验时间",
      "canonical_metric_name": "最近SF6化学试验时间",
      "stat": "max",
      "time": {"range": "8y", "anchor": "recent", "type": "trailing_window"}
    }
  ],
  "rules": [
    {
      "id": "r1",
      "when": {
        "op": "and",
        "args": [
          {"metric": "m1", "op": "gt", "value": 8, "window": "year"},
          {"field": "灭弧介质", "op": "eq", "value": "SF6"}
        ]
      },
      "then": {
        "action": "generate_work_order",
        "message": "断路器SF6化学试验超期8年，请安排检修工单"
      }
    }
  ],
  "output": {"priority": "normal"}
}
```

### 7.3 DSL 校验（DSLValidator）

7 层校验：

| 校验项 | 规则 | 自动修复 |
|--------|------|---------|
| 顶层字段 | 只允许 entity/target_entities/scenario/scope_filters/metrics/rules/output | 删除非法字段 |
| entity 互斥 | entity 和 target_entities 不能同时存在 | - |
| entity 合法性 | 必须在标准知识库中（multi_equipment 特殊允许） | - |
| scenario 合法性 | 必须在标准知识库中 | - |
| metrics | 非空，每个指标有 id/name/stat，指标名在 KB 中，适用于当前实体+场景 | 标准化名称、补充 canonical_metric_name |
| rules | 非空，有 when/then，条件嵌套 ≤ 3 层，metric 引用存在 | - |
| scope_filters | field 在语义字段白名单中，op 合法 | - |

---

## 八、Step 7 回译 + 确认/拒绝

### 8.1 回译（BackTranslator）

```python
def translate(self, dsl):
    entity = self._entity_label(dsl.get("entity"))       # "breaker" → "断路器"
    scenario = self._scene_label(dsl.get("scenario"))     # "sf6_test" → "SF6化学试验超期"
    metrics = [m.get("name") for m in dsl.get("metrics")]
    rule_count = len(dsl.get("rules", []))

    return f"这条规则面向 {entity}，场景是 {scenario}，包含 {rule_count} 条规则，关注指标为 {', '.join(metrics)}。"
```

### 8.2 确认处理（ConfirmationHandler）

```python
# 用户说"确认"/"可以"/"ok" → status: confirmed → 保存 DSL
# 用户说"不对"/"修改"/"no"  → status: revise_requested → 负例入库 + 语料降权
# 用户说其他              → status: awaiting_confirmation → 追问
```

**拒绝后的处理链路：**
1. 当前 DSL 加入 `negative_examples`
2. 对应的检索范文 ID 加入 `deprioritized_corpus_ids`
3. `retry_count + 1`
4. 下次 `/intent` 调用时：
   - 检索阶段：被拒范文分数 × 0.3
   - 生成阶段：`session_context` 包含 negative_examples，LLM 可以参考避免

---

## 九、降级与容错设计

| 组件 | 超时 | 降级策略 |
|------|------|---------|
| LLM Slot 组装 | 8 秒 | 回退到纯正则+词典结果 |
| DSL 生成 | 12 秒 | 从 Slot 构建最小 DSL + 范文 action |
| 向量检索 | 30 秒 | 降级为空结果，只用 BM25 |
| Embedding | 30 秒 | 向量检索整体降级 |
| 数据库 | - | 健康检查报 degraded |

**健康检查三路探活：**
```python
async def health_probe(self):
    # DB: SELECT 1
    # LLM: chat("只回复ok") → 检查回复是否为 "ok"
    # Embedding: embed("健康检查") → 检查向量非空
    # 全部 ok → "ok"，任一失败 → "degraded"
```

---

## 十、测试体系

```
tests/
├── unit/                    # 单元测试（无外部依赖）
│   ├── test_regex_extractor    → 正则模式覆盖
│   ├── test_alias_scanner      → 别名匹配
│   ├── test_hybrid_ranker      → RRF 排序逻辑
│   ├── test_back_translator    → 6 个回译用例
│   ├── test_confirmation       → 14 个确认/拒绝分类
│   ├── test_slot_validator     → 校验逻辑分支
│   └── test_dsl_validator      → DSL 语义校验
├── integration/             # 集成测试（需要 DB/LLM）
│   ├── test_slot_pipeline      → 标准化→AC→正则→消歧 全链路
│   ├── test_retrieval_pipeline → BM25+Vector+Hybrid 全链路
│   ├── test_api_routes         → REST 接口契约
│   └── test_confirmation_flow  → 多轮对话
├── regression/              # 回归测试
│   └── test_slot_acceptance    → 68 条基线用例（核心指标必须全部命中）
└── smoke/                   # 冒烟测试
    ├── test_orchestrator_dsl_smoke   → 全 Pipeline 端到端
    ├── test_frontend_smoke_cases     → 前端 UI 场景
    └── test_real_corpus_smoke_cases  → 真实语料
```

**回归验证集：**
- 42 条 existing cases
- 26 条 markdown cases
- 合并后形成 68 条槽位基线（核心 entity + scenario + metrics 必须全部匹配）
- 49 个 pytest 文件

---

## 十一、面试高频追问 & 参考答案

### Q1：为什么选 pgvector 不选 Milvus/Chroma？

> 三个原因：
> 1. **已有 PG 基建**：项目数据库就是 PostgreSQL，加个 pgvector 扩展零成本
> 2. **数据规模匹配**：语料库几百到几千条，pgvector 完全够用，Milvus 适合亿级
> 3. **SQL 原生过滤**：可以在 WHERE 子句中做 entity_type + scenario 精确过滤后再做向量排序，Milvus 的 metadata filtering 没这么灵活
> 4. **部署约束**：电网内网环境，多一个组件就多一套审批流程

### Q2：RRF 的 k=60 怎么调的？

> k 控制排名差异的平滑程度。k 越大，高排名和低排名的分数差越小。
> 实际在 20/40/60/80/100 五个值上做了实验，用 68 条 acceptance cases 的 Top-3 命中率作为评估指标，k=60 在我们的语料规模下效果最好。
> 直觉上：k=60 意味着排名第 1 和第 60 的分数差约 2 倍（1/61 vs 1/121），足够区分但不至于让某一路完全主导。

### Q3：Slot 抽取的准确率多少？怎么评估的？

> 用 68 条 slot acceptance cases 做回归测试。评估标准分三级：
> - **通过**：entity + scenario + metrics 核心字段全部匹配
> - **部分通过**：核心字段匹配，scope_hints/conditions 缺失
> - **不通过**：核心字段有误
>
> 更准确地说，当前口径是 42 条 existing cases 加 26 条 markdown cases 汇成 68 条基线。每次代码变更都跑全量回归，保证不退步。

### Q4：DSL 两层设计的意义？

> Business DSL 是业务层面的抽象（"什么实体、什么场景、什么指标、什么条件"），不涉及具体的 SQL 和物理表。
> Execution DSL（Phase B，不在我负责范围内）负责把 Business DSL 翻译成具体的 SQL JOIN 查询。
>
> 分层的好处：
> 1. 业务人员可以看懂 Business DSL 并确认
> 2. 底层表结构变化不影响 Business DSL
> 3. LLM 只需要生成业务层面的规则，不需要理解数据库 schema，减少出错概率

### Q5：遇到过什么难 case？

**Case 1：别名冲突**
> "开关柜SF6气体" —— "开关" 匹配 breaker，"开关柜" 匹配 switch_cabinet。
> 解决：AliasScanner 按别名长度排序（priority=len(alias)），更长的别名优先。

**Case 2：条件语义歧义**
> "断路器运行年限超过15年且SF6泄漏超标" —— "运行年限超过15年"是设备属性过滤（scope_filter），不是指标条件。
> 解决：正则层把"15年"抽为 duration hit，LLM Slot 组装时 prompt 明确区分"指标阈值"和"设备属性"。

**Case 3：不支持的能力**
> "断路器缺陷同比增长5%以上生成告警" —— "同比增长"是当前不支持的计算类型。
> 解决：`unsupported_capability` 检测逻辑（正则检测"同比/环比" + "增长/下降" + 百分比），返回改写建议。

### Q6：场景特定的文本清洗是什么？

> 某些场景有特殊的噪声模式：
> - `short_circuit_impact`（大电流冲击）：去掉油色谱试验的尾部描述
> - `switch_record`（投切记录）：去掉频率约束的冗余描述
>
> 在 Step 1 文本标准化之后、Step 2 之前，根据初步场景判断做二次清洗。这样后续的正则和 LLM 不会被噪声干扰。

### Q7：怎么保证 LLM 输出的 JSON 格式正确？

> 1. System Prompt 明确要求"输出严格 JSON，不要 markdown"
> 2. 用 `chat_json()` 方法调用，底层做了 JSON parse
> 3. 如果 parse 失败，走 fallback 逻辑（从 Slot 构建最小 DSL）
> 4. DSL Validator 做最后一道校验，确保结构合法

### Q8：私有化部署怎么做的？

> Docker 4 容器：
> 1. **rag-backend**：FastAPI + uvicorn
> 2. **rag-postgres**：PostgreSQL + pgvector
> 3. **rag-embedding**：Embedding 服务
> 4. **rag-frontend**：Vue 3 静态资源，容器内 Nginx 代理 `/api/*` 和 `/health`
>
> 部署目标服务器是 `192.168.0.219`，环境变量配置 LLM 和 Embedding 地址，前端容器默认走同源代理。

### Q9：如果让你优化，你会改什么？

> 1. **引入 Rerank 模型**：目前 RRF 是基于排名的简单融合，可以加 Cross-encoder Rerank 进一步提升检索精度
> 2. **流式输出**：DSL 生成目前是等 LLM 全部输出后再返回，可以改为 SSE 流式，提升用户体验
> 3. **更精细的分词**：目前 BM25 用的是简单分词 + 字级 token，可以引入 jieba 或领域词典做更准确的分词
> 4. **评估自动化**：目前 68 条 case 是手工标注，可以建立自动评估 Pipeline + 指标看板

---

## 十二、完整数据流示例

**输入**："SF6开关化学试验超期8年怎么预警？35kV电压开关"

```
Step 0: 无待澄清 → 跳过

Step 1 标准化:
  "SF6开关化学试验超期8年怎么预警？35kV电压开关"
  → "SF6开关化学试验超期8年怎么预警35kV电压开关"（去标点）

Step 2 多源抽取:
  AC hits:
    - {type:"entity", value:"breaker", alias:"开关", priority:2, start:3}
    - {type:"scenario_hint", value:"sf6_test", alias:"化学试验", priority:4, start:5}
  Regex hits:
    - {type:"duration", value:{value:8, unit:"年"}, position:11}
    - {type:"voltage", value:"35kV", position:17}
  消歧结果:
    - entity: breaker, scenario: sf6_test

Step 3 LLM Slot 组装:
  SlotOutput:
    entity: "breaker"
    scenario: "sf6_test"
    metrics: [{name:"最近SF6化学试验时间", canonical:"最近SF6化学试验时间", stat:"max"}]
    scope_hints: [{field:"电压等级", op:"eq", value:"35kV"}]
    conditions: [{threshold:{type:"duration", value:"8y"}}]
    action: "generate_warning"

  Slot 校验: ✓ 通过
  自动修复: 补充 scope_hints: 运行状态=在运, 资产性质≠用户

Step 4 混合检索:
  BM25 Top-5: [ex_sf6_overdue, ex_sf6_leak, ex_sf6_pressure, ...]
  Vector Top-5: [ex_sf6_overdue, ex_chem_test, ex_sf6_leak, ...]
  RRF Top-3: [ex_sf6_overdue(两路都第一), ex_sf6_leak, ex_chem_test]

Step 5 DSL 生成:
  LLM 输入: user_input + slots + 3条范文 + scene_details
  LLM 输出: {entity:"breaker", scenario:"sf6_test", metrics:[...], rules:[...]}
  DSLNormalizer 标准化

Step 6 DSL 校验:
  ✓ entity breaker 合法
  ✓ scenario sf6_test 适用于 breaker
  ✓ 指标 "最近SF6化学试验时间" 适用于 sf6_test
  ✓ rules 引用的 m1 存在
  自动修复: 补充 canonical_metric_name

Step 7 回译:
  "这条规则面向 断路器，场景是 SF6化学试验超期，包含 1 条规则，关注指标为 最近SF6化学试验时间。"

返回: status="awaiting_confirmation", business_dsl={...}, back_translation="..."
```
