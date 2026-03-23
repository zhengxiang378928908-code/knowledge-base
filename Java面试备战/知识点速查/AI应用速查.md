# AI 应用开发（Agent / Multi-Agent / RAG 工程化）

## Agent 核心概念

### 什么是 AI Agent

Agent = LLM + 工具调用 + 记忆 + 规划

```
用户指令
  → Agent（LLM 作为大脑）
    → 思考：理解任务，拆解步骤
    → 行动：调用工具（API / 数据库 / 搜索）
    → 观察：获取工具返回结果
    → 循环：直到任务完成
```

**和普通 LLM 调用的区别**：普通调用是一问一答，Agent 能自主规划、多步执行、使用工具、根据中间结果调整策略。

### Agent 四大核心能力

| 能力 | 说明 | 例子 |
|------|------|------|
| **规划（Planning）** | 把复杂任务拆解为子步骤 | ReAct、Chain of Thought、Task Decomposition |
| **工具使用（Tool Use）** | 调用外部 API / 工具 | Function Calling、MCP |
| **记忆（Memory）** | 短期（对话上下文）+ 长期（向量库存储） | 对话历史、用户画像 |
| **行动（Action）** | 根据规划和观察执行操作 | 调 API、写文件、发消息 |

### ReAct 框架（面试必知）

```
Thought: 我需要查一下用户的订单状态
Action: 调用订单查询 API，参数 order_id=12345
Observation: 订单状态为"已发货"，物流单号 SF123456
Thought: 用户还需要物流信息，我再查一下物流
Action: 调用物流查询 API，参数 tracking_id=SF123456
Observation: 包裹在北京中转站
Thought: 信息够了，可以回复用户
Answer: 您的订单已发货，目前在北京中转站...
```

**面试话术**：ReAct 是 Reasoning + Acting 的结合，LLM 在每一步先思考要做什么，再调用工具执行，拿到结果后继续思考，循环直到得出最终答案。和 CoT 的区别是 ReAct 多了工具调用，不只是推理。

## Function Calling（工具调用）

### 原理

```
1. 开发者定义工具的 JSON Schema（函数名、参数、描述）
2. 用户提问 + 工具定义一起发给 LLM
3. LLM 判断是否需要调用工具，返回函数名和参数
4. 开发者执行函数，把结果回传给 LLM
5. LLM 基于工具结果生成最终回答
```

### 代码示例

```python
tools = [{
    "type": "function",
    "function": {
        "name": "query_device_status",
        "description": "查询电力设备的运行状态",
        "parameters": {
            "type": "object",
            "properties": {
                "device_id": {"type": "string", "description": "设备编号"},
                "device_type": {"type": "string", "enum": ["breaker", "transformer"]}
            },
            "required": ["device_id"]
        }
    }
}]

response = client.chat.completions.create(
    model="gpt-4",
    messages=messages,
    tools=tools,
    tool_choice="auto"  # LLM 自行判断是否调用
)
```

### MCP（Model Context Protocol）

- Anthropic 提出的开放协议，统一 Agent 与外部工具/数据的对接方式
- 类似 USB 接口标准：一个协议接入所有工具，不用每个工具单独适配
- MCP Server 暴露工具 → Agent 通过 MCP Client 调用

## Multi-Agent（多智能体）

### 单 Agent vs Multi-Agent

| 对比项 | 单 Agent | Multi-Agent |
|--------|----------|-------------|
| 架构 | 一个 LLM 处理所有任务 | 多个专职 Agent 协作 |
| 适用场景 | 简单任务、单一领域 | 复杂任务、跨领域协作 |
| Prompt 管理 | 一个大 Prompt | 每个 Agent 独立 Prompt，职责清晰 |
| 可维护性 | Prompt 膨胀后难维护 | 各 Agent 独立迭代 |
| 容错 | 一步错全错 | 单个 Agent 失败可重试或替换 |

### 常见多 Agent 协作模式

**1. 顺序链式（Pipeline）**
```
Agent A（信息收集）→ Agent B（分析）→ Agent C（生成报告）
```
适用：流程固定、步骤明确的任务

**2. 主从模式（Orchestrator + Workers）**
```
Orchestrator（编排者）
  ├── Worker A（搜索）
  ├── Worker B（计算）
  └── Worker C（写作）
```
适用：需要动态决定调用哪个子 Agent

**3. 辩论/协商模式（Debate）**
```
Agent A（正方）⇄ Agent B（反方）→ Judge Agent（裁决）
```
适用：需要多角度分析的决策场景

**4. 分组协作（Group Chat）**
```
多个 Agent 在群聊中讨论，Manager 协调发言顺序
```
适用：开放式问题、头脑风暴

### 你项目的 Agent 思路

你的 RAG 项目虽然没有显式用 Multi-Agent 框架，但本质上是 **Pipeline 模式的多步 Agent 编排**：

```
文本标准化 Agent → Slot 抽取 Agent（正则+AC+LLM）
  → 校验 Agent（完整性检查，不完整则澄清）
  → 检索 Agent（BM25 + 向量混合检索）
  → DSL 生成 Agent（LLM + Few-shot）
  → 校验 Agent（结构/语义/引用校验）
  → 回译确认
```

**面试话术**：我的项目是一个多步编排的 AI Pipeline，每个步骤有独立的职责和降级策略。虽然没有用 LangGraph 这类框架，但核心思路和 Multi-Agent 的 Pipeline 模式一致——每个环节专注一件事，有独立的输入输出和异常处理，而不是把所有逻辑塞进一个大 Prompt。

## 主流 Agent / Multi-Agent 框架

| 框架 | 特点 | 适用场景 |
|------|------|----------|
| **LangChain** | 最流行的 LLM 应用框架，链式调用 | 通用 RAG、简单 Agent |
| **LangGraph** | LangChain 团队出品，图结构编排 Agent | 复杂多步骤、有条件分支的 Agent |
| **CrewAI** | 角色扮演式多 Agent，定义角色+任务+工具 | 团队协作类任务 |
| **AutoGen (微软)** | 多 Agent 对话框架 | 代码生成、多轮协商 |
| **Dify** | 低代码 AI 应用平台 | 快速搭建 RAG / Agent 应用 |
| **Coze (字节)** | 低代码 Bot 平台 | 快速搭建聊天机器人 |

### LangGraph 示例（了解即可）

```python
from langgraph.graph import StateGraph

# 定义状态
class AgentState(TypedDict):
    messages: list
    next_step: str

# 构建图
graph = StateGraph(AgentState)
graph.add_node("researcher", research_agent)
graph.add_node("writer", writer_agent)
graph.add_edge("researcher", "writer")
graph.set_entry_point("researcher")

app = graph.compile()
result = app.invoke({"messages": [user_input]})
```

## RAG 工程化要点（面试加分）

### 优化方向

| 阶段 | 优化点 | 方案 |
|------|--------|------|
| 索引 | 分块策略 | 语义分块 > 固定长度；注意 overlap |
| 索引 | 元数据增强 | 给 chunk 加标签、来源、时间等过滤字段 |
| 检索 | 混合检索 | BM25（关键词）+ 向量（语义）→ RRF 融合 |
| 检索 | Query 改写 | HyDE（假设性文档嵌入）、多查询扩展 |
| 排序 | Rerank | Cross-encoder 重排序，提升 Top-K 精度 |
| 生成 | Prompt 工程 | Few-shot、角色设定、输出格式约束 |
| 评估 | 自动评估 | Recall@K、MRR、RAGAS 框架 |

### Query 改写（高级技巧）

```
用户原始问题："断路器怎么维护"
  ↓
HyDE：LLM 先生成一段假设性回答 → 用回答去检索（语义更丰富）
多查询：拆成多个子问题 → 分别检索 → 合并结果
```

### Agentic RAG（Agent + RAG 结合）

```
用户提问
  → Agent 判断：是否需要检索？检索哪个知识库？
  → 检索结果不够好？→ Agent 自动改写 query 重试
  → 检索到了？→ Agent 判断是否足够回答
  → 不够？→ Agent 调用其他工具补充信息
  → 生成最终回答
```

**面试话术**：传统 RAG 是固定流程，Agentic RAG 让 Agent 来决定要不要检索、怎么检索、结果够不够好，有自主判断和重试能力。我的项目里的 Slot 校验和多轮澄清就体现了这个思路——不够完整就主动要求补充，而不是硬生成。

## 高频面试题

1. **什么是 AI Agent？和普通 LLM 调用的区别？**（Agent = LLM + 工具 + 记忆 + 规划，能自主多步执行）
2. **ReAct 框架是什么？**（Thought → Action → Observation 循环，推理和行动交替）
3. **Function Calling 的原理？**（定义工具 Schema → LLM 决定调用 → 执行后结果回传 → LLM 生成最终回答）
4. **单 Agent 和 Multi-Agent 的区别？什么时候用多 Agent？**（任务复杂、跨领域、需要分工协作时用多 Agent）
5. **常见的多 Agent 协作模式？**（Pipeline、主从编排、辩论、群聊）
6. **RAG vs Fine-tuning 怎么选？**（知识更新频繁 / 需要可解释性用 RAG，特定风格生成用 Fine-tuning）
7. **RAG 检索效果不好怎么优化？**（混合检索、Query 改写、Rerank、分块策略优化）
8. **如何评估 RAG 系统？**（检索：Recall@K / MRR；生成：准确率 / 忠实度）
9. **你的项目和 Agent 有什么关系？**（Pipeline 式多步编排，每步独立职责+降级，本质是 Agent 思路）
10. **什么是 Agentic RAG？**（Agent 自主决定是否检索、如何检索、是否重试，比固定流程更智能）
