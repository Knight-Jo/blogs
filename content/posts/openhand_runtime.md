+++
date = '2026-04-10T09:51:52+08:00'
title = 'Openhand_runtime'
draft = false
+++

## 目录
1. [概述](#1-概述)
2. [Base Runtime 抽象接口](#2-base-runtime-抽象接口)
3. [DockerRuntime 实现](#3-dockerruntime-实现)
4. [LocalRuntime 实现](#4-localruntime-实现)
5. [RemoteRuntime 与 KubernetesRuntime](#5-remoteruntime-与-kubernetesruntime)
6. [Action Execution Server](#6-action-execution-server)
7. [动作执行机制](#7-动作执行机制)
8. [沙箱管理](#8-沙箱管理)
9. [安全机制](#9-安全机制)
10. [架构流程图](#10-架构流程图)
11. [面试问题](#11-面试问题)

---

## 1. 概述

### 1.1 什么是 Runtime

**Runtime** 是 OpenHands 的"手脚"，负责在实际隔离环境中执行 Agent 生成的 Action。

**文件**: `/home/gdw/tutorial/OpenHands/openhands/runtime/`

### 1.2 Runtime 职责

```
Runtime
    │
    ├── 执行 Bash 命令
    ├── 执行 Python 代码 (IPython/Jupyter)
    ├── 文件系统操作 (读/写/编辑)
    ├── 浏览器自动化
    ├── MCP 工具调用
    └── 环境隔离
```

### 1.3 运行时类型

| 运行时 | 说明 |
|--------|------|
| **DockerRuntime** | Docker 容器隔离（默认） |
| **LocalRuntime** | 本地机器执行（无隔离） |
| **RemoteRuntime** | 连接远程运行时服务器 |
| **KubernetesRuntime** | K8s 集群中执行 |

---

## 2. Base Runtime 抽象接口

### 2.1 Runtime 基类

**文件**: `/home/gdw/tutorial/OpenHands/openhands/runtime/base.py`

```python
class Runtime(ABC):
    sid: str                    # Session ID
    config: OpenHandsConfig     # 配置
    initial_env_vars: dict[str, str]  # 环境变量
    attach_to_existing: bool    # 是否附加到现有运行时
    status_callback: Callable   # 状态回调
    runtime_status: RuntimeStatus  # 当前状态
    security_analyzer: SecurityAnalyzer  # 安全分析
```

### 2.2 状态流转

```
STOPPED → BUILDING_RUNTIME → STARTING_RUNTIME → RUNTIME_STARTED →
SETTING_UP_WORKSPACE → SETTING_UP_GIT_HOOKS → READY
```

### 2.3 抽象方法

```python
class Runtime(ABC):
    @abstractmethod
    async def connect(self) -> None:
        """连接到运行时"""
        pass

    @abstractmethod
    def run(self, action: CmdRunAction) -> CmdOutputObservation:
        """执行 Bash 命令"""
        pass

    @abstractmethod
    def run_ipython(self, action: IPythonRunCellAction) -> IPythonRunCellObservation:
        """执行 Python 代码"""
        pass

    @abstractmethod
    def read(self, action: FileReadAction) -> FileReadObservation:
        """读取文件"""
        pass

    @abstractmethod
    def write(self, action: FileWriteAction) -> FileWriteObservation:
        """写入文件"""
        pass

    @abstractmethod
    def edit(self, action: FileEditAction) -> FileEditObservation:
        """编辑文件"""
        pass

    @abstractmethod
    def browse(self, action: BrowseURLAction) -> BrowserOutputObservation:
        """浏览 URL"""
        pass

    @abstractmethod
    def browse_interactive(self, action: BrowseInteractiveAction) -> BrowserOutputObservation:
        """交互浏览"""
        pass
```

---

## 3. DockerRuntime 实现

### 3.1 架构概述

**文件**: `/home/gdw/tutorial/OpenHands/openhands/runtime/impl/docker/docker_runtime.py`

DockerRuntime 继承自 `ActionExecutionClient`，通过 HTTP 与容器内的 ActionExecutionServer 通信。

```python
class DockerRuntime(ActionExecutionClient):
    # 端口范围
    EXECUTION_SERVER_PORT_RANGE = (30000, 39999)
    VSCODE_PORT_RANGE = (40000, 49999)
    APP_PORT_RANGE_1 = (50000, 54999)
    APP_PORT_RANGE_2 = (55000, 59999)
```

### 3.2 容器初始化流程

```python
async def connect(self) -> None:
    """连接到 Docker 容器"""
    if self.attach_to_existing:
        await self._attach_to_container()
    else:
        await self.init_container()

    await self.wait_until_alive()
    await self.setup_initial_env()
    await self.set_runtime_status(READY)
```

### 3.3 容器创建

**init_container()** 流程：

1. **分配端口**（带文件锁）
```python
port = self._allocate_port(
    self.EXECUTION_SERVER_PORT_RANGE,
    lock_dir=self.sandbox_dir
)
```

2. **配置环境变量**
```python
env = {
    'port': str(port),
    'VSCODE_PORT': str(vscode_port),
    'APP_PORT_1': str(app_port_1),
    'APP_PORT_2': str(app_port_2),
    'SESSION_API_KEY': self._get_session_api_key(),
    ...
}
```

3. **处理卷挂载**
```python
volumes = self._process_volumes(bind_mounts, overlay_mounts)
```

4. **创建并启动容器**
```python
container = self.docker_client.containers.run(
    image=self.runtime_base_image,
    command='...',
    volumes=volumes,
    ports={'30000/tcp': port},
    ...
)
```

### 3.4 卷挂载类型

```python
# 绑定挂载 - 直接映射主机路径
{
    'type': 'bind',
    'source': '/host/path',
    'target': '/container/path'
}

# Overlay 挂载 - 写时复制
{
    'type': 'overlay',
    'device': 'overlay',
    'o': 'lowerdir=/host/path,upperdir=/container/upper,workdir=/container/work'
}
```

**Overlay 机制**：
- lowerdir：只读底层（主机路径）
- upperdir：容器专属的可写层
- workdir：工作目录
- 实现 COW（Copy-on-Write）效率

---

## 4. LocalRuntime 实现

### 4.1 架构概述

**文件**: `/home/gdw/tutorial/OpenHands/openhands/runtime/impl/local/local_runtime.py`

LocalRuntime 直接在本地机器执行 action_execution_server，无沙箱隔离。

```python
class LocalRuntime(Runtime):
    _WARM_SERVERS: list[ActionExecutionServerInfo] = []  # 预热服务器池
    _RUNNING_SERVERS: dict[str, ActionExecutionServerInfo] = {}
```

### 4.2 预热池机制

```python
# 环境变量配置
INITIAL_NUM_WARM_SERVERS=0    # 启动时创建数量
DESIRED_NUM_WARM_SERVERS=0    # 目标池大小

def _create_warm_server(self) -> ActionExecutionServerInfo:
    """创建预热服务器"""
    server = ActionExecutionServer(...)
    server.start()  # 在后台线程启动
    return ActionExecutionServerInfo(
        pid=proc.pid,
        port=port,
        server=server,
        workspace_dir=workspace_dir
    )
```

**预热池流程**：
1. 启动时创建 `INITIAL_NUM_WARM_SERVERS` 个服务器
2. 使用时从池中取出
3. 使用后检查池大小，必要时补充

### 4.3 LocalRuntime 警告

```python
def __init__(self, ...):
    logger.warning(
        'Initializing LocalRuntime. WARNING: NO SANDBOX IS USED. '
        'This is an experimental feature...'
    )
```

**无隔离风险**：
- 命令直接在宿主机执行
- 有风险的操作可能影响主机
- 仅用于开发/测试

---

## 5. RemoteRuntime 与 KubernetesRuntime

### 5.1 RemoteRuntime

**文件**: `/home/gdw/tutorial/OpenHands/openhands/runtime/impl/remote/remote_runtime.py`

连接远程运行时服务器：

```python
class RemoteRuntime(Runtime):
    api_key: str                          # 认证
    remote_runtime_api_url: str           # API 端点
    remote_runtime_resource_factor: float  # 资源因子
```

### 5.2 KubernetesRuntime

**文件**: `/home/gdw/tutorial/OpenHands/openhands/runtime/impl/kubernetes/kubernetes_runtime.py`

在 K8s 集群中创建 Pod：

```python
class KubernetesRuntime(Runtime):
    _k8s_config: KubernetesConfig
    _namespace: str = 'openhands'
```

**创建的资源**：
- **Pod**：主运行时容器
- **Service**：ClusterIP 内部通信
- **PVC**：PersistentVolumeClaim 存储
- **Ingress**：外部访问 VSCode

**Pod 配置**：
```python
V1Pod(
    spec=V1PodSpec(
        containers=[V1Container(
            name='runtime',
            image=self.pod_image,
            ports=container_ports,
            volume_mounts=volume_mounts,
            resources=V1ResourceRequirements(
                limits={'memory': memory_limit},
                requests={'cpu': cpu_request, 'memory': memory_request}
            ),
            readiness_probe=health_check,
            security_context=V1SecurityContext(privileged=privileged)
        )],
        volumes=volumes,
        node_selector=node_selector,
        tolerations=tolerations
    )
)
```

---

## 6. Action Execution Server

### 6.1 HTTP API 端点

**文件**: `/home/gdw/tutorial/OpenHands/openhands/runtime/action_execution_server.py`

```python
class ActionExecutionServer:
    @app.get('/alive')
    async def alive(): ...

    @app.post('/execute_action')
    async def execute_action(action: Action): ...

    @app.post('/list_files')
    async def list_files(path: str): ...

    @app.post('/upload_file')
    async def upload_file(file: UploadFile, path: str): ...

    @app.get('/download_files')
    async def download_files(path: str): ...

    @app.post('/update_mcp_server')
    async def update_mcp_server(config: MCPConfig): ...
```

### 6.2 ActionExecutor

```python
class ActionExecutor:
    bash_session: BashSession           # TMUX 会话
    plugins: dict[str, Plugin]         # 插件
    file_editor: OHEditor              # 文件编辑器
    browser: BrowserEnv                 # 浏览器
    memory_monitor: MemoryMonitor        # 内存监控
```

**执行入口**：
```python
async def run_action(self, action) -> Observation:
    async with self.lock:  # 信号量保证单动作执行
        action_type = action.action
        observation = await getattr(self, action_type)(action)
        return observation
```

---

## 7. 动作执行机制

### 7.1 Bash 执行

**文件**: `/home/gdw/tutorial/OpenHands/openhands/runtime/utils/bash.py`

```python
class BashSession:
    session: libtmux.Session    # TMUX 会话
    window: libtmux.Window       # TMUX 窗口
    pane: libtmux.Pane           # TMUX 窗格

    # 超时配置
    NO_CHANGE_TIMEOUT_SECONDS: int = 30  # 无输出检测
    POLL_INTERVAL: float = 0.5            # 轮询间隔
```

**命令解析**：
```python
# 使用 bashlex 解析复合命令
import bashlex
ast = bashlex.parse(command)
# 处理分号分隔和 && 链接的命令
```

**输出提取**：
- 注入特殊 PS1 提示符 `[POH]`
- 正则匹配提取：exit_code、working_dir、timestamp
- 处理 TMUX 历史限制导致的截断

**超时处理**：
1. **无变化超时**：输出停滞 30 秒
2. **硬超时**：超过动作指定的超时时间
3. **Bash 会话忙**：exit_code=-1，最多 3 次重试

### 7.2 IPython/Jupyter 执行

**文件**: `/home/gdw/tutorial/OpenHands/openhands/runtime/plugins/jupyter/`

```
JupyterPlugin → JupyterKernel → Jupyter KernelGateway
```

**JupyterKernel**：
```python
class JupyterKernel:
    """通过 WebSocket 与 KernelGateway 通信"""

    async def execute(self, code: str) -> dict:
        # 1. 发送 execute_request
        msg = {
            'header': {'msg_type': 'execute_request', 'msg_id': uuid},
            'channel': 'shell',
            'content': {
                'code': code,
                'silent': False,
                'store_history': False
            }
        }
        await self.websocket.send(json.dumps(msg))

        # 2. 等待响应
        response = await self._wait_for_response(msg_id)

        # 3. 解析结果
        return {
            'output': response['content']['data'],
            'error': response['content'].get('ename')
        }
```

**支持的功能**：
- 标准输出/错误
- 图像输出（PNG base64）
- Rich 输出

### 7.3 文件操作

**读取** (`lines 449-516`)：
```python
def read(self, action: FileReadAction) -> FileReadObservation:
    # 1. 二进制文件检测
    if is_binary(path):
        content = base64.b64encode(open(path, 'rb').read()).decode()

    # 2. 支持范围读取
    if action.start or action.end:
        content = read_range(path, action.start, action.end)

    return FileReadObservation(content=content)
```

**写入** (`lines 518-570`)：
```python
def write(self, action: FileWriteAction) -> FileWriteObservation:
    # 1. 创建父目录
    os.makedirs(os.path.dirname(action.path), exist_ok=True)

    # 2. 写入文件
    with open(action.path, 'w') as f:
        f.write(action.content)

    # 3. 保持权限（可选）
    if hasattr(action, 'mode'):
        os.chmod(action.path, action.mode)
```

**编辑** (`lines 572-596`)：
```python
def edit(self, action: FileEditAction) -> FileEditObservation:
    # 委托给 OHEditor
    editor = OHEditor()
    diff = editor.edit_file(
        path=action.path,
        command=action.command,
        old_str=action.old_str,
        new_str=action.new_str,
        ...
    )
```

### 7.4 浏览器自动化

**文件**: `/home/gdw/tutorial/OpenHands/openhands/runtime/browser/browser_env.py`

基于 **BrowserGym** 框架：

```python
class BrowserEnv:
    """使用 BrowserGym 进行浏览器自动化"""

    def __init__(self):
        self.browser_process = ...
        self.browser_side, self.env_side = multiprocessing.Pipe()

    async def step(self, action: dict) -> tuple:
        # 发送动作到浏览器进程
        self.browser_side.send((request_id, action_data))

        # 等待结果
        result = self.env_side.recv()

        # 提取 DOM 内容
        html_str = flatten_dom_to_str(result['dom_object'])
        result['text_content'] = html2text.handle(html_str)

        return result
```

**支持的环境**：
- `browsergym/openended`：通用浏览
- `browsergym/webarena`：WebArena 基准
- `browsergym/visualwebarena`：视觉 WebArena
- `browsergym/miniwob`：MiniWoB++ 任务

---

## 8. 沙箱管理

### 8.1 容器生命周期

```
__init__()
    │
    ▼
connect()
    │
    ├──► _attach_to_container() [attach_to_existing=True]
    │        │
    │        └── 等待容器就绪
    │
    └──► init_container() [attach_to_existing=False]
             │
             ├── 分配端口
             ├── 创建卷挂载
             ├── 拉取镜像
             ├── 创建容器
             └── 启动容器
    │
    ▼
wait_until_alive()
    │
    ▼
setup_initial_env()
    │
    ▼
READY (可执行动作)
    │
    ▼
close()
    │
    ├── 停止容器
    └── 释放端口锁
```

### 8.2 预热池

**目的**：减少容器启动延迟

```
首次请求
    │
    ▼
检查预热池
    │
    ├──► 池中有可用服务器 → 直接使用
    │
    └──► 池为空
             │
             ├── 创建新服务器
             └── 返回
    │
    ▼
服务器使用后
    │
    ├──► 放回预热池 (如果池未满)
    │
    └──► 销毁 (如果池已满)
```

### 8.3 端口管理

**文件锁机制**：
```python
# openhands/runtime/utils/port_lock.py
import fcntl

def acquire_lock(lock_file):
    fd = open(lock_file, 'w')
    fcntl.flock(fd, fcntl.LOCK_EX)  # 阻塞直到获得锁
    return fd

def release_lock(fd):
    fcntl.flock(fd, fcntl.LOCK_UN)
    fd.close()
```

**端口分配**：
```python
# Linux/WSL
EXECUTION_SERVER_PORT_RANGE = (30000, 39999)
VSCODE_PORT_RANGE = (40000, 49999)

# Windows
EXECUTION_SERVER_PORT_RANGE = (30000, 34999)
VSCODE_PORT_RANGE = (35000, 39999)
```

---

## 9. 安全机制

### 9.1 隔离机制

| 运行时 | 隔离级别 |
|--------|----------|
| **DockerRuntime** | 容器级隔离 |
| **LocalRuntime** | 无隔离 |
| **KubernetesRuntime** | Pod 级隔离 + 资源限制 |
| **RemoteRuntime** | 远程服务器隔离 |

### 9.2 Docker 隔离

```python
# 容器安全配置
container = docker_client.containers.run(
    image=self.runtime_base_image,
    security_opt=['seccomp:default'],  # 安全计算
    network_mode='bridge',
    init=True,  # 使用 init 进程处理信号
    ...
)
```

### 9.3 资源限制

**Kubernetes 资源限制**：
```python
resources = V1ResourceRequirements(
    limits={'memory': '4Gi'},
    requests={'cpu': '500m', 'memory': '1Gi'}
)
```

**内存监控**：
```python
class MemoryMonitor:
    def check_memory(self) -> bool:
        usage = psutil.Process().memory_info().rss
        return usage < self.memory_limit
```

### 9.4 认证

**Session API Key**：
```python
# 所有请求需要验证
headers = {'X-Session-API-Key': SESSION_API_KEY}

# 中间件验证
@app.middleware
async def auth(request: Request, call_next):
    if request.url.path != '/alive':
        key = request.headers.get('X-Session-API-Key')
        if key != SESSION_API_KEY:
            raise HTTPException(401)
```

---

## 10. 架构流程图

### 10.1 动作执行完整流程

```
AgentController
    │
    ▼
EventStream.add_event(action) ──────────────┐
    │                                        │
    │                                        │ 发布 Action
    │                                        ▼
    │                              ┌─────────────────┐
    │                              │  EventStream    │
    │                              └────────┬────────┘
    │                                        │
    │  异步分发                              ▼
    │                              ┌─────────────────┐
    │                              │     Runtime     │
    │                              │                 │
    │                              │ on_event()      │
    │                              │ _handle_action()│
    │                              └────────┬────────┘
    │                                        │
    │                                        ▼
    │                              ┌─────────────────┐
    │                              │ ActionExecutor  │
    │                              │                 │
    │                              │ run_action()    │
    │                              └────────┬────────┘
    │                                        │
    │        ┌───────────────────────────────┼───────────────────────────────┐
    │        │                               │                               │
    │        ▼                               ▼                               ▼
    │  ┌─────────────┐              ┌─────────────┐              ┌─────────────┐
    │  │ BashSession │              │ JupyterKernel│              │ BrowserEnv  │
    │  │ (TMUX)      │              │ (WebSocket)  │              │ (Process)   │
    │  └──────┬──────┘              └──────┬───────┘              └──────┬──────┘
    │         │                            │                             │
    │         ▼                            ▼                             ▼
    │  ┌─────────────┐              ┌─────────────┐              ┌─────────────┐
    │  │  Shell      │              │  IPython    │              │  Browser    │
    │  │  Command    │              │  Kernel     │              │  Automation │
    │  └──────┬──────┘              └──────┬───────┘              └──────┬──────┘
    │         │                            │                             │
    │         └─────────────────────────────┼─────────────────────────────┘
    │                                       │
    ▼                                       ▼
Observation ◄──────────────────── EventStream.add_event(observation)
    │
    ▼
AgentController 处理
    │
    ▼
Agent.step() 决定下一步
```

### 10.2 DockerRuntime 架构

```
                    DockerRuntime
                    ┌─────────────────────────────────────┐
                    │                                     │
                    │  ┌─────────────────────────────┐   │
                    │  │     Port Allocation          │   │
                    │  │  30000-39999 (API)           │   │
                    │  │  40000-49999 (VSCode)       │   │
                    │  │  50000-59999 (App)          │   │
                    │  └─────────────────────────────┘   │
                    │                                     │
                    │  ┌─────────────────────────────┐   │
                    │  │   ActionExecutionClient     │   │
                    │  │   (HTTP Client)             │   │
                    │  └──────────────┬──────────────┘   │
                    └─────────────────┼───────────────────┘
                                        │
                    ┌───────────────────┴───────────────────┐
                    │           Docker Network              │
                    └───────────────────┬───────────────────┘
                                        │
                    ┌───────────────────┴───────────────────┐
                    │              Container                  │
                    │  ┌─────────────────────────────────┐   │
                    │  │  ActionExecutionServer (FastAPI)│   │
                    │  └──────────────┬──────────────────┘   │
                    │                 │                       │
                    │  ┌──────────────┴──────────────┐        │
                    │  │      ActionExecutor         │        │
                    │  ├─────────────────────────────┤        │
                    │  │ BashSession (TMUX)         │        │
                    │  │ JupyterPlugin (Kernel)      │        │
                    │  │ BrowserEnv (BrowserGym)     │        │
                    │  │ OHEditor                   │        │
                    │  └─────────────────────────────┘        │
                    │                                        │
                    │  ┌─────────────────────────────────┐   │
                    │  │     Volume Mounts               │   │
                    │  │  - /workspace (代码)            │   │
                    │  │  - /openhands (工具)           │   │
                    │  └─────────────────────────────────┘   │
                    └────────────────────────────────────────┘
```

### 10.3 预热池架构

```
                    ┌──────────────────────────────────────┐
                    │           LocalRuntime                │
                    │                                      │
                    │  ┌────────────────────────────────┐  │
                    │  │       Warm Pool Manager         │  │
                    │  │                                │  │
                    │  │  _WARM_SERVERS: [               │  │
                    │  │    ServerInfo(pid=1234, port=30000),│
                    │  │    ServerInfo(pid=5678, port=30001),│
                    │  │  ]                              │  │
                    │  └────────────────────────────────┘  │
                    │              │                         │
                    │              │ 取出一个                  │
                    │              ▼                         │
                    │  ┌────────────────────────────────┐  │
                    │  │    Request → Runtime           │  │
                    │  └────────────────────────────────┘  │
                    │              │                         │
                    │              │ 用完归还                 │
                    │              ▼                         │
                    │  ┌────────────────────────────────┐  │
                    │  │    Warm Pool Manager          │  │
                    │  │    (检查是否需要补充)          │  │
                    │  └────────────────────────────────┘  │
                    └──────────────────────────────────────┘
```

---

## 11. 面试问题

### Q1: Runtime 在 OpenHands 中扮演什么角色？

**参考答案**：
> Runtime 是 OpenHands 的"手脚"，负责**在实际环境中执行 Action**。
>
> **核心职责**：
> 1. **环境隔离**：在 Docker/K8s 等隔离环境中执行
> 2. **动作执行**：Bash 命令、Python 代码、文件操作、浏览器
> 3. **结果返回**：将执行结果作为 Observation 返回
>
> **与 AgentController 的关系**：
> - Controller 驱动主循环，决定做什么
> - Runtime 执行动作，报告结果
> - 两者通过 EventStream 通信

---

### Q2: DockerRuntime 是如何实现的？

**参考答案**：
> **架构**：
> - DockerRuntime 继承 ActionExecutionClient
> - 通过 HTTP 与容器内的 ActionExecutionServer 通信
>
> **创建流程**：
> 1. **端口分配**：带文件锁防止冲突
> 2. **环境配置**：设置 port、API key 等
> 3. **卷挂载**：
>    - 绑定挂载：主机路径直接映射
>    - Overlay 挂载：COW 机制
> 4. **容器创建**：使用 docker-py 库
>
> **隔离机制**：
> - 独立容器文件系统
> - 网络隔离（bridge 模式）
> - 资源限制（CPU、内存）

---

### Q3: 什么是 ActionExecutionServer？它做什么？

**参考答案**：
> **ActionExecutionServer** 是运行在沙箱内的 HTTP 服务器。
>
> **API 端点**：
> ```python
> GET  /alive              # 健康检查
> POST /execute_action     # 执行动作
> POST /list_files        # 列出文件
> POST /upload_file       # 上传文件
> GET  /download_files    # 下载文件
> POST /update_mcp_server # 更新 MCP
> ```
>
> **ActionExecutor**：
> - 管理 BashSession（TMUX）
> - 管理 JupyterPlugin
> - 管理 BrowserEnv
> - 使用信号量保证单动作执行

---

### Q4: Bash 命令是如何执行的？

**参考答案**：
> **BashSession** 使用 TMUX 实现：
> ```python
> class BashSession:
>     session = libtmux.Session()
>     window = session.new_window()
>     pane = window.attached_pane
>
>     def run_command(self, cmd):
>         pane.send_keys(cmd)
>         # 轮询输出直到完成
> ```
>
> **输出提取**：
> - 注入特殊 PS1 提示符 `[POH]`
> - 解析 exit_code、working_dir
>
> **超时处理**：
> - 无变化超时：30 秒无输出
> - 硬超时：动作指定的超时
> - Bash 会话忙：exit_code=-1 重试

---

### Q5: IPython/Jupyter 是如何工作的？

**参考答案**：
> **架构**：
> ```
> JupyterPlugin → JupyterKernelGateway → Kernel
> ```
>
> **通信**：WebSocket 协议
> ```python
> # 发送
> {
>     'header': {'msg_type': 'execute_request'},
>     'content': {'code': 'print("hello")'}
> }
>
> # 接收
> {
>     'header': {'msg_type': 'execute_result'},
>     'content': {'data': 'hello'}
> }
> ```
>
> **支持**：
> - stdout/stderr 输出
> - 图像输出（base64）
> - Rich 输出

---

### Q6: 什么是预热池？为什么要用它？

**参考答案**：
> **预热池**是预先启动的 ActionExecutionServer 池。
>
> **问题**：首次执行需要启动容器（几十秒），太慢
>
> **解决方案**：
> ```python
> _WARM_SERVERS = []  # 预热服务器列表
>
> # 启动时创建
> for _ in range(INITIAL_NUM_WARM_SERVERS):
>     server = create_server()
>     server.start()
>     _WARM_SERVERS.append(server)
>
> # 使用时取出
> if _WARM_SERVERS:
>     server = _WARM_SERVERS.pop()
> else:
>     server = create_server()  # 没有预热服务器时创建新的
> ```
>
> **优势**：任务立即开始，无需等待容器启动

---

### Q7: LocalRuntime 和 DockerRuntime 的区别是什么？

**参考答案**：
>
> | 维度 | DockerRuntime | LocalRuntime |
> |------|---------------|--------------|
> | **隔离** | 容器级隔离 | 无隔离 |
> | **启动延迟** | 几十秒 | 几乎即时 |
> | **资源** | 需 Docker 守护进程 | 直接在主机 |
> | **安全性** | 高 | 低（命令直接运行） |
> | **一致性** | 环境固定 | 依赖主机环境 |
> | **适用** | 生产/信任边界 | 开发/测试 |
>
> **LocalRuntime 警告**：
> ```python
> logger.warning('NO SANDBOX IS USED. This is an experimental feature...')
> ```

---

### Q8: 容器是如何启动和管理的？

**参考答案**：
> **启动流程**：
> ```python
> # 1. 创建容器
> container = docker_client.containers.run(
>     image='openhands/runtime:latest',
>     command='python -m openhands.runtime.action_execution_server',
>     volumes={'/workspace': {'bind': '/workspace', 'mode': 'rw'}},
>     ports={'30000/tcp': 30000}
> )
>
> # 2. 等待就绪
> container.start()
> time.sleep(5)  # 或健康检查
>
> # 3. 停止容器
> container.stop()
>
> # 4. 清理
> container.remove()
> ```
>
> **生命周期**：
> - 创建 → 启动 → 就绪 → 执行 → 停止 → 删除

---

### Q9: 安全机制有哪些？

**参考答案**：
>
> **隔离安全**：
> - Docker 容器隔离
> - K8s Pod 隔离
> - 网络隔离
>
> **资源限制**：
> - 内存限制
> - CPU 限制
> - 内存监控（MemoryMonitor）
>
> **认证**：
> - Session API Key 验证
> - 所有请求头携带 `X-Session-API-Key`
>
> **操作安全**：
> - 高风险动作需用户确认（confirmation_mode）
> - SecurityAnalyzer 分析动作风险
> - hidden=True 的敏感操作不记录

---

### Q10: 如何扩展新的 Runtime 实现？

**参考答案**：
> **步骤**：
>
> 1. **继承 Runtime 基类**
> ```python
> class MyRuntime(Runtime):
>     async def connect(self):
>         # 连接逻辑
>         pass
>
>     def run(self, action: CmdRunAction):
>         # 命令执行
>         pass
> ```
>
> 2. **实现抽象方法**
> ```python
> def read(self, action: FileReadAction):
>     ...
>
> def write(self, action: FileWriteAction):
>     ...
> ```
>
> 3. **注册到 Builder**
> ```python
> # runtime/builder/base.py
> RuntimeBuilder.register('my_runtime', MyRuntimeBuilder)
> ```

---

### Q11: 文件编辑是如何实现的？

**参考答案**：
> **OHEditor** 提供高级文件编辑：
> ```python
> editor = OHEditor()
>
> # 替换编辑
> editor.edit_file(
>     path='src/main.py',
>     old_str='def hello():',
>     new_str='def hello_world():',
> )
>
> # 插入编辑
> editor.edit_file(
>     path='src/main.py',
>     command='insert',
>     new_str='    return "hello"',
>     after_line=10,
> )
> ```
>
> **Diff 生成**：
> - 使用 `openhands_aci.utils.diff`
> - 生成 unified diff 格式
> - 可用于预览和回滚

---

### Q12: 浏览器自动化是如何工作的？

**参考答案**：
> **BrowserGym** 是 OpenAI 开发的浏览器自动化框架：
>
> **架构**：
> ```python
> BrowserEnv
>     │
>     ├──► Browser Process (multiprocessing)
>     │          │
>     │          ├──► BrowserGym Environment
>     │          │          │
>     │          │          ├──► Chromium
>     │          │          │
>     │          │          └──► accessibility_tree
>     │          │
>     │          └──► Pipe (IPC)
>     │
>     └──► ActionExecutor
> ```
>
> **动作类型**：
> - `goto(url)`：导航
> - `click(selector)`：点击
> - `type(selector, text)`：输入
> - `noop()`：无操作（等待页面稳定）
>
> **观察**：
> - DOM 树
> - Accessibility tree（无障碍树）
> - 截图
