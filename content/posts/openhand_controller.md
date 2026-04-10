+++
date = '2026-04-10T09:51:58+08:00'
title = 'Openhand_controller'
draft = false
+++

## 目录
1. [概述](#1-概述)
2. [AgentController 核心结构](#2-agentcontroller-核心结构)
3. [状态管理](#3-状态管理)
4. [主循环实现](#4-主循环实现)
5. [事件流集成](#5-事件流集成)
6. [委托/多智能体系统](#6-委托多智能体系统)
7. [循环检测与错误处理](#7-循环检测与错误处理)
8. [配置系统](#8-配置系统)
9. [架构流程图](#9-架构流程图)
10. [面试问题](#10-面试问题)

---

## 1. 概述

### 1.1 什么是 AgentController

**AgentController** 是 OpenHands 的"引擎"，负责：
- 驱动智能体的主循环
- 管理任务状态
- 协调各组件通信
- 处理异常和循环检测

**文件**: `/home/gdw/tutorial/OpenHands/openhands/controller/agent_controller.py`

### 1.2 核心职责

```
AgentController
    │
    ├── 主循环驱动 (_step)
    ├── 状态管理 (StateTracker)
    ├── 事件路由 (EventStream)
    ├── 委托管理 (Delegate)
    ├── 循环检测 (StuckDetector)
    ├── 安全分析 (SecurityAnalyzer)
    └── 配置控制 (ControlFlags)
```

---

## 2. AgentController 核心结构

### 2.1 类定义

```python
class AgentController:
    id: str
    agent: Agent
    max_iterations: int
    event_stream: EventStream
    state: State
    confirmation_mode: bool
    agent_to_llm_config: dict[str, LLMConfig]
    agent_configs: dict[str, AgentConfig]
    parent: 'AgentController | None' = None
    delegate: 'AgentController | None' = None
    _pending_action_info: tuple[Action, float] | None = None
    _closed: bool = False
```

### 2.2 初始化流程

```python
def __init__(
    self,
    agent: Agent,
    event_stream: EventStream,
    conversation_stats: ConversationStats,
    iteration_delta: int,  # 最大迭代次数
    budget_per_task_delta: float | None = None,  # 最大预算
    agent_to_llm_config: dict[str, LLMConfig] | None = None,
    agent_configs: dict[str, AgentConfig] | None = None,
    sid: str | None = None,
    file_store: FileStore | None = None,
    user_id: str | None = None,
    confirmation_mode: bool = False,
    initial_state: State | None = None,
    is_delegate: bool = False,
    headless_mode: bool = True,
    status_callback: Callable | None = None,
    replay_events: list[Event] | None = None,
    security_analyzer: 'SecurityAnalyzer | None' = None,
):
```

**初始化步骤**：
1. 设置 EventStream 并订阅
2. 创建 StateTracker
3. 初始化或恢复状态
4. 初始化 StuckDetector
5. 设置 ReplayManager
6. 添加系统消息

---

## 3. 状态管理

### 3.1 State 类

**文件**: `/home/gdw/tutorial/OpenHands/openhands/controller/state/state.py`

```python
@dataclass
class State:
    session_id: str = ''
    user_id: str | None = None

    # 迭代控制
    iteration_flag: IterationControlFlag = field(default_factory=lambda: IterationControlFlag(
        limit_increase_amount=100, current_value=0, max_value=100
    ))

    # 预算控制
    budget_flag: BudgetControlFlag | None = None

    # 确认模式
    confirmation_mode: bool = False

    # 历史记录
    history: list[Event] = field(default_factory=list)

    # 输入输出
    inputs: dict = field(default_factory=dict)
    outputs: dict = field(default_factory=dict)

    # 智能体状态
    agent_state: AgentState = AgentState.LOADING

    # 委托层级
    delegate_level: int = 0

    # 指标
    metrics: Metrics = field(default_factory=Metrics)

    # 额外数据
    extra_data: dict[str, Any] = field(default_factory=dict)
    last_error: str = ''
```

### 3.2 StateTracker

**文件**: `/home/gdw/tutorial/OpenHands/openhands/controller/state/state_tracker.py`

```python
class StateTracker:
    def __init__(self, sid: str | None, file_store: FileStore | None, user_id: str | None):
        self.state: State
        self.agent_history_filter = EventFilter(
            exclude_types=(NullAction, NullObservation, ChangeAgentStateAction, AgentStateChangedObservation),
            exclude_hidden=True,
        )
```

**关键方法**：
- `set_initial_state()` - 初始化或恢复状态
- `add_history()` - 添加事件到历史
- `run_control_flags()` - 执行控制标志检查
- `save_state()` - 持久化到文件存储

### 3.3 ControlFlags

**文件**: `/home/gdw/tutorial/OpenHands/openhands/controller/state/control_flags.py`

```python
@dataclass
class IterationControlFlag(ControlFlag[int]):
    limit_increase_amount: int = 100
    current_value: int = 0
    max_value: int = 100

    def reached_limit(self) -> bool:
        self._hit_limit = self.current_value >= self.max_value
        return self._hit_limit

    def step(self):
        if self.reached_limit():
            raise RuntimeError(f'Agent reached maximum iteration...')
        self.current_value += 1

    def increase_limit(self, headless_mode: bool) -> None:
        if not headless_mode and self._hit_limit:
            self.max_value += self.limit_increase_amount
            self._hit_limit = False
```

---

## 4. 主循环实现

### 4.1 Step 执行流程

**文件**: `/home/gdw/tutorial/OpenHands/openhands/controller/agent_controller.py` (line 863)

```python
async def _step(self) -> None:
    """执行父智能体或委托智能体的一步"""

    # 1. 检查智能体状态 - 只在 RUNNING 时执行
    if self.get_agent_state() != AgentState.RUNNING:
        return

    # 2. 检查待处理动作
    if self._pending_action:
        return

    # 3. 记录当前步骤
    self.log('debug', f'LEVEL {self.state.delegate_level} LOCAL STEP...')

    # 4. 同步预算标志与 LLM 成本
    self.state_tracker.sync_budget_flag_with_metrics()

    # 5. 检查是否卡住
    if self.agent.config.enable_stuck_detection and self._is_stuck():
        await self._react_to_exception(AgentStuckInLoopError('Agent got stuck in a loop'))
        return

    # 6. 执行控制标志检查
    try:
        self.state_tracker.run_control_flags()
    except Exception as e:
        await self._react_to_exception(e)
        return

    # 7. 从智能体获取动作或重放
    action: Action = NullAction()
    if self._replay_manager.should_replay():
        action = self._replay_manager.step()  # 从轨迹重放
    else:
        try:
            action = self.agent.step(self.state)  # 调用 LLM
        except ContextWindowExceededError:
            if self.agent.config.enable_history_truncation:
                self.event_stream.add_event(CondensationRequestAction(), EventSource.AGENT)
                return
            raise LLMContextWindowExceedError()

    # 8. 处理确认模式
    if action.runnable and self.state.confirmation_mode:
        await self._handle_security_analyzer(action)
        # 根据安全风险设置确认状态...

    # 9. 添加动作到事件流
    if not isinstance(action, NullAction):
        self.event_stream.add_event(action, action._source)
```

### 4.2 事件触发循环

```python
def on_event(self, event: Event) -> None:
    """事件回调 - EventStream 通知"""
    # 如果有活跃委托，转发事件
    if self.delegate is not None:
        if delegate_state not in (AgentState.FINISHED, AgentState.ERROR, AgentState.REJECTED):
            asyncio.get_event_loop().run_until_complete(self.delegate._on_event(event))
            return
        else:
            self.end_delegate()
            return

    asyncio.get_event_loop().run_until_complete(self._on_event(event))

async def _on_event(self, event: Event) -> None:
    if hasattr(event, 'hidden') and event.hidden:
        return

    self.state_tracker.add_history(event)

    if isinstance(event, Action):
        await self._handle_action(event)
    elif isinstance(event, Observation):
        await self._handle_observation(event)

    should_step = self.should_step(event)
    if should_step:
        await self._step_with_exception_handling()
```

### 4.3 AgentState 状态机

```
LOADING
   │
   v
AWAITING_USER_INPUT <───────────────────> RUNNING
   │                                          │
   │                                          v
   │                              AWAITING_USER_CONFIRMATION
   │                                          │
   │                              USER_CONFIRMED/USER_REJECTED
   │                                          │
   +---------> PAUSED <───────────────────────┤
   │              │
   │              v
   +------> STOPPED/ERROR/FINISHED/REJECTED
```

---

## 5. 事件流集成

### 5.1 订阅 EventStream

```python
# __init__ 中
if not self.is_delegate:
    self.event_stream.subscribe(
        EventStreamSubscriber.AGENT_CONTROLLER, self.on_event, self.id
    )
```

### 5.2 EventStream 事件路由

```python
def should_step(self, event: Event) -> bool:
    """判断事件是否应触发智能体步骤"""

    # 有委托时不触发
    if self.delegate is not None:
        return False

    if isinstance(event, Action):
        if isinstance(event, MessageAction) and event.source == EventSource.USER:
            return True
        if isinstance(event, AgentDelegateAction):
            return True
        if isinstance(event, CondensationAction):
            return True
        return False

    if isinstance(event, Observation):
        if isinstance(event, NullObservation) and event.cause > 0:
            return True
        if isinstance(event, AgentStateChangedObservation):
            return False
        return True

    return False
```

---

## 6. 委托/多智能体系统

### 6.1 启动委托

```python
async def start_delegate(self, action: AgentDelegateAction) -> None:
    """启动委托智能体"""
    # 1. 创建被委托的智能体
    agent_cls: type[Agent] = Agent.get_cls(action.agent)
    delegate_agent = agent_cls(config=agent_config, llm_registry=self.agent.llm_registry)

    # 2. 创建共享状态的委托状态
    state = State(
        session_id=self.id.removesuffix('-delegate'),
        iteration_flag=self.state.iteration_flag,      # 共享！
        budget_flag=self.state.budget_flag,            # 共享！
        delegate_level=self.state.delegate_level + 1,
        metrics=self.state.metrics,                    # 共享！
        start_id=self.event_stream.get_latest_event_id() + 1,
        parent_metrics_snapshot=self.state_tracker.get_metrics_snapshot(),
        parent_iteration=self.state.iteration_flag.current_value,
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

### 6.2 委托状态流转

```
Parent AgentController
    │
    ├── 发送 AgentDelegateAction
    │
    ▼
Delegate AgentController (is_delegate=True)
    │
    ├── 处理子任务
    ├── 共享 EventStream
    └── 独立的状态机
    │
    ▼
完成 → AgentDelegateObservation → Parent 继续
```

### 6.3 委托结束

```python
def end_delegate(self) -> None:
    # 更新父级的迭代计数
    self.state.iteration_flag.current_value = (
        self.delegate.state.iteration_flag.current_value
    )

    # 发送委托结果观察
    obs = AgentDelegateObservation(outputs=delegate_outputs, content=content)
    self.event_stream.add_event(obs, EventSource.AGENT)

    self.delegate = None
```

---

## 7. 循环检测与错误处理

### 7.1 StuckDetector

**文件**: `/home/gdw/tutorial/OpenHands/openhands/controller/stuck.py`

```python
class StuckDetector:
    def is_stuck(self, headless_mode: bool = True) -> bool:
        # 过滤用户消息和空事件
        # 检查最近 4 个动作/观察的模式
```

**检测场景**：

| 场景 | 条件 | 处理 |
|------|------|------|
| 重复动作-观察 | 相同动作 4 次相同观察 | 恢复 |
| 重复动作-错误 | 相同动作 3 次错误观察 | 恢复 |
| 独白 | 相同 MessageAction 3 次无观察 | 恢复 |
| 动作-观察模式 | 交替模式重复 | 恢复 |
| 上下文窗口错误循环 | 10+ 重复压缩事件 | 报错 |

### 7.2 错误处理

```python
async def _react_to_exception(self, e: Exception) -> None:
    self.state.last_error = f'{type(e).__name__}: {str(e)}'

    # 分类错误
    if isinstance(e, AuthError):
        status = RuntimeStatus.AUTH_ERROR
    elif isinstance(e, RateLimitError):
        status = RuntimeStatus.RATE_LIMIT_ERROR
    elif isinstance(e, ContentPolicyViolationError):
        status = RuntimeStatus.CONTENT_POLICY_ERROR
    else:
        status = RuntimeStatus.ERROR

    await self.set_agent_state_to(AgentState.ERROR)
```

### 7.3 异常恢复流程

```
异常发生
    │
    ▼
_react_to_exception()
    │
    ├── 记录错误
    ├── 分类异常类型
    ├── 设置 AgentState = ERROR
    │
    ▼
如果可恢复 → LoopRecoveryAction
    │
    ▼
继续主循环
```

---

## 8. 配置系统

### 8.1 AgentConfig

**文件**: `/home/gdw/tutorial/OpenHands/openhands/core/config/agent_config.py`

```python
class AgentConfig(BaseModel):
    cli_mode: bool = Field(default=False)
    enable_browsing: bool = Field(default=True)
    enable_editor: bool = Field(default=True)
    enable_jupyter: bool = Field(default=True)
    enable_cmd: bool = Field(default=True)
    enable_think: bool = Field(default=True)
    enable_finish: bool = Field(default=True)
    enable_history_truncation: bool = Field(default=True)
    enable_stuck_detection: bool = Field(default=True)
    enable_plan_mode: bool = Field(default=True)
    condenser: CondenserConfig = Field(default_factory=ConversationWindowCondenserConfig)
```

### 8.2 OpenHandsConfig

**文件**: `/home/gdw/tutorial/OpenHands/openhands/core/config/openhands_config.py`

```python
class OpenHandsConfig(BaseModel):
    max_iterations: int = Field(default=OH_MAX_ITERATIONS)  # 默认 100
    max_budget_per_task: float | None = Field(default=None)
```

### 8.3 Controller 初始化配置

```python
controller = AgentController(
    agent=agent,
    conversation_stats=conversation_stats,
    iteration_delta=config.max_iterations,        # 最大迭代
    budget_per_task_delta=config.max_budget_per_task,  # 最大预算
    agent_to_llm_config=config.get_agent_to_llm_config_map(),
    event_stream=event_stream,
    initial_state=initial_state,
    headless_mode=headless_mode,
    confirmation_mode=config.security.confirmation_mode,
    replay_events=replay_events,
    security_analyzer=runtime.security_analyzer,
)
```

---

## 9. 架构流程图

### 9.1 完整执行流程

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户提交任务                            │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    EventStream.add_event()                      │
│  - 发布用户消息为 MessageAction                                 │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AgentController.on_event()                    │
│  - 接收事件                                                     │
│  - 状态跟踪                                                     │
│  - should_step() 判断是否执行步骤                              │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      _step_with_exception_handling()            │
│  - _step()                                                      │
└─────────────────────────────────────────────────────────────────┘
                                │
            ┌───────────────────┴───────────────────┐
            ▼                                       ▼
┌─────────────────────────┐             ┌─────────────────────────┐
│     _step()             │             │     重放模式            │
│                         │             │                         │
│  1. 检查状态            │             │  _replay_manager.step() │
│  2. 同步预算            │             │  从保存的轨迹重放       │
│  3. 循环检测            │             │                         │
│  4. 控制标志检查        │             │                         │
│  5. agent.step()        │             │                         │
│  6. 安全分析            │             │                         │
│  7. 发布动作            │             │                         │
└─────────────────────────┘             └─────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    EventStream.add_event(action)                │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Runtime                                │
│  - 接收 Action                                                  │
│  - 执行动作                                                     │
│  - 返回 Observation                                            │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AgentController._handle_observation()         │
│  - 更新状态                                                     │
│  - 继续主循环                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 状态流转图

```
                    ┌──────────────┐
                    │   LOADING    │
                    └──────┬───────┘
                           │
                           ▼
              ┌────────────────────────┐
              │  AWAITING_USER_INPUT   │◄─────────────┐
              └───────────┬────────────┘              │
                          │                          │
                          │ 用户输入                  │
                          ▼                          │
              ┌────────────────────────┐             │
              │       RUNNING          │────────────►│
              └───────────┬────────────┘             │
                          │                          │
                          │ 高风险动作                │
                          ▼                          │
              ┌────────────────────────┐             │
              │AWAITING_USER_CONFIRMATION│           │
              └───────────┬────────────┘             │
                          │                          │
              ┌───────────┴───────────┐             │
              ▼                       ▼              │
      ┌───────────────┐       ┌───────────────┐     │
      │USER_CONFIRMED │       │ USER_REJECTED  │     │
      └───────┬───────┘       └───────┬───────┘     │
              │                       │              │
              └───────────┬───────────┘              │
                          │                          │
                          ▼                          │
              ┌────────────────────────┐             │
              │        PAUSED         │──────────────┘
              └───────────┬────────────┘
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
      ┌───────────────┐       ┌───────────────┐
      │    ERROR      │       │   FINISHED    │
      └───────────────┘       └───────────────┘
```

---

## 10. 面试问题

### Q1: AgentController 的主要职责是什么？

**参考答案**：
> AgentController 是 OpenHands 的**中央协调器**，主要职责：
>
> 1. **主循环驱动**：通过 `_step()` 驱动智能体循环执行
> 2. **状态管理**：通过 StateTracker 维护状态
> 3. **事件路由**：订阅 EventStream，处理 Action 和 Observation
> 4. **委托管理**：支持多智能体协作
> 5. **循环检测**：检测并恢复卡住的状态
> 6. **安全分析**：分析动作风险，处理确认模式
> 7. **控制标志**：管理迭代次数和预算限制

---

### Q2: AgentController 如何驱动主循环？

**参考答案**：
> **触发机制**：EventStream 回调
> ```python
> def on_event(self, event: Event) -> None:
>     asyncio.get_event_loop().run_until_complete(self._on_event(event))
>
> async def _on_event(self, event: Event) -> None:
>     # 处理事件
>     if should_step(event):
>         await self._step_with_exception_handling()
> ```
>
> **Step 执行流程**：
> 1. 检查 AgentState 是否为 RUNNING
> 2. 同步预算与 LLM 成本
> 3. 循环检测（StuckDetector）
> 4. 运行控制标志（迭代/预算限制）
> 5. 调用 `agent.step(state)` 获取 Action
> 6. 安全分析（如启用确认模式）
> 7. 发布 Action 到 EventStream
>
> **终止条件**：
> - 达到最大迭代次数
> - 达到最大预算
> - Agent 返回 AgentFinishAction
> - 检测到循环卡住

---

### Q3: 状态管理是如何工作的？

**参考答案**：
> **State 结构**：
> - `agent_state`：RUNNING、PAUSED、ERROR 等
> - `history`：事件历史列表
> - `iteration_flag`：迭代控制
> - `budget_flag`：预算控制
> - `delegate_level`：委托嵌套层级
> - `metrics`：LLM 成本和 token 使用
>
> **StateTracker**：
> - `set_initial_state()`：初始化或恢复
> - `add_history()`：添加事件到历史
> - `run_control_flags()`：检查迭代/预算限制
> - `sync_budget_flag_with_metrics()`：同步预算与实际成本
>
> **状态持久化**：
> ```python
> state.save_to_session(sid, file_store, user_id)
> State.restore_from_session(sid, file_store, user_id)
> ```

---

### Q4: 什么是 ControlFlags？如何工作？

**参考答案**：
> **ControlFlags** 用于防止无限循环和超支：
>
> 1. **IterationControlFlag**
> ```python
> class IterationControlFlag:
>     max_value: int = 100      # 最大迭代次数
>     current_value: int = 0    # 当前迭代
>
>     def step(self):
>         if self.reached_limit():
>             raise RuntimeError('Agent reached maximum iteration...')
>         self.current_value += 1
> ```
>
> 2. **BudgetControlFlag**
> ```python
> class BudgetControlFlag:
>     max_value: float          # 最大预算
>     current_value: float      # 已消耗
>
>     def step(self):
>         if self.reached_limit():
>             raise RuntimeError('Agent reached maximum budget...')
> ```
>
> **限制增加**：在非 headless 模式下，限制可动态增加 100

---

### Q5: 如何检测和处理循环卡住？

**参考答案**：
> **StuckDetector** 检测以下模式：
>
> | 场景 | 检测条件 | 处理 |
> |------|----------|------|
> | 重复动作-观察 | 相同动作 4 次相同观察 | LoopRecoveryAction |
> | 重复动作-错误 | 相同动作 3 次错误观察 | LoopRecoveryAction |
> | 独白 | 3 次 MessageAction 无观察 | LoopRecoveryAction |
> | 上下文窗口错误 | 10+ 压缩事件 | 抛出异常 |
>
> **处理流程**：
> ```python
> if self._is_stuck():
>     await self._react_to_exception(
>         AgentStuckInLoopError('Agent got stuck in a loop')
>     )
> ```

---

### Q6: 多智能体委托是如何实现的？

**参考答案**：
> **委托机制**：
> 1. Parent Agent 发送 `AgentDelegateAction`
> 2. Controller 创建 **Delegate AgentController**
> 3. Delegate **共享** Parent 的：
>    - EventStream（同一消息总线）
>    - Metrics（成本/迭代计数）
>    - iteration_flag/budget_flag
> 4. Delegate 有独立的状态机
>
> **关键代码**：
> ```python
> state = State(
>     delegate_level=self.state.delegate_level + 1,
>     metrics=self.state.metrics,  # 共享
>     iteration_flag=self.state.iteration_flag,  # 共享
> )
> ```
>
> **通信**：通过 EventStream，Delegate 的结果以 `AgentDelegateObservation` 返回

---

### Q7: AgentController 如何处理错误？

**参考答案**：
> **错误分类与处理**：
> ```python
> async def _react_to_exception(self, e: Exception) -> None:
>     self.state.last_error = f'{type(e).__name__}: {str(e)}'
>
>     if isinstance(e, AuthError):
>         status = RuntimeStatus.AUTH_ERROR
>     elif isinstance(e, RateLimitError):
>         status = RuntimeStatus.RATE_LIMIT_ERROR
>     elif isinstance(e, ContentPolicyViolationError):
>         status = RuntimeStatus.CONTENT_POLICY_ERROR
>
>     await self.set_agent_state_to(AgentState.ERROR)
> ```
>
> **状态设置**：
> - CONTEXT_WINDOW_EXCEEDED → 历史压缩
> - AUTH_ERROR → 终止
> - RATE_LIMIT_ERROR → 等待重试
> - CONTENT_POLICY → 终止

---

### Q8: confirmation_mode 是如何工作的？

**参考答案**：
> **目的**：在高风险动作执行前要求用户确认
>
> **流程**：
> ```python
> if action.runnable and self.state.confirmation_mode:
>     await self._handle_security_analyzer(action)
>     # action.confirmation_state 设置为:
>     # - CONFIRMED：已确认，可执行
>     # - REJECTED：用户拒绝
>     # - REQUIRE_CONFIRMATION：需要确认
> ```
>
> **Action 的安全风险**：
> ```python
> CmdRunAction(
>     command='rm -rf /',
>     security_risk=ActionSecurityRisk.HIGH
> )
> ```
>
> **处理**：
> - HIGH 风险 → REQUIRE_CONFIRMATION
> - MEDIUM 风险 → 可能需要确认
> - LOW 风险 → CONFIRMED

---

### Q9: 什么是 replay_events？如何实现重放？

**参考答案**：
> **ReplayManager** 用于重放已保存的执行轨迹：
>
> ```python
> class ReplayManager:
>     def should_replay(self) -> bool:
>         return len(self.events) > 0
>
>     def step(self) -> Action:
>         return self.events.pop(0)  # 逐个返回保存的动作
> ```
>
> **用途**：
> - 调试：重放特定场景
> - 测试：验证修复
> - 演示：回放执行过程

---

### Q10: AgentController 与 Agent 的关系是什么？

**参考答案**：
> **职责分工**：
>
> | 组件 | 职责 | 关注点 |
> |------|------|--------|
> | **Agent** | 决策 | 调用 LLM、生成动作、解析响应 |
> | **AgentController** | 协调 | 主循环、状态、事件路由、异常处理 |
>
> **协作模式**：
> ```python
> # Controller 驱动循环
> action = self.agent.step(self.state)  # Controller 调用 Agent
>
> # Agent 只负责决策
> class CodeActAgent(Agent):
>     def step(self, state: State) -> Action:
>         # 构建 prompt
>         # 调用 LLM
>         # 解析为 Action
>         return action
> ```
>
> **解耦设计**：
> - Agent 不知道 Controller 的存在
> - Controller 通过 State 向 Agent 传递信息
> - 通过 EventStream 解耦

---

### Q11: 如何扩展 AgentController 添加新功能？

**参考答案**：
> **扩展点**：
>
> 1. **新的 Action 处理**
> ```python
> async def _handle_action(self, event: Action) -> None:
>     if isinstance(event, MyNewAction):
>         await self._handle_my_new_action(event)
> ```
>
> 2. **新的 Observation 处理**
> ```python
> async def _handle_observation(self, event: Observation) -> None:
>     if isinstance(event, MyNewObservation):
>         await self._handle_my_new_observation(event)
> ```
>
> 3. **新的控制标志**
> ```python
> class MyControlFlag(ControlFlag):
>     def step(self):
>         # 自定义逻辑
> ```
>
> 4. **新的循环检测**
> ```python
> class MyStuckDetector(StuckDetector):
>     def is_stuck(self) -> bool:
>         # 自定义检测
> ```
