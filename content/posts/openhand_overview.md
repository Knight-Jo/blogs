+++
date = '2026-04-10T09:52:11+08:00'
title = 'Openhand_overview'
draft = false
+++

## 文档索引

| 文档 | 内容 |
|------|------|
| **tutorial.md** | 项目概述、架构、面试指南 |
| **[tutorial_agent.md](./tutorial_agent.md)** | Agent 详解：CodeAct、工具、多智能体、MCP、Skills |
| **[tutorial_controller.md](./tutorial_controller.md)** | Controller 详解：主循环、状态管理、委托、循环检测 |
| **[tutorial_event_stream.md](./tutorial_event_stream.md)** | EventStream 详解：事件流、发布-订阅、持久化 |
| **[tutorial_runtime.md](./tutorial_runtime.md)** | Runtime 详解：沙箱、Docker、执行机制、预热池 |

---

## 项目概述

**OpenHands** 是一个 AI 原生的软件工程智能体平台，可以让 AI 像人类开发者一样完成各种软件工程任务，如编写代码、运行命令、浏览网页、操作 Git 仓库等。

### OpenHands 能做什么？

- **自动化代码开发**：编写、修改、重构代码
- **Bug 修复**：分析和修复代码问题
- **代码审查**：审查 Pull Request
- **自动化测试**：运行和编写测试
- **技术调研**：搜索文档、阅读代码库
- **发布准备**：提交代码、创建 PR

### 应用场景

| 场景 | 说明 |
|------|------|
| **本地开发** | 通过 CLI 或 GUI 与 OpenHands 交互 |
| **云端服务** | 托管在云端的服务，支持 Slack、Jira 集成 |
| **企业部署** | 支持 Kubernetes 私有化部署 |
| **SDK 集成** | 通过 Python SDK 构建自己的 AI 应用 |

---

## 系统架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户界面层                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │  CLI     │  │  Web GUI  │  │  Slack   │  │   SDK    │        │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘        │
└───────┼─────────────┼─────────────┼─────────────┼───────────────┘
        │             │             │             │
        v             v             v             v
┌─────────────────────────────────────────────────────────────────┐
│                        服务层 (App Server)                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Conversation Manager (会话管理)              │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  Sandbox    │  │   Event     │  │   User      │              │
│  │  Service    │  │   Service   │  │   Service   │              │
│  └──────┬──────┘  └──────┬──────┘  └─────────────┘              │
└─────────┼────────────────┼──────────────────────────────────────┘
          │                │
          v                v
┌─────────────────────────────────────────────────────────────────┐
│                      智能体层 (Agent Core)                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  AgentController                          │   │
│  │                  (智能体控制器)                            │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   Agent     │  │    LLM      │  │   Event     │              │
│  │  (智能体)    │  │  (大模型)    │  │  Stream     │              │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
└─────────┼────────────────┼────────────────┼─────────────────────┘
          │                │                │
          v                v                v
┌─────────────────────────────────────────────────────────────────┐
│                      运行时层 (Runtime)                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Action Execution Server                     │   │
│  │              (动作执行服务器)                              │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   Docker    │  │  IPython    │  │  Browser    │              │
│  │  (容器)      │  │  (代码执行)  │  │  (浏览)      │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└─────────────────────────────────────────────────────────────────┘
```

### 核心组件

#### 1. 智能体 (Agent)

智能体是 OpenHands 的"大脑"，负责：
- 理解用户任务
- 生成执行计划
- 调用工具完成操作

OpenHands 提供多种内置智能体：

| 智能体 | 用途 |
|--------|------|
| **CodeActAgent** | 主要使用的智能体，采用 CodeAct 方法论 |
| **BrowsingAgent** | 网页浏览任务 |
| **ReadonlyAgent** | 只读操作 |
| **DummyAgent** | 测试用 |

**CodeAct 是什么？**
CodeAct 将所有工具调用统一到一个"代码执行空间"，智能体可以：
- 执行 Bash 命令
- 运行 Python 代码
- 浏览网页
- 编辑文件

#### 2. 控制器 (AgentController)

AgentController 是智能体的"引擎"，负责：
- 管理智能体的主循环
- 维护任务状态
- 协调各组件通信

```
用户任务 → AgentController → LLM → Action → Runtime → Observation → 状态更新
```

#### 3. 事件系统 (Event Stream)

OpenHands 采用**事件驱动架构**：

```
┌──────────┐    Action     ┌───────────┐    Action     ┌──────────┐
│   Agent  │ ────────────► │ EventStream│ ────────────► │ Runtime  │
│          │               │  (事件流)   │               │ (运行时)  │
│          │ ◄──────────── │            │ ◄──────────── │          │
└──────────┘   Observation └───────────┘   Observation └──────────┘
```

**事件类型**：
- **Action（动作）**：智能体发出的操作指令，如 `CmdRunAction`（运行命令）、`FileReadAction`（读文件）
- **Observation（观察）**：执行结果，如 `CmdOutputObservation`（命令输出）

#### 4. 运行时 (Runtime)

Runtime 是智能体的"手脚"，负责在隔离环境中执行实际操作：

| 运行时 | 说明 |
|--------|------|
| **Docker** | 在 Docker 容器中执行（推荐，默认） |
| **Local** | 直接在本地机器执行 |
| **Remote** | 连接远程执行服务器 |
| **Kubernetes** | K8s 集群中执行 |

#### 5. LLM 集成

OpenHands 通过 **LiteLLM** 支持几乎所有主流大模型：

```
OpenHands → LiteLLM → Anthropic / OpenAI / Azure / 本地模型
```

支持的模型示例：
- Claude 3.5 Sonnet
- GPT-4o
- Azure OpenAI
- 任何 OpenAI 兼容 API

---

## 核心工作流程

```
1. 用户提交任务
   └─► "帮我修复这个 Bug"

2. AgentController 接收任务
   └─► 创建 State（状态）

3. 智能体循环 (Agent Loop)
   ┌─────────────────────────────┐
   │  a) 生成 prompt             │
   │  b) 调用 LLM                │
   │  c) 解析响应为 Action        │
   │  d) Runtime 执行 Action     │
   │  e) 获取 Observation         │
   │  f) 更新 State               │
   │  g) 检查是否完成             │
   └─────────────────────────────┘

4. 任务完成
   └─► 返回结果给用户
```

---

## 目录结构

```
OpenHands/
├── openhands/                 # 核心 Python 代码
│   ├── agenthub/             # 智能体实现
│   │   ├── codeact_agent/    # CodeAct 智能体
│   │   └── ...
│   ├── controller/          # AgentController
│   ├── events/               # 事件系统
│   │   ├── action/           # 动作类型
│   │   └── observation/      # 观察类型
│   ├── runtime/              # 运行时实现
│   │   └── impl/             # Docker/Local/Remote
│   ├── llm/                  # LLM 集成
│   ├── app_server/           # API 服务器
│   └── ...
├── frontend/                  # React 前端
├── enterprise/                # 企业功能
└── containers/                # Docker 配置
```

---

## 面试指南

### 如何介绍 OpenHands

**公式**：项目概述 + 核心价值 + 技术亮点

> "OpenHands 是一个开源的 AI 软件工程智能体平台，可以让 AI 像人类开发者一样完成代码开发、Bug 修复、代码审查等工作。
>
> 它的核心设计是**事件驱动的智能体系统**：智能体（Agent）通过大模型理解任务，生成动作（Action），由运行时（Runtime）在隔离环境中执行，返回观察结果（Observation），形成主循环直到任务完成。
>
> 技术亮点包括：1）采用 CodeAct 方法论统一动作空间；2）基于事件流的可扩展架构；3）支持多种运行时（Docker/K8s）；4）通过 LiteLLM 兼容所有主流大模型。"

### 核心概念速记

| 概念 | 一句话解释 |
|------|-----------|
| **Agent** | 智能体，大脑，决定做什么 |
| **Controller** | 控制器，引擎，驱动主循环 |
| **Event Stream** | 事件流，消息总线，连接各组件 |
| **Runtime** | 运行时，手脚，实际执行操作 |
| **Action** | 动作，智能体的指令 |
| **Observation** | 观察，执行的结果 |
| **CodeAct** | 一种方法论，统一动作空间 |

---

## 常见面试问题

### Q1: OpenHands 的整体架构是怎样的？

**参考答案**：
> OpenHands 采用分层架构：
> - **用户界面层**：CLI、Web GUI、Slack 集成、SDK
> - **服务层**：App Server 处理 REST API、会话管理、沙箱服务
> - **智能体层**：AgentController 驱动主循环，Agent 负责决策，EventStream 作为消息总线
> - **运行时层**：Action Execution Server 在 Docker/K8s 等隔离环境中执行实际操作
>
> 核心是**事件驱动**：Agent 发 Action → EventStream 转发 → Runtime 执行 → 返回 Observation → 更新 State。

---

### Q2: 智能体（Agent）是如何工作的？

**参考答案**：
> 智能体基于大模型实现，主要流程：
> 1. 接收当前状态（State）和历史事件
> 2. 生成 prompt，包含任务描述和可用工具
> 3. 调用 LLM 获取响应
> 4. 解析响应为具体的 Action
> 5. Action 由 Runtime 执行，返回 Observation
> 6. 循环直到任务完成

> OpenHands 主要使用 **CodeActAgent**，它将所有工具统一为代码执行形式（bash、python、文件操作、网页浏览）。

---

### Q3: 什么是 CodeAct？它有什么优势？

**参考答案**：
> CodeAct（Code Action）是一种智能体动作空间设计方法。
>
> **传统方式**：为每种操作定义独立工具（如 `search_web`、`read_file`、`run_command`），工具数量多，调用复杂。
>
> **CodeAct**：只用一个"代码执行"工具，智能体生成 Python 代码来描述操作，代码中可以调用各种功能函数。
>
> **优势**：
> - 动作空间小且统一，LLM 更容易学习
> - 可以组合复杂操作
> - 更自然，贴近人类开发者行为

---

### Q4: Event Stream 在系统中的作用是什么？

**参考答案**：
> EventStream 是 OpenHands 的**中央消息总线**，采用发布-订阅模式。
>
> **作用**：
> 1. **解耦**：Agent、Runtime、Server 等组件不直接通信，都通过 EventStream
> 2. **可观察**：所有事件都被记录，便于调试和回放
> 3. **可扩展**：新组件只需订阅感兴趣的事件类型
>
> **事件类型**：
> - `Action`：Agent 发出（CmdRunAction、FileEditAction 等）
> - `Observation`：Runtime 返回（CmdOutputObservation、FileContentObservation 等）
> - `Message`：用户消息、系统消息

---

### Q5: Runtime 是如何实现的？为什么要用 Docker？

**参考答案**：
> Runtime 是执行动作的"手脚"，定义了抽象接口 `BaseRuntime`，具体实现有：
> - `DockerRuntime`：在 Docker 容器中执行（默认）
> - `LocalRuntime`：直接在宿主机执行
> - `RemoteRuntime`：连接远程服务器
> - `KubernetesRuntime`：K8s 集群中执行
>
> **使用 Docker 的原因**：
> 1. **隔离性**：避免恶意代码影响宿主机器
> 2. **一致性**：每次执行环境相同
> 3. **资源控制**：可以限制 CPU、内存
> 4. **可清理**：执行完毕即可销毁容器

---

### Q6: OpenHands 支持哪些大模型？如何扩展？

**参考答案**：
> 通过 **LiteLLM** 库，OpenHands 支持几乎所有主流 LLM：
> - Anthropic：Claude 3.5、Claude 4
> - OpenAI：GPT-4o、GPT-4 Turbo
> - Azure OpenAI
> - 任何 OpenAI 兼容 API
>
> **扩展新模型**：
> 1. 在配置中指定模型名称和 API 密钥
> 2. LiteLLM 会自动处理协议转换
> 3. 自定义模型只需实现 OpenAI 兼容接口

---

### Q7: 如何保证系统的可扩展性？

**参考答案**：
> OpenHands 通过几个设计保证可扩展性：
>
> 1. **事件驱动架构**：新组件只需订阅事件，无需修改现有代码
> 2. **插件化 Agent**：在 `agenthub/` 目录下添加新类即可
> 3. **插件化 Runtime**：继承 `BaseRuntime` 实现新运行时
> 4. **配置驱动**：行为通过配置文件和环境变量控制
> 5. **工具可扩展**：Agent 的工具集可以在配置中添加/移除

---

### Q8: 系统的状态管理是如何工作的？

**参考答案**：
> State（状态）贯穿整个任务执行周期：
>
> ```
> State {
>   agent_id: str
>   task: str                    # 当前任务
>   history: List[Event]         # 历史事件
>   memory: ShortTermMemory      # 短期记忆
>   ...
> }
> ```
>
> **状态流转**：
> 1. 任务开始，创建初始 State
> 2. 每次 Agent Loop 执行后，State 更新
> 3. EventStore 持久化事件序列
> 4. Memory Condenser 在上下文过长时压缩历史
>
> 这样设计保证了**可回溯性**：任何时候都可以重放历史事件。

---

### Q9: 在实际使用中遇到过什么问题？如何解决？

**参考思路**：
> 可以结合个人经验回答，以下是常见问题：
>
> - **LLM 调用超时**：配置重试机制和超时时间
> - **上下文过长**：使用 Memory Condenser 压缩历史
> - **沙箱启动慢**：使用沙箱预热池
> - **文件编码问题**：统一使用 UTF-8
> - **长任务中断**：实现检查点保存机制

---

### Q10: OpenHands 与其他 AI 编程助手（如 GitHub Copilot）的区别？

**参考答案**：
>
> | 维度 | OpenHands | Copilot |
> |------|------------|----------|
> | **交互方式** | 对话式、自动化执行 | 补全式、IDE 插件 |
> | **执行能力** | 可以执行代码、命令 | 主要生成代码片段 |
> | **自主性** | 可以自主完成任务 | 需要人类引导 |
> | **部署方式** | 可私有化部署 | SaaS |
> | **定制性** | 开源，可完全定制 | 封闭 |
> | **场景** | 端到端任务自动化 | 代码补全 |
>
> 简单说：**Copilot 是副驾驶**，帮人写代码；**OpenHands 是自动驾驶**，帮人完成任务。

---

## 快速开始

### 安装

```bash
pip install openhands
```

### 配置

```bash
# 设置 API 密钥
export LLM_API_KEY="your-api-key"
export LLM_MODEL="anthropic/claude-3-5-sonnet-20241022"
```

### 使用 CLI

```bash
# 启动交互式会话
openhands

# 执行指定任务
openhands --task "帮我修复 src/bug.py 中的空指针异常"
```

### 使用 Python SDK

```python
from openhands.agent import CodeActAgent
from openhands.llm import LLM
from openhands.controller import AgentController

# 初始化
llm = LLM(model='anthropic/claude-3-5-sonnet-20241022')
agent = CodeActAgent(llm=llm)
controller = AgentController(agent=agent)

# 执行任务
controller.run("帮我修复 src/bug.py 中的 Bug")
```

---

## 推荐阅读

### 核心组件详解

- [Agent 详解](./tutorial_agent.md) - 智能体设计、CodeAct、工具系统、多智能体、MCP、Skills
- [Controller 详解](./tutorial_controller.md) - 主循环、状态管理、委托、循环检测
- [EventStream 详解](./tutorial_event_stream.md) - 事件流、发布-订阅、持久化、设计模式
- [Runtime 详解](./tutorial_runtime.md) - 沙箱、Docker、执行机制、预热池

### 其他资源

- [官方 README](../README.md)
- [架构文档](./openhands/architecture/README.md)
- [Agent 执行流程](./openhands/architecture/agent-execution.md)
- [事件系统](./openhands/events/)

---

*本文档面向技术面试准备，建议结合实际使用体验深入理解核心概念。*
