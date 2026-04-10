+++
date = '2026-04-10T09:51:46+08:00'
title = 'Openhand_memory'
draft = false
+++

## 概述

OpenHands 的记忆系统是一个多层次的系统，负责管理 AI Agent 的信息检索和对话历史处理。它使 Agent 能够在执行任务时记住重要的上下文信息，并根据需要检索知识。

记忆系统主要由三个核心组件构成：

1. **Memory** (`openhands/memory/memory.py`) - 处理召回动作和微代理知识检索
2. **ConversationMemory** (`openhands/memory/conversation_memory.py`) - 将事件流转换为 LLM 可用的消息格式
3. **Condenser** (`openhands/memory/condenser/`) - 当对话历史过长时进行压缩和摘要

---

## 1. Recall 类型

OpenHands 定义了两种召回类型，用于区分不同来源的记忆检索：

```python
# openhands/events/recall_type.py
class RecallType(str, Enum):
    WORKSPACE_CONTEXT = 'workspace_context'  # 工作区上下文（仓库、运行时等）
    KNOWLEDGE = 'knowledge'                  # 知识微代理触发
```

| 类型 | 触发方式 | 用途 |
|------|---------|------|
| `WORKSPACE_CONTEXT` | 用户消息附带召回请求 | 获取仓库信息、运行时环境、对话指令 |
| `KNOWLEDGE` | 匹配知识微代理的关键词 | 检索用户自定义的知识库 |

---

## 2. 微代理 (Microagent) 系统

微代理是 OpenHands 记忆系统的重要组成部分，允许用户定义可被动态检索的知识片段。

### 2.1 微代理类型

```python
# openhands/microagent/types.py
class MicroagentType(str, Enum):
    KNOWLEDGE = 'knowledge'       # 关键词触发型知识
    REPO_KNOWLEDGE = 'repo_knowledge'  # 仓库级知识，始终激活
    TASK = 'task'                 # 任务型，需要用户输入
```

| 类型 | 描述 | 触发方式 |
|------|------|---------|
| `KNOWLEDGE` | 可选微代理 | 对话中包含关键词时触发 |
| `REPO_KNOWLEDGE` | 仓库级知识 | 始终加载，提供仓库特定信息 |
| `TASK` | 任务型 | 需要用户通过 `/agent_name` 命令调用 |

### 2.2 微代理存储位置

微代理从以下位置加载：

| 位置 | 说明 |
|------|------|
| `openhands/skills/` | 全局公共微代理 |
| `~/.openhands/microagents/` | 用户自定义微代理 |
| `.openhands/microagents/repo.md` | 仓库级微代理 |
| `.cursorrules`, `AGENTS.md`, `agent.md` | 第三方兼容格式 |

### 2.3 微代理文件格式

```markdown
# openhands/microagent/microagent.py
# 知识微代理示例
name: my_knowledge
trigger: python, django, flask
---
这是一段关于 Python Web 开发的知识。

# 仓库微代理示例
name: repo_instructions
always_load: true
---
本仓库使用 Django 4.2 + Python 3.11 开发。
```

### 2.4 核心类

```python
# openhands/microagent/microagent.py
class BaseMicroagent(BaseModel):
    name: str
    content: str
    metadata: MicroagentMetadata
    source: str
    type: MicroagentType

class KnowledgeMicroagent(BaseMicroagent):
    """关键词触发的知识微代理"""
    def match_trigger(self, message: str) -> str | None:
        """检查消息是否匹配触发关键词"""

class RepoMicroagent(BaseMicroagent):
    """仓库级微代理，始终激活"""

class TaskMicroagent(KnowledgeMicroagent):
    """任务型微代理，支持变量替换"""
    def extract_variables(self, content: str) -> list[str]
    def requires_user_input(self) -> bool
```

---

## 3. Memory 组件

`Memory` 类是记忆系统的核心处理器，订阅事件流并处理召回动作。

### 3.1 核心职责

```python
# openhands/memory/memory.py
class Memory:
    sid: str
    event_stream: EventStream
    repo_microagents: dict[str, RepoMicroagent]      # 仓库微代理
    knowledge_microagents: dict[str, KnowledgeMicroagent]  # 知识微代理
    repository_info: RepositoryInfo | None           # 仓库信息
    runtime_info: RuntimeInfo | None                 # 运行时信息
    conversation_instructions: ConversationInstructions | None

    async def _on_event(self, event: Event) -> None:
        """处理事件流中的事件"""
        if isinstance(event, RecallAction):
            if event.recall_type == RecallType.WORKSPACE_CONTEXT:
                # 处理工作区上下文召回
                workspace_obs = self._on_workspace_context_recall(event)
            elif event.recall_type == RecallType.KNOWLEDGE:
                # 处理知识微代理召回
                microagent_obs = self._on_microagent_recall(event)
```

### 3.2 工作区上下文召回

当触发 `WORKSPACE_CONTEXT` 召回时，`Memory` 返回：

- **仓库信息**: 仓库名称、目录路径、分支名
- **运行时信息**: 运行时类型、自定义密钥描述、工作目录
- **对话指令**: 用户提供的对话级别指令
- **仓库微代理知识**: 所有 `REPO_KNOWLEDGE` 类型微代理的内容

### 3.3 知识微代理召回

当触发 `KNOWLEDGE` 召回时，`Memory` 会：

1. 接收用户的查询消息
2. 遍历所有已加载的知识微代理
3. 使用 `match_trigger()` 检查是否匹配触发词
4. 返回匹配微代理的 `MicroagentKnowledge` 观察结果

---

## 4. ConversationMemory 组件

`ConversationMemory` 负责将事件流转换为 LLM 可消费的 `Message` 列表。

### 4.1 处理流程

```python
# openhands/memory/conversation_memory.py
class ConversationMemory:
    def process_events(
        self,
        condensed_history: list[Event],
        initial_user_action: MessageAction,
        forgotten_event_ids: set[int] | None = None,
        max_message_chars: int | None = None,
        vision_is_active: bool = False,
    ) -> list[Message]:
        """将事件列表转换为消息列表"""
```

### 4.2 消息构建规则

| 事件类型 | 转换为 | 说明 |
|---------|-------|------|
| `MessageAction` | `UserMessage` / `AssistantMessage` | 用户或助手消息 |
| `AgentThinkAction` | `AssistantMessage` (含思考) | Agent 思考过程 |
| `ToolCall` | `AssistantMessage` (含工具调用) | 工具调用 |
| `ToolReturn` | `UserMessage` (模拟工具返回) | 工具执行结果 |
| `Observation` | `UserMessage` | 观察结果 |

### 4.3 系统消息与初始消息

- **系统消息**: 确保在消息列表开头包含系统级指令
- **初始用户消息**: 处理可能被遗忘的初始事件 ID

---

## 5. Condenser 系统 (历史压缩)

当对话历史超过 token 限制时，Condenser 系统负责压缩和精简历史。

### 5.1 Condenser 基类

```python
# openhands/memory/condenser/condenser.py
class Condenser(ABC):
    def condense(self, view: View) -> View | Condensation
    def condensed_history(self, state: State) -> View | Condensation
    def add_metadata(self, key: str, value: Any) -> None

class RollingCondenser(Condenser, ABC):
    """滚动压缩器，判断何时应该压缩"""
    def should_condense(self, view: View) -> bool
    def get_condensation(self, view: View) -> Condensation
```

### 5.2 可用 Condenser 实现

| Condenser | 说明 |
|-----------|------|
| `NoOpCondenser` | 不做任何处理 |
| `AmortizedForgettingCondenser` | 简单的半分半留遗忘 |
| `LLMSummarizingCondenser` | 基于 LLM 的文本摘要 |
| `StructuredSummaryCondenser` | LLM 结构化状态摘要（函数调用风格） |
| `ConversationWindowCondenser` | 保留关键事件 + 最近一半 |
| `ObservationMaskingCondenser` | 遮蔽旧观察结果 |
| `BrowserOutputCondenser` | 遮蔽浏览器输出 |
| `LLMAttentionCondenser` | LLM 注意力机制选择重要事件 |
| `RecentEventsCondenser` | 仅保留最近事件 |
| `CondenserPipeline` | 链式组合多个 Condenser |

### 5.3 配置示例

```python
# openhands/core/config/condenser_config.py
@dataclass
class LLMSummarizingCondenserConfig(CondenserConfig):
    type: str = "llm_summarizing"
    summarize_after_n_events: int = 50
    summary_instruction: str = "请简要总结对话历史"
```

### 5.4 Condenser  Pipeline

可以将多个 Condenser 组合成管道使用：

```python
# 示例：先摘要，再遮蔽敏感信息
pipeline = CondenserPipeline([
    LLMSummarizingCondenser(config),
    BrowserOutputCondenser(),
])
```

---

## 6. View 类

`View` 是 Condenser 操作的数据结构，表示事件列表的视图：

```python
# openhands/memory/view.py
class View(BaseModel):
    events: list[Event]
    unhandled_condensation_request: bool = False
    forgotten_event_ids: set[int] = set()

    @staticmethod
    def from_events(events: list[Event]) -> View:
        """从事件列表创建视图，处理压缩事件"""
```

---

## 7. 数据流总结

```
用户消息
    │
    ├─→ RecallAction (WORKSPACE_CONTEXT)
    │       │
    │       └─→ Memory._on_workspace_context_recall()
    │               │
    │               ├─→ 仓库信息
    │               ├─→ 运行时信息
    │               ├─→ 对话指令
    │               └─→ 仓库微代理知识
    │                       │
    │                       └─→ RecallObservation ──→ EventStream
    │
    ├─→ RecallAction (KNOWLEDGE)
    │       │
    │       └─→ Memory._on_microagent_recall()
    │               │
    │               └─→ MicroagentKnowledge ──→ EventStream
    │
    └─→ ConversationMemory.process_events()
            │
            └─→ list[Message] ──→ LLM

当历史过长时:
    │
    └─→ Condenser.condense()
            │
            └─→ Condensation / 精简后的事件列表
```

---

## 8. 自定义记忆系统

### 8.1 创建自定义知识微代理

在 `~/.openhands/microagents/` 目录下创建 `.md` 文件：

```markdown
# ~/.openhands/microagents/python_best_practices.md
name: python_best_practices
trigger: python, pip, virtualenv
---
# Python 最佳实践

## 依赖管理
- 使用 `pip freeze > requirements.txt` 锁定依赖
- 优先使用虚拟环境隔离项目依赖

## 代码规范
- 遵循 PEP 8 规范
- 使用 type hints 提升代码可读性
```

### 8.2 创建仓库级微代理

在项目根目录创建 `.openhands/microagents/repo.md`：

```markdown
# .openhands/microagents/repo.md
name: my_project_context
always_load: true
---
# 项目特定上下文

本项目使用 FastAPI + PostgreSQL 构建 RESTful API。
```

---

## 9. 相关文件索引

| 文件路径 | 说明 |
|---------|------|
| `openhands/memory/memory.py` | Memory 类主实现 |
| `openhands/memory/conversation_memory.py` | ConversationMemory 实现 |
| `openhands/memory/view.py` | View 类定义 |
| `openhands/memory/condenser/condenser.py` | Condenser 基类 |
| `openhands/memory/condenser/impl/*.py` | 各 Condenser 实现 |
| `openhands/microagent/microagent.py` | 微代理基类 |
| `openhands/microagent/types.py` | 微代理类型定义 |
| `openhands/events/recall_type.py` | RecallType 枚举 |
| `openhands/events/action/agent.py` | RecallAction 定义 |
| `openhands/events/observation/agent.py` | RecallObservation 等定义 |
| `openhands/core/config/condenser_config.py` | Condenser 配置类 |

---

## 10. 总结

OpenHands 的记忆系统通过三个核心组件协同工作：

1. **Memory** 负责从多种来源（仓库信息、运行时环境、微代理知识）检索上下文
2. **ConversationMemory** 负责将事件流转换为 LLM 可理解的连续消息
3. **Condenser** 系统负责在对话历史过长时进行智能压缩

这套设计使得 OpenHands Agent 能够：
- 动态检索用户定义的知识
- 保持长时间的跨任务对话
- 在历史过长时自动进行智能摘要和压缩
