+++
date = '2026-04-10T09:47:47+08:00'
title = 'Openhands Agent浅析'

draft = false
+++

## 目录
1. [Agent 概述](#1-agent-概述)
2. [Agent 基类设计](#2-agent-基类设计)
3. [CodeActAgent 详解](#3-codeactagent-详解)
4. [工具系统](#4-工具系统)
5. [多智能体模式](#5-多智能体模式)
6. [Skills 系统](#6-skills-系统)
7. [MCP 集成](#7-mcp-集成)
8. [Microagent 微智能体](#8-microagent-微智能体)
9. [架构图](#9-架构图)
10. [面试问题](#10-面试问题)

---

## 1. Agent 概述

### 1.1 什么是 Agent

在 OpenHands 中，**Agent（智能体）** 是整个系统的"大脑"。它负责：
- 理解用户任务
- 制定执行计划
- 生成具体动作（Action）
- 决定何时完成任务

Agent 基于大语言模型（LLM）实现，通过 `step()` 方法驱动每次决策。

### 1.2 Agent 类型

| Agent | 类 | 用途 |
|-------|-----|------|
| **CodeActAgent** | `openhands/agenthub/codeact_agent/` | 主要智能体，统一动作空间 |
| **BrowsingAgent** | `openhands/agenthub/browsing_agent/` | 网页浏览 |
| **ReadonlyAgent** | `openhands/agenthub/readonly_agent/` | 只读操作（继承 CodeActAgent） |
| **LocAgent** | `openhands/agenthub/loc_agent/` | 代码库探索 |
| **DummyAgent** | `openhands/agenthub/dummy_agent/` | 测试用 |

---

## 2. Agent 基类设计

**文件**: `/home/gdw/tutorial/OpenHands/openhands/controller/agent.py`

### 2.1 核心抽象

```python
class Agent(ABC):
    _registry: dict[str, type['Agent']] = {}  # 类注册表
    sandbox_plugins: list[PluginRequirement] = []
    config_model: type[AgentConfig] = AgentConfig

    @abstractmethod
    def step(self, state: 'State') -> 'Action':
        """执行一步智能体决策 - 子类必须实现"""
        pass
```

### 2.2 类注册模式

OpenHands 使用**类注册模式**实现 Agent 的发现和加载：

```python
class Agent(ABC):
    @classmethod
    def register(cls, name: str, agent_cls: type['Agent']) -> None:
        """注册智能体到注册表"""
        if name in cls._registry:
            raise AgentAlreadyRegisteredError(name)
        cls._registry[name] = agent_cls

    @classmethod
    def get_cls(cls, name: str) -> type['Agent']:
        """根据名称获取智能体类"""
        if name not in cls._registry:
            raise AgentNotRegisteredError(name)
        return cls._registry[name]

    @classmethod
    def list_agents(cls) -> list[str]:
        """列出所有注册的智能体"""
        return list(cls._registry.keys())
```

**使用方式**：
```python
# 注册
class MyAgent(Agent):
    pass
Agent.register('my_agent', MyAgent)

# 获取
agent_cls = Agent.get_cls('codeact_agent')
```

### 2.3 工具管理

```python
class Agent(ABC):
    def __init__(self, config: AgentConfig, llm_registry: LLMRegistry):
        self.tools: list[ChatCompletionToolParam] = []  # 内置工具
        self.mcp_tools: dict[str, ChatCompletionToolParam] = {}  # MCP 工具
        self.llm = llm_registry.get_llm(config.llm)

    def set_mcp_tools(self, mcp_tools: list[dict]) -> None:
        """设置 MCP 工具"""
        for tool in mcp_tools:
            _tool = ChatCompletionToolParam(**tool)
            self.mcp_tools[_tool['function']['name']] = _tool
            self.tools.append(_tool)
```

---

## 3. CodeActAgent 详解

### 3.1 什么是 CodeAct

**CodeAct** 源自论文 [CodeAct: Towards Unified Code Actions](https://arxiv.org/abs/2402.01030)，核心思想是：

> **将所有工具调用统一到一个"代码执行空间"**

**传统方式**：为每种操作定义独立工具（`search_web`、`read_file`、`run_command`），工具数量多且分散。

**CodeAct**：只用一个"代码执行"工具，智能体生成 Python 代码来描述操作。

### 3.2 CodeActAgent 结构

**文件**: `/home/gdw/tutorial/OpenHands/openhands/agenthub/codeact_agent/codeact_agent.py`

```python
class CodeActAgent(Agent):
    VERSION = '2.2'
    sandbox_plugins: list[PluginRequirement] = [
        AgentSkillsRequirement(),  # 提供 Python 函数库
        JupyterRequirement(),       # 交互式 Python 解释器
    ]
```

### 3.3 Step 执行流程

```python
def step(self, state: State) -> 'Action':
    # 1. 返回待处理动作（如果有）
    if self.pending_actions:
        return self.pending_actions.popleft()

    # 2. 检查是否需要压缩历史
    match self.condenser.condensed_history(state):
        case View(events=events, forgotten_event_ids=forgotten_ids):
            condensed_history = events
        case Condensation(action=condensation_action):
            return condensation_action  # 返回压缩动作

    # 3. 构建 LLM 消息
    messages = self._get_messages(condensed_history, initial_user_message, forgotten_event_ids)

    # 4. 调用 LLM
    params = {
        'messages': messages,
        'tools': check_tools(self.tools, self.llm.config),
        'extra_body': {...}
    }
    response = self.llm.completion(**params)

    # 5. 解析响应为动作
    actions = self.response_to_actions(response)
    for action in actions:
        self.pending_actions.append(action)
    return self.pending_actions.popleft()
```

### 3.4 可用工具

| 工具 | 动作类 | 功能 |
|------|--------|------|
| `bash` | `CmdRunAction` | 执行 Bash 命令 |
| `IPythonTool` | `IPythonRunCellAction` | 执行 Python 代码 |
| `BrowserTool` | `BrowseInteractiveAction` | 浏览器交互 |
| `FinishTool` | `AgentFinishAction` | 标记任务完成 |
| `ThinkTool` | `AgentThinkAction` | 记录推理过程 |
| `StrReplaceEditorTool` | `FileEditAction`/`FileReadAction` | 文件操作 |
| `TaskTrackerTool` | `TaskTrackingAction` | 任务管理 |
| `CondensationRequestTool` | `CondensationRequestAction` | 内存压缩 |

### 3.5 工具配置

工具根据配置选择性启用：

```python
def _get_tools(self) -> list['ChatCompletionToolParam']:
    tools = []
    if self.config.enable_cmd:
        tools.append(create_cmd_run_tool(...))
    if self.config.enable_think:
        tools.append(ThinkTool)
    if self.config.enable_finish:
        tools.append(FinishTool)
    if self.config.enable_browsing:
        tools.append(BrowserTool)
    if self.config.enable_jupyter:
        tools.append(IPythonTool)
    if self.config.enable_plan_mode:
        tools.append(create_task_tracker_tool(...))
    if self.config.enable_editor:
        tools.append(create_str_replace_editor_tool(...))
    return tools
```

---

## 4. 工具系统

### 4.1 工具定义模式

工具定义为 `ChatCompletionToolParam` 对象（LiteLLM 格式）：

```python
# ThinkTool 示例
ThinkTool = ChatCompletionToolParam(
    type='function',
    function=ChatCompletionToolParamFunctionChunk(
        name='think',
        description='Use the tool to think about something...',
        parameters={
            'type': 'object',
            'properties': {
                'thought': {'type': 'string', 'description': 'The thought to log.'},
            },
            'required': ['thought'],
        },
    ),
)
```

### 4.2 动作解析

**文件**: `/home/gdw/tutorial/OpenHands/openhands/agenthub/codeact_agent/function_calling.py`

```python
def response_to_actions(response: ModelResponse, mcp_tool_names: list[str] | None = None) -> list[Action]:
    """将 LLM 响应转换为 Action 列表"""
    for tool_call in response.tool_calls:
        if tool_call.function.name == 'bash':
            action = CmdRunAction(command=arguments['command'], ...)
        elif tool_call.function.name == 'IPythonTool':
            action = IPythonRunCellAction(code=arguments['code'])
        elif tool_call.function.name == 'finish':
            action = AgentFinishAction()
        # ... 更多工具
    return actions
```

### 4.3 动作响应流程

```
LLM 响应 (ModelResponse)
    │
    ▼
response_to_actions()
    │
    ├── tool_call.name == 'bash'
    │       └── CmdRunAction(command='ls -la')
    ├── tool_call.name == 'IPythonTool'
    │       └── IPythonRunCellAction(code='print("hello")')
    └── tool_call.name == 'finish'
            └── AgentFinishAction()
    │
    ▼
返回 Action 列表
    │
    ▼
AgentController 执行 Action
    │
    ▼
Runtime 返回 Observation
```

---

## 5. 多智能体模式

### 5.1 委托机制

OpenHands 支持**多智能体协作**，通过 `AgentDelegateAction` 实现：

**文件**: `/home/gdw/tutorial/OpenHands/openhands/controller/agent_controller.py`

```python
async def start_delegate(self, action: AgentDelegateAction) -> None:
    """启动委托智能体处理子任务"""
    # 1. 创建被委托的智能体
    agent_cls: type[Agent] = Agent.get_cls(action.agent)
    delegate_agent = agent_cls(config=agent_config, llm_registry=self.agent.llm_registry)

    # 2. 创建共享状态的委托状态
    state = State(
        session_id=self.id.removesuffix('-delegate'),
        inputs=action.inputs or {},
        delegate_level=self.state.delegate_level + 1,
        metrics=self.state.metrics,  # 共享指标
        # ...
    )

    # 3. 创建委托控制器
    self.delegate = AgentController(
        sid=self.id + '-delegate',
        agent=delegate_agent,
        event_stream=self.event_stream,  # 共享事件流！
        initial_state=state,
        is_delegate=True,
        # ...
    )
```

### 5.2 委托动作

```python
@dataclass
class AgentDelegateAction(Action):
    agent: str          # 委托目标，如 'BrowsingAgent'
    inputs: dict        # 任务输入
    action: str = ActionType.DELEGATE
```

### 5.3 多智能体通信模式

**共享 EventStream 模式**：

```
┌──────────────────────────────────────────────────────────────┐
│                      EventStream                            │
│  (共享消息总线)                                              │
└──────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ Parent Agent    │  │ Delegate Agent  │  │    Runtime      │
│ Controller      │  │ Controller      │  │                 │
│                 │  │                 │  │                 │
│ - 发起委托      │  │ - 处理子任务    │  │ - 执行动作      │
│ - 等待结果      │  │ - 返回结果      │  │ - 返回观察结果  │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

**特点**：
1. 父和子**共享同一个 EventStream**
2. Metrics（指标）**共享**
3. `delegate_level` 递增，用于嵌套深度限制
4. 委托结果通过 `AgentDelegateObservation` 返回

### 5.4 委托结束流程

```python
def end_delegate(self) -> None:
    # 从委托者复制迭代计数
    self.state.iteration_flag.current_value = (
        self.delegate.state.iteration_flag.current_value
    )

    # 发送委托结果观察
    obs = AgentDelegateObservation(
        outputs=delegate_outputs,
        content=content
    )
    self.event_stream.add_event(obs, EventSource.AGENT)

    self.delegate = None
```

### 5.5 多智能体设计模式

| 模式 | 说明 | 使用场景 |
|------|------|----------|
| **父子委托** | Parent 创建 Delegate 处理子任务 | 任务分解 |
| **专家路由** | 根据任务类型路由到不同 Agent | 专业分工 |
| **层级嵌套** | 多层委托形成树形结构 | 复杂任务分解 |
| **并行执行** | 多个 Agent 同时处理不同子任务 | 效率提升 |

---

## 6. Skills 系统

### 6.1 什么是 Skills

**Skills** 是 OpenHands 提供给智能体的 **Python 函数库**，通过 Jupyter/IPython 环境暴露给 Agent 调用。

**文件**: `/home/gdw/tutorial/OpenHands/openhands/runtime/plugins/agent_skills/`

### 6.2 Skill 模块

| 模块 | 功能 |
|------|------|
| `file_ops` | 文件操作（open_file, goto_line, scroll_down, search_dir） |
| `file_reader` | 文件读取工具 |
| `repo_ops` | Git 仓库操作 |
| `file_editor` | 高级文件编辑 |

### 6.3 Skill 使用方式

```python
# file_ops.py
def open_file(path: str, line_number: int | None = 1, context_lines: int = 100) -> None:
    """打开文件并可选定位到指定行"""
    global CURRENT_FILE, CURRENT_LINE, WINDOW
    CURRENT_FILE = os.path.abspath(path)
    # ... 实现

# Agent 通过 IPython 调用
action = IPythonRunCellAction(code='open_file("src/main.py", line_number=50)')
```

### 6.4 Skill 与 Tool 的区别

| 维度 | Tool | Skill |
|------|------|-------|
| **定义位置** | Agent._get_tools() | runtime/plugins/agent_skills/ |
| **暴露方式** | LLM function calling | Python 函数 |
| **调用方式** | LLM 生成 tool_call | Agent 执行 IPython 代码 |
| **灵活性** | 固定参数 | 任意 Python 代码 |

---

## 7. MCP 集成

### 7.1 什么是 MCP

**MCP (Model Context Protocol)** 是一种扩展智能体工具能力的协议。

**文件**: `/home/gdw/tutorial/OpenHands/openhands/mcp/`

### 7.2 MCPClient

```python
class MCPClient(BaseModel):
    """连接到 MCP 服务器的工具集合"""
    client: Optional[Client] = None
    tools: list[MCPClientTool] = Field(default_factory=list)
    tool_map: dict[str, MCPClientTool] = Field(default_factory=dict)
```

### 7.3 MCP 传输类型

| 传输类型 | 配置类 | 使用场景 |
|----------|--------|----------|
| `StdioTransport` | `MCPStdioServerConfig` | 本地进程 |
| `SSETransport` | `MCPSSEServerConfig` | Server-Sent Events |
| `StreamableHttpTransport` | `MCPSHTTPServerConfig` | HTTP 流式传输 |

### 7.4 MCPAction

**文件**: `/home/gdw/tutorial/OpenHands/openhands/events/action/mcp.py`

```python
@dataclass
class MCPAction(Action):
    name: str                              # MCP 工具名称
    arguments: dict[str, Any] = field(default_factory=dict)
    thought: str = ''
    action: str = ActionType.MCP
    runnable: ClassVar[bool] = True
    security_risk: ActionSecurityRisk = ActionSecurityRisk.UNKNOWN
```

### 7.5 MCP 工具添加到 Agent

```python
# Agent 初始化时
self.set_mcp_tools(mcp_tools)  # 从配置加载 MCP 工具
```

---

## 8. Microagent 微智能体

### 8.1 概述

**Microagent** 是一种轻量级的**专门知识智能体**，可在对话期间被触发。

**文件**: `/home/gdw/tutorial/OpenHands/openhands/microagent/microagent.py`

### 8.2 Microagent 类型

| 类型 | 触发方式 | 使用场景 |
|------|----------|----------|
| `REPO_KNOWLEDGE` | 始终激活 | 代码库特定指南 |
| `KNOWLEDGE` | 关键词触发 | 语言/框架最佳实践 |
| `TASK` | 斜杠命令（`/{name}`） | 需要用户输入的任务 |

### 8.3 基类结构

```python
class BaseMicroagent(BaseModel):
    name: str
    content: str
    metadata: MicroagentMetadata
    source: str
    type: MicroagentType

class KnowledgeMicroagent(BaseMicroagent):
    """通过关键词触发"""
    def match_trigger(self, message: str) -> str | None:
        message = message.lower()
        for trigger in self.triggers:
            if trigger.lower() in message:
                return trigger
        return None

class TaskMicroagent(KnowledgeMicroagent):
    """通过 /{name} 触发，需要用户输入"""
    def requires_user_input(self) -> bool:
        variables = self.extract_variables(self.content)
        return len(variables) > 0
```

### 8.4 MicroagentMetadata

```python
class MicroagentMetadata(BaseModel):
    name: str = 'default'
    type: MicroagentType = MicroagentType.REPO_KNOWLEDGE
    version: str = '1.0.0'
    agent: str = 'CodeActAgent'
    triggers: list[str] = []       # KNOWLEDGE 类型的关键词
    inputs: list[InputMetadata] = []  # TASK 类型的输入
    mcp_tools: MCPConfig | None = None  # 额外的 MCP 工具
```

---

## 9. 架构图

### 9.1 Agent 类层次结构

```
                    <<ABC>>
                     Agent
            ┌─────────────────┐
            │ - _registry     │
            │ - tools[]       │
            │ - mcp_tools{}   │
            │ - llm           │
            ├─────────────────┤
            │ + step(state)   │ ← 抽象方法
            │ + reset()       │
            │ + name          │
            │ + complete      │
            │ + set_mcp_tools │
            └────────┬────────┘
                     │
     ┌───────────────┼───────────────┬───────────────┬──────────────┐
     │               │               │               │              │
     ▼               ▼               ▼               ▼              ▼
CodeActAgent   BrowsingAgent   ReadonlyAgent   DummyAgent     LocAgent
 (CodeAct)        (Web)        (Read-only)     (Testing)    (Code Search)
     │               │               │
     │          uses:           extends:
     │     HighLevelActionSet   CodeActAgent
     │          from             but overrides:
     │       BrowserGym          _get_tools()
     ▼                           (read-only tools)
tools/
  ├─ bash.py          → CmdRunAction
  ├─ ipython.py       → IPythonRunCellAction
  ├─ str_replace_     → FileEditAction/
    editor.py            FileReadAction
  ├─ browser.py       → BrowseInteractiveAction
  ├─ finish.py        → AgentFinishAction
  ├─ think.py         → AgentThinkAction
  ├─ task_tracker.py  → TaskTrackingAction
  └─ condensation_    → CondensationRequestAction
     request.py
```

### 9.2 Agent 执行流程

```
用户任务
    │
    ▼
┌─────────────────────────────────────┐
│       AgentController               │
│  - 维护 State                       │
│  - 管理主循环                        │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│         CodeActAgent.step()         │
│                                     │
│  1. 检查 pending_actions            │
│  2. 压缩历史（如果需要）            │
│  3. 构建 LLM 消息                    │
│  4. 调用 LLM                        │
│  5. 解析响应为 Actions              │
└─────────────────────────────────────┘
    │
    ▼
Actions: [CmdRunAction, FileReadAction, ...]
    │
    ▼
┌─────────────────────────────────────┐
│         EventStream.add_event()     │
│  - 发布 Action 到事件流             │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│            Runtime                  │
│  - 执行 Action                      │
│  - 返回 Observation                 │
└─────────────────────────────────────┘
    │
    ▼
Observation → AgentController → Agent.step()
```

---

## 10. 面试问题

### Q1: OpenHands 有哪些类型的 Agent？CodeActAgent 有什么特点？

**参考答案**：
> OpenHands 内置多种 Agent：
> - **CodeActAgent**：主要智能体，采用 CodeAct 方法论
> - **BrowsingAgent**：网页浏览专用
> - **ReadonlyAgent**：只读操作
> - **LocAgent**：代码库定位
> - **DummyAgent**：测试用
>
> CodeActAgent 的**核心特点**：
> 1. **统一动作空间**：所有工具调用统一到代码执行
> 2. **多工具支持**：bash、Python、文件编辑、浏览器
> 3. **可配置工具集**：根据配置启用/禁用不同工具
> 4. **内置 Skills**：通过 Jupyter 环境提供 Python 函数库

---

### Q2: 什么是 CodeAct？它相比传统工具调用有什么优势？

**参考答案**：
> CodeAct 源自论文 [CodeAct: Towards Unified Code Actions](https://arxiv.org/abs/2402.01030)。
>
> **核心思想**：将所有 LLM 智能体的动作统一到一个"代码执行空间"，智能体生成 Python 代码来描述操作，而不是调用独立的工具函数。
>
> **传统方式的问题**：
> - 工具数量多（几十上百个），LLM 难以选择
> - 工具参数复杂，学习成本高
> - 难以组合复杂操作
>
> **CodeAct 的优势**：
> - **动作空间小**：只需一个代码执行工具
> - **自然组合**：代码中可以组合多种操作
> - **更贴近人类**：开发者也是通过写代码来完成任务
> - **灵活性高**：代码逻辑可以很复杂

---

### Q3: OpenHands 的工具系统是如何设计的？

**参考答案**：
> **工具定义**：
> - 工具定义为 `ChatCompletionToolParam` 对象（LiteLLM 格式）
> - 包含 name、description、parameters schema
>
> **工具到动作的转换**：
> - `Agent._get_tools()` 根据配置生成工具列表
> - `response_to_actions()` 将 LLM 的 tool_call 解析为具体的 Action 类
>
> **动作执行**：
> - Action 发布到 EventStream
> - Runtime 订阅并执行 Action
> - 返回 Observation
>
> **示例**：
> ```python
> # 工具定义
> bash_tool = ChatCompletionToolParam(
>     type='function',
>     function={
>         'name': 'bash',
>         'description': 'Execute bash commands',
>         'parameters': {...}
>     }
> )
>
> # LLM 响应解析
> if tool_call.function.name == 'bash':
>     action = CmdRunAction(command=arguments['command'])
> ```

---

### Q4: OpenHands 如何实现多智能体协作？

**参考答案**：
> 通过 **AgentDelegateAction** 实现多智能体协作。
>
> **委托流程**：
> 1. Parent Agent 发送 `AgentDelegateAction`，指定目标 Agent 类型和输入
> 2. AgentController 创建 Delegate AgentController
> 3. Parent 和 Delegate **共享同一个 EventStream**
> 4. Delegate 处理子任务，结果通过 `AgentDelegateObservation` 返回
>
> **共享机制**：
> - EventStream：消息总线共享
> - Metrics：成本/迭代指标共享
> - State：delegate_level 递增，隔离状态
>
> **使用场景**：
> - 任务分解：复杂任务拆分为子任务
> - 专业分工：不同 Agent 处理不同类型任务
> - 层级嵌套：多层委托形成树形结构

---

### Q5: Skills 和 Tools 有什么区别？

**参考答案**：
>
> | 维度 | Tool | Skill |
> |------|------|-------|
> | **定义位置** | Agent._get_tools() | runtime/plugins/agent_skills/ |
> | **暴露给 LLM** | 是，作为 function calling | 否，通过 IPython 执行 |
> | **调用方式** | LLM 生成 tool_call | Agent 执行 Python 代码 |
> | **灵活性** | 固定参数 schema | 任意 Python 代码 |
> | **用途** | LLM 决策使用 | 辅助函数功能 |
>
> **配合使用**：
> - Tools 是 LLM 可以"看到"并调用的动作
> - Skills 是 LLM 在代码中可以直接使用的函数库
> - 例如：LLM 用 `bash` tool 执行命令，在命令中调用 `open_file()` skill 函数

---

### Q6: MCP 是什么？OpenHands 如何集成 MCP？

**参考答案**：
> **MCP (Model Context Protocol)** 是一种扩展智能体工具能力的协议。
>
> **OpenHands MCP 集成**：
> 1. **MCPClient**：管理 MCP 连接和工具
> 2. **MCPAction**：封装 MCP 工具调用
> 3. **传输类型**：支持 Stdio、SSE、HTTP 等
>
> **添加 MCP 工具到 Agent**：
> ```python
> agent.set_mcp_tools(mcp_tools)  # 从配置加载
> ```
>
> **MCP 工具执行流程**：
> 1. LLM 选择 MCP 工具
> 2. 生成 MCPAction
> 3. Runtime 通过 MCP client 执行
> 4. 返回 MCPObservation

---

### Q7: Microagent 是什么？有什么用途？

**参考答案**：
> **Microagent** 是轻量级的专门知识智能体，可在对话期间被触发。
>
> **三种类型**：
> 1. **REPO_KNOWLEDGE**：始终激活，提供代码库特定指南
> 2. **KNOWLEDGE**：通过关键词触发，如"Python"、"React"
> 3. **TASK**：通过斜杠命令触发，如`/review`、`/test`
>
> **使用场景**：
> - 代码规范注入（REPO_KNOWLEDGE）
> - 框架最佳实践（KNOWLEDGE）
> - 特定任务执行（TASK）
>
> **触发示例**：
> ```python
> # KNOWLEDGE 类型
> triggers = ['python', 'django']
> if 'python' in user_message:
>     # 注入 Python 最佳实践
>
> # TASK 类型
> /deploy --env=production  # 触发部署任务
> ```

---

### Q8: 如何自定义开发一个新的 Agent？

**参考答案**：
> **步骤**：
> 1. **继承 Agent 基类**
> ```python
> from openhands.controller.agent import Agent
> class MyAgent(Agent):
>     def step(self, state: State) -> Action:
>         # 实现决策逻辑
>         pass
> ```
>
> 2. **注册 Agent**
> ```python
> Agent.register('my_agent', MyAgent)
> ```
>
> 3. **配置使用**
> ```python
> # config.toml
> [agent]
> name = "my_agent"
> ```
>
> **可选：自定义工具**
> ```python
> def _get_tools(self):
>     tools = super()._get_tools()
>     tools.append(my_custom_tool)
>     return tools
> ```

---

### Q9: Agent 的类注册模式有什么好处？

**参考答案**：
> **类注册模式**允许动态发现和加载 Agent。
>
> **好处**：
> 1. **松耦合**：Agent 使用方不需要硬编码导入
> 2. **插件化**：新增 Agent 无需修改框架代码
> 3. **运行时选择**：可以动态选择使用哪个 Agent
> 4. **配置驱动**：通过配置指定 Agent 类型
>
> **实现**：
> ```python
> # 框架代码
> agent_cls = Agent.get_cls(config.name)
> agent = agent_cls(config, llm_registry)
>
> # 插件代码
> @register('my_agent')
> class MyAgent(Agent):
>     pass
> ```

---

### Q10: OpenHands 的工具有哪些安全考虑？

**参考答案**：
> **工具层面**：
> - 高风险操作（如删除文件）有安全风险标记
> - `confirmation_mode` 可以要求用户确认
> - `security_analyzer` 分析动作风险
>
> **执行层面**：
> - Runtime 在隔离环境（Docker/K8s）中执行
> - LocalRuntime 有明确警告，无沙箱隔离
>
> **EventStream 层面**：
> - `hidden=True` 的敏感操作不记录到历史
> - 敏感信息在存储前被替换
>
> **配置示例**：
> ```python
> # 高风险动作需要确认
> CmdRunAction(
>     command='rm -rf /',
>     confirmation_state=ActionConfirmationStatus.REQUIRE_CONFIRMATION
> )
> ```

