# RAG 与 AI 应用开发

> 投 AI 应用岗时重点准备，结合你的 DSL 意图引擎项目。

---

## 一、RAG 完整链路

### 1.1 什么是 RAG

**Retrieval-Augmented Generation（检索增强生成）**：
- 用户提问 → 检索相关文档 → 把文档作为上下文喂给 LLM → LLM 生成答案
- 解决 LLM 的知识截止、幻觉、领域知识不足等问题

### 1.2 标准 RAG 流程

```
用户输入
   ↓
1. 文本预处理（清洗、标准化）
   ↓
2. 分块（Chunking）
   ↓
3. Embedding（向量化）
   ↓
4. 向量检索（Retrieval）
   ↓
5. [可选] Rerank（重排序）
   ↓
6. 构建 Prompt（Context + Question）
   ↓
7. LLM 生成回答
   ↓
8. [可选] 后处理（校验、格式化）
```

### 1.3 你项目中的 RAG 变体

你的项目不是标准 QA 型 RAG，而是**结构化生成型 RAG**：

```
用户自然语言规则
   ↓
1. 文本标准化（全角→半角，场景特定噪声去除）
   ↓
2. 多源 Slot 抽取（正则 + AC 自动机 + LLM）
   ↓
3. Slot 校验 + 澄清
   ↓
4. 混合检索（BM25 + pgvector 向量）→ 召回相似规则范文
   ↓
5. RRF 混合排序
   ↓
6. DSL 生成（LLM + Few-shot 范文）
   ↓
7. DSL 校验（结构/语义/引用完整性）
   ↓
8. 回译确认
```

**与标准 RAG 的区别：**
- 输出不是自然语言，而是结构化 DSL
- 检索的不是文档片段，而是规则范文（包含完整 DSL 示例）
- 多了 Slot 抽取 + 校验的前处理阶段
- 多了 DSL 校验 + 回译的后处理阶段

---

## 二、分块策略（Chunking）

### 2.1 常见分块方式

| 方式 | 原理 | 适用场景 |
|------|------|---------|
| 固定长度 | 按字符数/Token 数切分 | 通用文本 |
| 句子分割 | 按句号/换行切分 | 自然语言文档 |
| 段落分割 | 按段落切分 | 结构化文档 |
| 语义分块 | 相邻句子 embedding 相似度 < 阈值时切分 | 高质量要求 |
| 递归分割 | 先按大分隔符，不够再按小分隔符 | LangChain 默认 |

### 2.2 分块参数

- **chunk_size**：每块大小（通常 500-1500 tokens）
- **chunk_overlap**：块之间重叠（通常 chunk_size 的 10-20%）
- 重叠的目的：避免关键信息被切断

### 2.3 你项目的特殊性

你的项目中每条语料是一条完整的规则范文，不需要传统的文档分块：
```python
# 每条 corpus 就是一个完整的"chunk"
doc = f"{example.rule_name} {example.rule_text} {' '.join(example.search_tags)}"
```
这种领域特定的语料管理方式比通用分块更精确。

---

## 三、Embedding（向量化）

### 3.1 什么是 Embedding

将文本转换为固定维度的稠密向量，语义相似的文本在向量空间中距离更近。

### 3.2 常见 Embedding 模型

| 模型 | 维度 | 特点 |
|------|------|------|
| text-embedding-ada-002 (OpenAI) | 1536 | 通用性好 |
| text-embedding-3-small (OpenAI) | 512-1536 | 性价比高，支持维度缩减 |
| bge-large-zh (BAAI) | 1024 | 中文效果好，开源 |
| m3e (Moka) | 768 | 中文开源 |

### 3.3 你项目中的 Embedding

```python
# 使用 OpenAI 兼容接口，1024 维向量
embedding_vec = await embedding_client.embed(text)
# 存入 pgvector
# embedding_vec vector(1024)
```

---

## 四、向量数据库

### 4.1 常见向量数据库对比

| 数据库 | 特点 | 适用场景 |
|--------|------|---------|
| **pgvector** | PG 扩展，SQL 集成 | 中小规模，已有 PG 基建 |
| Milvus | 分布式，高性能 | 大规模（亿级向量） |
| Chroma | 轻量，Python 友好 | 原型开发、小规模 |
| Pinecone | 全托管云服务 | 不想运维 |
| Weaviate | GraphQL 接口，模块化 | 多模态检索 |
| Qdrant | Rust 实现，高性能 | 中大规模 |

### 4.2 为什么选 pgvector

- 项目已经用了 PostgreSQL，无需引入新组件
- 规则范文数量在几百到几千级别，pgvector 完全够用
- SQL 原生支持，可以用 WHERE 条件做精确过滤 + 向量近似搜索
- 部署简单（电网内网环境限制）

```sql
-- pgvector 的优势：可以同时做精确过滤和向量搜索
SELECT * FROM kb4_corpus
WHERE is_active = TRUE
  AND entity_type = 'breaker'       -- 精确过滤
  AND scenario = 'sf6_test'          -- 精确过滤
ORDER BY embedding_vec <=> :query    -- 向量排序
LIMIT 5;
```

### 4.3 向量索引类型

| 索引 | 原理 | 特点 |
|------|------|------|
| IVFFlat | 聚类后在最近的几个聚类中搜索 | 构建快，精度好 |
| HNSW | 多层图结构，从顶层向下搜索 | 查询快，内存大 |
| 暴力搜索 | 遍历所有向量 | 精确，数据量小时可用 |

你项目用的 IVFFlat：
```sql
CREATE INDEX idx_kb4_embedding ON kb4_corpus
USING ivfflat (embedding_vec vector_cosine_ops);
```

---

## 五、检索策略

### 5.1 稀疏检索（BM25）

BM25 是经典的关键词匹配算法：

```
score(D, Q) = Σ IDF(qi) × (f(qi, D) × (k1 + 1)) / (f(qi, D) + k1 × (1 - b + b × |D|/avgdl))
```

- IDF：逆文档频率，越罕见的词权重越高
- TF：词频，出现次数越多权重越高（但有饱和）
- 文档长度归一化

### 5.2 稠密检索（向量检索）

- 语义层面的相似度匹配
- 可以匹配"同义不同词"

### 5.3 混合检索（你的方案）

**Reciprocal Rank Fusion（RRF）：**

```python
# 两路检索结果融合
for rank, item in enumerate(vector_results):
    scores[item.id] += 1.0 / (k + rank + 1)

for rank, item in enumerate(bm25_results):
    scores[item.id] += 1.0 / (k + rank + 1)

# 两路都靠前的文档得分最高
# k=60 是平滑常数
```

**你的额外创新：负例惩罚**
```python
# 用户拒绝的 DSL 对应的语料范文，下次检索时打折
for rule_id in deprioritized_ids:
    scores[rule_id] *= 0.3  # 70% 惩罚
```

### 5.4 Rerank（重排序）

你的项目没用 Rerank 模型，但可以作为优化方向。常见 Rerank 方案：
- **Cross-encoder**：把 query 和 document 拼接输入 BERT，精度高但慢
- **BGE-reranker**：开源 Rerank 模型
- **Cohere Rerank**：商业 API

---

## 六、Prompt Engineering

### 6.1 基本原则

1. **明确角色**：`"你是电力设备规则意图识别助手"`
2. **明确输出格式**：`"请输出 JSON，不要输出 markdown"`
3. **提供约束**：`"只保留有把握的信息"`
4. **Few-shot 示例**：提供 2-3 个高质量示例

### 6.2 你项目中的 Prompt 设计

**Slot 抽取 Prompt：**
```
System: "你是电力设备规则意图识别助手。请输出 JSON，字段固定为 entity, scenario, metrics, scope_hints, conditions, action。只保留有把握的信息。"

User: {
  "user_input": "断路器SF6泄漏年超过50mL",
  "ac_hits": [...],       // 词典匹配结果
  "regex_hits": [...],    // 正则匹配结果
  "disambiguation": {...} // 消歧结果
}
```

**DSL 生成 Prompt：**
```
System: "你是电力设备业务 DSL 生成器。请输出严格 JSON，不要输出 markdown，不要解释。只生成 Phase A business DSL。"

User: {
  "user_input": "...",
  "slots": {...},            // 已抽取的槽位
  "corpus_examples": [...],  // 检索到的范文（Few-shot）
  "constraints": {
    "only_business_dsl": true,
    "max_nesting_depth": 3
  }
}
```

### 6.3 温度设置策略

| 场景 | Temperature | 原因 |
|------|------------|------|
| Slot 抽取 | 0 | 需要确定性，不要创造性 |
| DSL 生成 | 0.1 | 近乎确定，允许极小灵活性 |
| 健康检查 | 0 | 最小化响应 |

### 6.4 超时与降级策略

| 组件 | 超时时间 | Fallback |
|------|---------|----------|
| Slot Assembly | 8 秒 | 回退到纯规则结果（正则+词典） |
| DSL Generation | 12 秒 | 根据 Slot 构建最小化 DSL |
| Embedding | 30 秒 | 向量检索降级为空 → 只用 BM25 |
| LLM Chat | 30 秒 | 返回 error 状态 |

---

## 七、幻觉问题

### 7.1 什么是幻觉

LLM 生成看似合理但事实上不正确的内容。

### 7.2 缓解方案

| 方案 | 原理 |
|------|------|
| RAG | 提供真实文档作为上下文 |
| 低温度 | 减少随机性 |
| 结构化输出 | 限制输出格式（JSON Schema） |
| 校验层 | 对生成结果做规则校验 |
| 回译确认 | 让用户确认结果 |

### 7.3 你项目中的防幻觉机制

1. **Few-shot 范文**：检索到的真实规则范文作为模板，LLM 参考生成
2. **DSL 校验器**：检查实体、场景、指标是否在标准知识库中
3. **语义字段白名单**：只允许预定义的字段名
4. **回译确认**：用户二次确认
5. **负例反馈**：拒绝的结果进入 negative_examples，下次避免

---

## 八、AI 应用面试高频问题

### Q1：如何评估 RAG 系统的效果？

**检索质量：**
- Recall@K：正确文档是否在 Top-K 中
- MRR（Mean Reciprocal Rank）：正确文档的平均排名倒数
- NDCG：考虑排名位置的评估指标

**生成质量：**
- 准确率：生成内容是否正确
- 忠实度：生成内容是否忠于检索到的文档
- 完整度：是否回答了用户的全部问题

**你项目的评估：**
- 68 条 Slot Acceptance Cases 做 Slot 抽取回归测试
- 299 条槽位覆盖测试
- 46 个测试文件覆盖 DSL + 检索链路

### Q2：RAG vs Fine-tuning？

| | RAG | Fine-tuning |
|--|-----|-------------|
| 知识更新 | 实时（更新检索库） | 需要重新训练 |
| 成本 | 低（不需要 GPU 训练） | 高 |
| 可解释性 | 高（可以展示检索来源） | 低 |
| 领域适应 | 好（换语料即可） | 好（但需要标注数据） |
| 适用场景 | 知识密集型、FAQ | 特定风格/格式的生成 |

你的项目选 RAG 的原因：电网规则语料频繁更新，需要实时反映最新规则；同时需要可解释性（用户需要确认 DSL 的正确性）。

### Q3：如何处理长上下文？

- **Map-Reduce**：分块处理后汇总
- **Refine**：逐块迭代优化回答
- **压缩**：对检索结果做摘要后再传给 LLM
- 你的项目：限制 Top-3 范文，控制上下文长度

### Q4：LLM API 调用的工程化考量？

- 重试机制（指数退避）
- 超时控制
- 降级方案（Fallback）
- 流式输出（SSE）
- 成本控制（Token 计数、缓存相同请求）
- 异步调用（你用的 httpx.AsyncClient）
