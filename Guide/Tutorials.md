[首页](../README.md)

# 使用教程

对于想要基于ACPs协议体系开发智能体的开发者，请参考本教程。

教程主要包括以下三个部分：

- [构建支持ACPs的智能体](#1-构建支持acps的智能体)
- [智能体可信注册](#2-智能体可信注册)
- [体验基于ACPs的智能体互联应用（demo）](#3-体验基于acps的智能体互联应用demo)

---

## 1. 构建支持ACPs的智能体

本教程介绍如何使用 `acps_sdk` 进行 AIP（Agent Interaction Protocol，智能体交互协议）开发。我们将详细讲解 AIP 的基本概念、SDK 的主要内容和边界，以及如何利用 SDK 开发 Partner 和 Leader 智能体。

> **参考文档**: [ACPs-spec-AIP.md](../../acps-specs/07-ACPs-spec-AIP/ACPs-spec-AIP.md) - AIP 协议规范完整定义

### 1.1 AIP 协议基本概念

AIP（Agent Interaction Protocol）是 ACPs 协议体系中用于规范智能体之间交互的核心协议。它定义了智能体如何协作、委托任务和交换信息。

#### 1.1.1 角色定义

AIP 协议定义了两种核心角色：

| 角色                  | 说明                                                                                      |
| --------------------- | ----------------------------------------------------------------------------------------- |
| **Leader（领导者）**  | 发布任务并组织交互的智能体。在一次完整的交互中，只能有一个 Leader。负责任务的创建与终止。 |
| **Partner（参与者）** | 接受任务并提供服务的智能体。Partner 接受来自 Leader 的任务后执行并返回执行结果。          |

在本项目中：

- **Leader**: 位于 `leader/` 目录，作为用户交互的桥梁，负责协调多个 Partner 完成复杂任务
- **Partner**: 位于 `partners/` 目录，每个online子目录是一个专业领域的 Partner（如北京城区景点、美食推荐等）

#### 1.1.2 交互模式

AIP 支持三种交互模式：

##### (1) 直连模式（Direct Interaction Mode）

Leader 直接与每个 Partner 进行一对一交互，Partner 之间无交互。这是最基础的交互模式。

![](./tutorials/pic/直连.png)

**实现方式**:

- **RPC 方式**: 基础的请求-响应模式
- **流式传输（SSE）**: 适用于大数据或实时推送
- **异步通知**: 基于回调的非阻塞通信

##### (2) 群组模式（Grouping Interaction Mode）

Leader 创建群组，通过消息队列（如 RabbitMQ）进行消息分发。所有群组成员均可发送和接收消息。

![](./tutorials/pic/群组.png)

**特点**:

- 消息透明：所有成员可见群组内消息
- 通过 `mentions` 字段指定消息接收者
- 支持多 Partner 协作

##### (3) 混合模式（Hybrid Interaction Mode）

在同一 Session 内，Leader 可以同时使用直连模式和群组模式与不同 Partner 交互。

#### 1.1.3 核心数据对象

AIP 协议定义了一套完整的数据对象体系，所有对象都继承自 `Message` 基类。

##### Message（消息基类）

所有交互数据的基类，定义了通用的消息属性：

```python
from acps_sdk.aip import Message

# Message 是所有交互数据的基类
class Message(BaseModel):
    type: str = "message"           # 对象类型标识符
    id: str                         # 消息唯一标识符
    sentAt: str                     # 发送时间 (ISO 8601)
    senderRole: Literal["leader", "partner"]  # 发送者角色
    senderId: str                   # 发送者的 AIC
    mentions: Optional[Union[Literal["all"], List[str]]] = None  # 群组模式下的提及
    dataItems: Optional[List[DataItem]] = None  # 消息内容
    groupId: Optional[str] = None   # 群组 ID
    sessionId: Optional[str] = None # 会话 ID
```

##### TaskCommand（任务命令）

Leader 向 Partner 发送的任务控制命令：

```python
from acps_sdk.aip import TaskCommand, TaskCommandType

# 创建一个启动任务的命令
command = TaskCommand(
    id="msg-001",
    sentAt="2025-01-27T10:00:00+08:00",
    senderRole="leader",
    senderId="leader-aic-001",
    command=TaskCommandType.Start,  # 命令类型
    taskId="task-001",
    sessionId="session-001",
    dataItems=[TextDataItem(text="请推荐北京的景点")]
)
```

**命令类型 (TaskCommandType)**:

| 命令       | 说明                                                   |
| ---------- | ------------------------------------------------------ |
| `Start`    | 开始任务                                               |
| `Continue` | 继续任务（用于 AwaitingInput/AwaitingCompletion 状态） |
| `Cancel`   | 取消任务                                               |
| `Complete` | 完成任务（确认 Partner 的产出物）                      |
| `Get`      | 获取任务当前状态                                       |
| `ReStream` | 重连流式传输                                           |

##### TaskResult（任务结果）

Partner 向 Leader 返回的任务状态和结果：

```python
from acps_sdk.aip import TaskResult, TaskStatus, TaskState, Product

# 创建一个任务结果
result = TaskResult(
    id="msg-002",
    sentAt="2025-01-27T10:00:01+08:00",
    senderRole="partner",
    senderId="partner-aic-001",
    taskId="task-001",
    status=TaskStatus(
        state=TaskState.AwaitingCompletion,
        stateChangedAt="2025-01-27T10:00:01+08:00",
    ),
    products=[
        Product(
            id="product-001",
            name="景点推荐",
            dataItems=[TextDataItem(text="推荐您参观故宫...")]
        )
    ],
    sessionId="session-001"
)
```

##### DataItem（数据项）

消息或产出物中的内容片段，支持三种类型：

```python
from acps_sdk.aip import TextDataItem, FileDataItem, StructuredDataItem

# 文本数据项
text_item = TextDataItem(text="这是一段文本")

# 文件数据项
file_item = FileDataItem(
    name="image.jpg",
    mimeType="image/jpeg",
    bytes="base64编码内容..."  # 或使用 uri 指向外部文件
)

# 结构化数据项
data_item = StructuredDataItem(data={"key": "value", "list": [1, 2, 3]})
```

#### 1.1.4 任务状态机

任务在生命周期中经历多个状态，状态转移遵循严格的规则：

![](./tutorials/pic/状态机.png)

**状态说明**:

| 状态                 | 说明                                   |
| -------------------- | -------------------------------------- |
| `Accepted`           | Partner 接受了任务，准备开始处理       |
| `Rejected`           | Partner 拒绝了任务（如超出能力范围）   |
| `Working`            | 任务正在执行中                         |
| `AwaitingInput`      | Partner 需要更多信息才能继续           |
| `AwaitingCompletion` | Partner 已生成产出物，等待 Leader 确认 |
| `Completed`          | 任务成功完成（终态）                   |
| `Failed`             | 任务执行失败（终态）                   |
| `Canceled`           | 任务被取消（终态）                     |

---

### 1.2 SDK 架构

#### 1.2.1 SDK 模块结构

`acps_sdk.aip` 包提供了 AIP 协议的完整实现：

```
acps_sdk/aip/
├── __init__.py              # 公共导出
├── aip_base_model.py        # 基础数据模型 (Message, TaskCommand, TaskResult 等)
├── aip_rpc_model.py         # RPC 数据模型 (JSONRPCRequest/Response)
├── aip_stream_model.py      # 流式传输数据模型 (SSE 事件)
├── aip_rpc_client.py        # RPC 客户端 (Leader 端使用)
├── aip_rpc_server.py        # RPC 服务端框架 (Partner 端使用)
├── aip_group_model.py       # 群组模式数据模型
├── aip_group_leader.py      # 群组模式 Leader 客户端
├── aip_group_partner.py     # 群组模式 Partner 客户端
└── mtls_config.py           # mTLS 配置
```

---

### 1.3 直连模式开发

#### 1.3.1 Partner 端开发

Partner 端使用 SDK 提供的服务端框架响应 Leader 的 RPC 请求。

##### 1.3.1.1 基本架构

```python
# partners/main.py - Partner 服务入口
from fastapi import FastAPI
from acps_sdk.aip.aip_rpc_model import RpcRequest, RpcResponse

app = FastAPI()

@app.post("/partners/{agent_name}/rpc", response_model=RpcResponse)
async def rpc_endpoint(agent_name: str, request: RpcRequest):
    # 分发到对应的 Agent 处理
    return await manager.dispatch(agent_name, request)
```

##### 1.3.1.2 使用 CommandHandlers 框架

SDK 提供了 `CommandHandlers` 框架来处理不同的任务命令：

```python
# partners/generic_runner.py
from acps_sdk.aip.aip_rpc_server import CommandHandlers, DefaultHandlers
from acps_sdk.aip.aip_base_model import TaskCommand, TaskResult, TaskState

class GenericRunner:
    def __init__(self, agent_name: str, base_dir: str):
        self.agent_name = agent_name

        # 注册命令处理器
        self.handlers = CommandHandlers(
            on_start=self.on_start,      # 处理 Start 命令
            on_get=self.on_get,          # 处理 Get 命令
            on_cancel=self.on_cancel,    # 处理 Cancel 命令
            on_complete=self.on_complete,# 处理 Complete 命令
            on_continue=self.on_continue,# 处理 Continue 命令
        )

    async def on_start(self, command: TaskCommand, task: Optional[TaskResult]) -> TaskResult:
        """处理 Start 命令 - 创建新任务"""
        # 1. 意图识别与准入判断
        decision = await self._decision_phase(command)

        if decision.action == "reject":
            # 拒绝任务
            return self._create_rejected_result(command, decision.reason)

        # 2. 创建并返回 Accepted 状态的任务
        task = self._create_task(command, TaskState.Accepted)

        # 3. 异步开始执行任务
        asyncio.create_task(self._execute_task(command.taskId))

        return task

    async def on_continue(self, command: TaskCommand, task: TaskResult) -> TaskResult:
        """处理 Continue 命令 - 继续执行任务"""
        # 检查当前状态是否允许 continue
        if task.status.state not in (TaskState.AwaitingInput, TaskState.AwaitingCompletion):
            return task  # 忽略无效的 continue

        # 更新任务状态为 Working
        return self._update_task_status(command.taskId, TaskState.Working)
```

##### 1.3.1.3 任务生命周期实现

以下是一个典型的 Partner 任务处理流程：

```python
async def _execute_task(self, task_id: str):
    """执行任务的完整生命周期"""
    try:
        # 阶段 1: 更新为 Working 状态
        self._update_task_status(task_id, TaskState.Working)

        # 阶段 2: 需求分析
        requirements = await self._analyze_requirements(task_id)

        if requirements.needs_more_info:
            # 信息不足，等待用户输入
            self._update_task_status(
                task_id,
                TaskState.AwaitingInput,
                data_items=[TextDataItem(text=requirements.question)]
            )
            return  # 等待 Continue 命令

        # 阶段 3: 生成产出物
        product = await self._generate_product(task_id, requirements)

        # 阶段 4: 提交产出物，等待确认
        task = self.tasks[task_id].task
        task.products = [product]
        self._update_task_status(task_id, TaskState.AwaitingCompletion)

    except Exception as e:
        # 任务失败
        self._update_task_status(
            task_id,
            TaskState.Failed,
            data_items=[TextDataItem(text=f"执行失败: {str(e)}")]
        )
```

##### 1.3.1.4 配置文件结构

每个 Partner Agent 需要以下配置文件：

```
partners/online/<agent_name>/
├── acs.json       # ACS 定义 (身份与能力)
├── config.toml    # 运行配置 (LLM、日志等)
└── prompts.toml   # Prompt 模板 (业务逻辑)
```

**acs.json 示例**:

```json
{
  "aic": "1.2.156.3088.1.34C2.478BDF.3GF546.1.0SEN",
  "name": "北京城区旅游智能体",
  "description": "为北京城六区提供景点推荐和行程规划服务",
  "capabilities": {
    "streaming": false,
    "notification": false,
    "messageQueue": ["rabbitmq:4.2"]
  },
  "skills": [
    {
      "id": "beijing_urban.cultural-attraction-recommendation",
      "name": "文化景点推荐",
      "description": "推荐北京城区的历史文化景点"
    }
  ]
}
```

#### 1.3.2 Leader 端开发

Leader 端使用 `AipRpcClient` 与 Partner 通信。

##### 1.3.2.1 使用 AipRpcClient

```python
# leader/assistant/core/executor.py
from acps_sdk.aip.aip_rpc_client import AipRpcClient
from acps_sdk.aip.aip_base_model import TaskResult, TaskState

class TaskExecutor:
    def __init__(self, leader_aic: str):
        self.leader_aic = leader_aic
        self._rpc_clients: Dict[str, AipRpcClient] = {}

    async def _get_client(self, partner_url: str) -> AipRpcClient:
        """获取或创建 RPC 客户端"""
        if partner_url not in self._rpc_clients:
            self._rpc_clients[partner_url] = AipRpcClient(
                partner_url=partner_url,
                leader_id=self.leader_aic
            )
        return self._rpc_clients[partner_url]

    async def start_task(self, partner_url: str, session_id: str, user_input: str) -> TaskResult:
        """向 Partner 发起任务"""
        client = await self._get_client(partner_url)
        return await client.start_task(session_id, user_input)

    async def continue_task(self, partner_url: str, task_id: str, session_id: str, user_input: str) -> TaskResult:
        """继续 Partner 的任务"""
        client = await self._get_client(partner_url)
        return await client.continue_task(task_id, session_id, user_input)

    async def complete_task(self, partner_url: str, task_id: str, session_id: str) -> TaskResult:
        """确认 Partner 的产出物"""
        client = await self._get_client(partner_url)
        return await client.complete_task(task_id, session_id)
```

##### 1.3.2.2 轮询模式执行

Leader 通常需要轮询 Partner 状态直到任务收敛：

```python
async def execute_until_converged(
    self,
    session_id: str,
    partner_tasks: Dict[str, PartnerTask],
) -> ExecutionResult:
    """执行任务直到所有 Partner 状态收敛"""

    result = ExecutionResult(phase=ExecutionPhase.STARTING)

    # Phase 1: 并发下发 start 命令
    start_tasks = [
        self._start_partner(session_id, task)
        for task in partner_tasks.values()
    ]
    await asyncio.gather(*start_tasks)

    # Phase 2: 轮询直到收敛
    start_time = datetime.now()
    while not self._is_converged(result):
        if (datetime.now() - start_time).seconds > self.config.convergence_timeout_s:
            result.phase = ExecutionPhase.TIMEOUT
            break

        # 获取所有 Partner 的最新状态
        await self._poll_all_partners(partner_tasks, result)

        # 检查是否有 Partner 需要输入
        if result.awaiting_input_partners:
            result.phase = ExecutionPhase.AWAITING_INPUT
            break

        # 检查是否所有 Partner 都完成或等待确认
        if self._all_awaiting_completion(result):
            result.phase = ExecutionPhase.AWAITING_COMPLETION
            break

        await asyncio.sleep(self.config.poll_interval_ms / 1000)

    return result

def _is_converged(self, result: ExecutionResult) -> bool:
    """检查是否所有任务都达到终态"""
    terminal_states = {TaskState.Completed, TaskState.Failed, TaskState.Canceled, TaskState.Rejected}
    for partner_result in result.partner_results.values():
        if partner_result.state not in terminal_states:
            return False
    return True
```

---

### 1.4 群组模式开发

群组模式通过 RabbitMQ 消息队列实现多 Partner 协作。

#### 1.4.1 Partner 端开发

##### 1.4.1.1 处理群组邀请

Partner 需要响应 Leader 的群组邀请请求：

```python
# partners/group_handler.py
from acps_sdk.aip.aip_group_partner import GroupPartnerMqClient, PartnerGroupState
from acps_sdk.aip.aip_group_model import RabbitMQRequest, RabbitMQResponse

class GroupHandler:
    def __init__(self, agent_name: str, runner: GenericRunner):
        self.agent_name = agent_name
        self.runner = runner
        self.partner_aic = runner.acs.get("aic")

        # 群组客户端缓存 (group_id -> client)
        self._group_clients: Dict[str, GroupPartnerMqClient] = {}

    async def handle_group_rpc(self, request: RabbitMQRequest) -> RabbitMQResponse:
        """处理群组邀请请求"""
        if request.method == "group":
            return await self._handle_join_group(request)
        return RabbitMQResponse(
            id=request.id,
            error={"code": -32601, "message": "Method not found"}
        )

    async def _handle_join_group(self, request: RabbitMQRequest) -> RabbitMQResponse:
        """加入群组"""
        group_id = request.params.group.groupId

        # 创建群组客户端
        client = GroupPartnerMqClient(partner_aic=self.partner_aic)

        # 设置消息处理器
        client.set_command_handler(self._on_task_command)
        client.set_task_result_handler(self._on_task_result)
        client.set_mgmt_command_handler(self._on_mgmt_command)

        # 加入群组
        response = await client.join_group(request)

        if response.result:
            self._group_clients[group_id] = client

        return response
```

##### 1.4.1.2 处理群组任务命令

```python
async def _on_task_command(self, command: TaskCommand, is_mentioned: bool) -> None:
    """处理来自群组的任务命令"""
    # 如果没有被提及，不处理
    if not is_mentioned:
        return

    # 记录任务到群组的映射
    self._task_group_map[command.taskId] = command.groupId

    # 根据命令类型处理
    if command.command == TaskCommandType.Start:
        # 处理 start 命令
        task_result = await self.runner.on_start(command, None)
        # 广播状态更新到群组
        await self._broadcast_task_update(task_result, command.groupId)

    elif command.command == TaskCommandType.Continue:
        task_ctx = self.runner.tasks.get(command.taskId)
        if task_ctx:
            task_result = await self.runner.on_continue(command, task_ctx.task)
            await self._broadcast_task_update(task_result, command.groupId)
```

##### 1.4.1.3 发送任务状态更新

```python
async def _broadcast_task_update(self, task_result: TaskResult, group_id: str) -> None:
    """广播任务状态更新到群组"""
    client = self._group_clients.get(group_id)
    if not client or not client.is_joined:
        return

    # 使用 SDK 方法发送状态更新
    await client.send_task_result(
        task_id=task_result.taskId,
        session_id=task_result.sessionId,
        state=task_result.status.state,
        products=task_result.products,
        status_data_items=task_result.status.dataItems,
    )
```

#### 1.4.2 Leader 端开发

##### 1.4.2.1 群组管理器

```python
# leader/assistant/core/group_manager.py
from acps_sdk.aip.aip_group_leader import GroupLeader, GroupLeaderMqClient, PartnerConnectionInfo
from acps_sdk.aip.aip_group_model import ACSObject

class GroupManager:
    def __init__(
        self,
        leader_aic: str,
        rabbitmq_config: RabbitMQConfig,
    ):
        self.leader_aic = leader_aic
        self.rabbitmq_config = rabbitmq_config

        # SDK GroupLeader 实例
        self._group_leader: Optional[GroupLeader] = None

        # Session ID -> 群组 ID 映射
        self._session_group_map: Dict[str, str] = {}

    async def start(self) -> None:
        """启动群组管理器"""
        self._group_leader = GroupLeader(
            leader_aic=self.leader_aic,
            rabbitmq_config={
                "host": self.rabbitmq_config.host,
                "port": self.rabbitmq_config.port,
                "user": self.rabbitmq_config.user,
                "password": self.rabbitmq_config.password,
                "vhost": self.rabbitmq_config.vhost,
            },
        )

    async def create_group_for_session(self, session_id: str) -> str:
        """为 Session 创建群组"""
        if session_id in self._session_group_map:
            return self._session_group_map[session_id]

        group_session = await self._group_leader.create_group_session(
            session_id=session_id,
            initial_partners=[],
        )

        self._session_group_map[session_id] = group_session.group_id
        return group_session.group_id
```

##### 1.4.2.2 邀请 Partner 加入群组

```python
async def invite_partner(
    self,
    session_id: str,
    partner_acs: ACSObject,
    partner_rpc_url: str,
) -> PartnerConnectionInfo:
    """邀请 Partner 加入群组"""
    group_session = await self._get_group_session(session_id)

    # 使用 SDK 邀请 Partner
    partner_info = await group_session.mq_client.invite_partner(
        partner_acs=partner_acs,
        partner_rpc_url=partner_rpc_url,
        timeout=self.config.partner_join_timeout,
    )

    logger.info(f"Partner {partner_acs.aic} joined group {group_session.group_id}")
    return partner_info
```

##### 1.4.2.3 群组模式任务执行器

```python
# leader/assistant/core/group_executor.py
class GroupTaskExecutor:
    """群组模式任务执行器"""

    def __init__(self, leader_aic: str, group_manager: GroupManager):
        self.leader_aic = leader_aic
        self.group_manager = group_manager

    async def execute(
        self,
        session_id: str,
        active_task_id: str,
        planning_result: PlanningResult,
    ) -> ExecutionResult:
        """执行任务（群组模式）"""

        result = ExecutionResult(phase=ExecutionPhase.STARTING)

        # Phase 0: 确保群组已创建
        group_id = await self.group_manager.create_group_for_session(session_id)

        # Phase 1: 邀请未加入群组的 Partner
        for partner in planning_result.selected_partners:
            if not await self._is_partner_in_group(session_id, partner.aic):
                await self.group_manager.invite_partner(
                    session_id,
                    ACSObject(aic=partner.aic),
                    partner.rpc_url,
                )

        # Phase 2: 发送 start 命令到群组
        group_session = await self.group_manager._get_group_session(session_id)

        for partner in planning_result.selected_partners:
            await group_session.mq_client.start_task(
                session_id=session_id,
                text_content=partner.task_content,
                task_id=f"{active_task_id}-{partner.aic}",
                mentions=[partner.aic],  # 只提及特定 Partner
            )

        # Phase 3: 等待状态收敛（通过消息队列接收更新）
        result = await self._wait_until_converged(session_id, planning_result, result)

        return result
```

##### 1.4.2.4 处理群组消息

```python
async def _setup_message_handler(self, session_id: str):
    """设置群组消息处理器"""
    group_session = await self.group_manager._get_group_session(session_id)

    async def handle_message(message):
        if isinstance(message, TaskResult):
            await self._on_task_result(session_id, message)
        elif isinstance(message, GroupMgmtResult):
            await self._on_mgmt_result(session_id, message)

    group_session.mq_client.set_message_handler(handle_message)

async def _on_task_result(self, session_id: str, result: TaskResult):
    """处理 Partner 的任务状态更新"""
    partner_aic = result.senderId
    task_id = result.taskId
    state = result.status.state

    # 更新本地状态缓存
    self._update_partner_state(session_id, partner_aic, task_id, state)

    # 检查是否需要触发回调
    if state == TaskState.AwaitingCompletion:
        # Partner 提交了产出物
        products = result.products
        await self._handle_products(session_id, partner_aic, products)

    elif state == TaskState.AwaitingInput:
        # Partner 需要更多输入
        question = self._extract_question(result)
        await self._handle_question(session_id, partner_aic, question)
```

---

### 1.5 最简实战示例

#### 1.5.1 完整的 Partner 实现示例

以下是一个简化的 Partner 完整实现：

```python
from fastapi import FastAPI
from acps_sdk.aip import (
    TaskCommand, TaskResult, TaskStatus, TaskState,
    TaskCommandType, TextDataItem, Product
)
from acps_sdk.aip.aip_rpc_model import RpcRequest, RpcResponse
from datetime import datetime, timezone
import uuid

app = FastAPI()

# 任务存储
tasks: Dict[str, TaskResult] = {}

def create_task_result(
    command: TaskCommand,
    state: TaskState,
    products: List[Product] = None,
    message: str = None
) -> TaskResult:
    """创建任务结果"""
    data_items = [TextDataItem(text=message)] if message else None

    return TaskResult(
        id=f"msg-{uuid.uuid4()}",
        sentAt=datetime.now(timezone.utc).isoformat(),
        senderRole="partner",
        senderId="example-agent-aic",
        taskId=command.taskId,
        status=TaskStatus(
            state=state,
            stateChangedAt=datetime.now(timezone.utc).isoformat(),
            dataItems=data_items,
        ),
        products=products,
        sessionId=command.sessionId,
    )

@app.post("/rpc", response_model=RpcResponse)
async def handle_rpc(request: RpcRequest) -> RpcResponse:
    command = request.params.command
    task_id = command.taskId

    # 处理 Start 命令
    if command.command == TaskCommandType.Start:
        # 检查是否应该接受任务
        user_input = command.dataItems[0].text if command.dataItems else ""

        if "天气" in user_input:  # 示例：拒绝天气相关请求
            result = create_task_result(
                command, TaskState.Rejected,
                message="抱歉，我不提供天气查询服务"
            )
        else:
            # 接受任务并立即生成产出物
            product = Product(
                id=f"product-{uuid.uuid4()}",
                name="回复",
                dataItems=[TextDataItem(text=f"收到您的请求: {user_input}")]
            )
            result = create_task_result(
                command, TaskState.AwaitingCompletion,
                products=[product]
            )

        tasks[task_id] = result
        return RpcResponse(id=request.id, result=result)

    # 处理 Complete 命令
    elif command.command == TaskCommandType.Complete:
        task = tasks.get(task_id)
        if task and task.status.state == TaskState.AwaitingCompletion:
            task.status = TaskStatus(
                state=TaskState.Completed,
                stateChangedAt=datetime.now(timezone.utc).isoformat(),
            )
        return RpcResponse(id=request.id, result=task)

    # 处理 Get 命令
    elif command.command == TaskCommandType.Get:
        task = tasks.get(task_id)
        return RpcResponse(id=request.id, result=task)

    # 其他命令...
    return RpcResponse(id=request.id, result=tasks.get(task_id))
```

#### 1.5.2 Leader 调用 Partner 示例

```python
import asyncio
from acps_sdk.aip import AipRpcClient, TaskState

async def main():
    # 创建 RPC 客户端
    client = AipRpcClient(
        partner_url="http://localhost:8011/partners/beijing_urban/rpc",
        leader_id="leader-example-aic"
    )

    try:
        # 1. 启动任务
        session_id = "session-001"
        result = await client.start_task(
            session_id=session_id,
            user_input="请推荐北京城区的文化景点"
        )

        print(f"任务状态: {result.status.state}")
        task_id = result.taskId

        # 2. 检查任务状态
        if result.status.state == TaskState.AwaitingCompletion:
            # Partner 已生成产出物
            for product in result.products or []:
                for item in product.dataItems:
                    if hasattr(item, 'text'):
                        print(f"产出物: {item.text}")

            # 3. 确认完成
            final_result = await client.complete_task(task_id, session_id)
            print(f"最终状态: {final_result.status.state}")

        elif result.status.state == TaskState.AwaitingInput:
            # Partner 需要更多信息
            question = result.status.dataItems[0].text
            print(f"Partner 询问: {question}")

            # 提供更多信息
            result = await client.continue_task(
                task_id, session_id,
                user_input="我想要适合带孩子的景点"
            )

    finally:
        await client.close()

if __name__ == "__main__":
    asyncio.run(main())
```

#### 1.5.3 群组模式 Partner 示例

```python
import asyncio
from fastapi import FastAPI
from acps_sdk.aip import (
    GroupPartnerMqClient, TaskCommand, TaskResult, TaskStatus,
    TaskState, TaskCommandType, TextDataItem, Product
)
from acps_sdk.aip.aip_group_model import RabbitMQRequest, RabbitMQResponse
from datetime import datetime, timezone
import uuid

app = FastAPI()

# Partner 配置
PARTNER_AIC = "partner-urban-aic"

# 群组客户端存储 (group_id -> client)
group_clients: Dict[str, GroupPartnerMqClient] = {}

# 任务存储
tasks: Dict[str, TaskResult] = {}


async def process_task(command: TaskCommand, group_id: str) -> TaskResult:
    """处理任务并生成结果"""
    user_input = command.dataItems[0].text if command.dataItems else ""

    # 模拟处理：生成产出物
    product = Product(
        id=f"product-{uuid.uuid4()}",
        name="景点推荐",
        dataItems=[TextDataItem(text=f"为您推荐故宫周边景点：景山公园、北海公园、天安门广场")]
    )

    result = TaskResult(
        id=f"msg-{uuid.uuid4()}",
        sentAt=datetime.now(timezone.utc).isoformat(),
        senderRole="partner",
        senderId=PARTNER_AIC,
        taskId=command.taskId,
        sessionId=command.sessionId,
        groupId=group_id,
        status=TaskStatus(
            state=TaskState.AwaitingCompletion,
            stateChangedAt=datetime.now(timezone.utc).isoformat(),
        ),
        products=[product],
    )

    tasks[command.taskId] = result
    return result


async def on_task_command(command: TaskCommand, is_mentioned: bool) -> None:
    """处理来自群组的任务命令"""
    if not is_mentioned:
        return  # 未被提及，忽略

    group_id = command.groupId
    client = group_clients.get(group_id)
    if not client:
        return

    if command.command == TaskCommandType.Start:
        # 处理 Start 命令
        result = await process_task(command, group_id)

        # 广播任务结果到群组
        await client.send_task_result(
            task_id=result.taskId,
            session_id=result.sessionId,
            state=result.status.state,
            products=result.products,
        )
        print(f"已发送任务结果: {result.taskId} -> {result.status.state}")

    elif command.command == TaskCommandType.Complete:
        # Leader 确认完成
        task = tasks.get(command.taskId)
        if task:
            task.status = TaskStatus(
                state=TaskState.Completed,
                stateChangedAt=datetime.now(timezone.utc).isoformat(),
            )
            await client.send_task_result(
                task_id=task.taskId,
                session_id=task.sessionId,
                state=TaskState.Completed,
            )
            print(f"任务已完成: {task.taskId}")


async def on_task_result(result: TaskResult) -> None:
    """处理其他 Partner 的任务结果（群组模式下可见）"""
    if result.senderId != PARTNER_AIC:
        print(f"收到其他 Partner 状态: {result.senderId} -> {result.status.state}")


@app.post("/group/rpc", response_model=RabbitMQResponse)
async def handle_group_invite(request: RabbitMQRequest) -> RabbitMQResponse:
    """处理 Leader 的群组邀请"""
    if request.method != "group":
        return RabbitMQResponse(
            id=request.id,
            error={"code": -32601, "message": "Method not found"}
        )

    group_info = request.params.group
    group_id = group_info.groupId

    # 创建群组客户端
    client = GroupPartnerMqClient(
        partner_aic=PARTNER_AIC,
        rabbitmq_host=group_info.rabbitmq.host,
        rabbitmq_port=group_info.rabbitmq.port,
        rabbitmq_user=group_info.rabbitmq.user,
        rabbitmq_password=group_info.rabbitmq.password,
    )

    # 设置消息处理器
    client.set_command_handler(on_task_command)
    client.set_task_result_handler(on_task_result)

    # 加入群组
    try:
        await client.connect()
        await client.join_group(group_id)
        group_clients[group_id] = client

        # 开始消费消息（后台运行）
        asyncio.create_task(client.start_consuming())

        print(f"已加入群组: {group_id}")

        return RabbitMQResponse(
            id=request.id,
            result={"status": "joined", "groupId": group_id}
        )

    except Exception as e:
        return RabbitMQResponse(
            id=request.id,
            error={"code": -32603, "message": str(e)}
        )


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8011)
```

#### 1.5.4 群组模式 Leader 示例

```python
import asyncio
from acps_sdk.aip import (
    GroupLeaderMqClient, GroupPartnerMqClient,
    ACSObject, TaskCommandType, TaskState
)

async def leader_example():
    """Leader 端群组模式示例"""

    # 创建 Leader 客户端
    leader = GroupLeaderMqClient(
        leader_aic="leader-example-aic",
        rabbitmq_host="localhost",
        rabbitmq_port=5672,
        rabbitmq_user="guest",
        rabbitmq_password="guest",
    )

    # 连接并创建群组
    await leader.connect()
    group_id = await leader.create_group()
    print(f"群组已创建: {group_id}")

    # 邀请 Partner 加入
    partner_acs = ACSObject(aic="partner-urban-aic")
    partner_info = await leader.invite_partner(
        partner_acs=partner_acs,
        partner_rpc_url="http://localhost:8011/group/rpc"
    )
    print(f"Partner 已加入: {partner_info.aic}")

    # 设置消息处理器
    async def handle_message(message):
        if hasattr(message, 'status'):
            print(f"收到状态更新: {message.senderId} -> {message.status.state}")

            # 如果 Partner 等待确认，自动发送 Complete
            if message.status.state == TaskState.AwaitingCompletion:
                await leader.complete_task(
                    task_id=message.taskId,
                    session_id=message.sessionId,
                )
                print(f"已发送 Complete 命令: {message.taskId}")

    leader.set_message_handler(handle_message)
    await leader.start_consuming()

    # 发送任务
    task_id = await leader.start_task(
        session_id="session-001",
        text_content="请推荐故宫周边的景点",
        mentions=["partner-urban-aic"]  # 指定 Partner
    )
    print(f"任务已发送: {task_id}")

    # 等待响应（实际应用中会有更复杂的逻辑）
    await asyncio.sleep(5)

    # 清理
    await leader.close()

if __name__ == "__main__":
    asyncio.run(leader_example())
```

---

### 附录

#### A. 错误码参考

| 错误码 | 名称                   | 说明                 |
| ------ | ---------------------- | -------------------- |
| -32700 | Parse error            | JSON 格式不正确      |
| -32600 | Invalid Request        | 无效的 JSON-RPC 请求 |
| -32601 | Method not found       | 方法不存在           |
| -32602 | Invalid params         | 参数无效             |
| -32603 | Internal error         | 内部错误             |
| -32001 | TaskNotFoundError      | 任务不存在           |
| -32002 | TaskNotCancelableError | 任务无法取消         |
| -32007 | GroupNotSupportedError | 不支持群组功能       |

#### B. 相关文件参考

| 文件                                                                                  | 说明                |
| ------------------------------------------------------------------------------------- | ------------------- |
| [acps_sdk/aip/__init__.py](../acps_sdk/aip/__init__.py)                               | SDK 公共导出        |
| [acps_sdk/aip/aip_base_model.py](../acps_sdk/aip/aip_base_model.py)                   | 基础数据模型        |
| [acps_sdk/aip/aip_rpc_client.py](../acps_sdk/aip/aip_rpc_client.py)                   | RPC 客户端          |
| [acps_sdk/aip/aip_group_partner.py](../acps_sdk/aip/aip_group_partner.py)             | 群组 Partner 客户端 |
| [acps_sdk/aip/aip_group_leader.py](../acps_sdk/aip/aip_group_leader.py)               | 群组 Leader 客户端  |
| [partners/generic_runner.py](../partners/generic_runner.py)                           | Partner 通用运行器  |
| [partners/group_handler.py](../partners/group_handler.py)                             | Partner 群组处理器  |
| [leader/assistant/core/executor.py](../leader/assistant/core/executor.py)             | Leader 任务执行器   |
| [leader/assistant/core/group_executor.py](../leader/assistant/core/group_executor.py) | Leader 群组执行器   |

#### C. 协议规范参考

- [ACPs-spec-AIP.md](../../acps-specs/07-ACPs-spec-AIP/ACPs-spec-AIP.md) - AIP 协议完整规范

---

## 2. 智能体可信注册

要想让您的智能体可以被其他人发现和使用，您需要先前往智能体注册服务商注册您的智能体。您可以前往任意智能体能力注册服务商注册您的智能体。具体的流程如下：

### 2.1 安装挑战服务器与ca-client
打开[ioa.pub](https://ioa.pub/registry-web/)的界面，首次登陆需要进行账号注册，当注册完成并登陆后，按照如下步骤进行操作：

#### (1) 点击图中的链接分别下载ca-client和ca-chanllege-server

点击初始界面右上角的 `CA证书申请工具下载-ca-client` 下载ca-client
![1](./tutorials/pic/1.png)

点击初始界面左上角 `注册` 进入注册新智能体的界面，然后点击 `注册之前：关于 challenge-ca-challenge-server` 下载挑战服务器
![2](./tutorials/pic/2.png)

#### (2) 下载好安装包后，运行以下命令进行安装

```bash
#创建虚拟环境
python3 -m venv .venv
source .venv/bin/activate

#安装第(1)步下载的安装包
pip install acps_ca_challenge-2.0.0-py3-none-any.whl
pip install acps_ca_client-2.0.0-py3-none-any.whl
```

#### (3) 创建挑战服务器配置文件

```bash
vim .env

#配置示例
UVICORN_HOST=0.0.0.0
UVICORN_PORT=8004
UVICORN_RELOAD=false
BASE_URL=/acps-atr-v1
CHALLENGE_DIR=./challenges
LOG_LEVEL=INFO

```

### 2.2 启动挑战服务器
安装完成后，系统会自动注册 `challenge-server` 命令。你可以直接运行：

```bash
# 启动服务 (默认加载当前目录下的 .env)
challenge-server

# 如果使用自定义配置文件名
ENV_FILE=.env.prod challenge-server
```
运行成功后如下图所示
![3](./tutorials/pic/3.png)


### 2.3 编写能力描述

您需要参考 [智能体能力描述](../03-ACPs-spec-ACS/ACPs-spec-ACS.md)，为您的智能体撰写 ACS，将第一步启动的挑战服务器的地址填进 ACS 的`x-caChallengeBaseUrl`字段中。这里为您提供一个 北京城区旅游规划助手 的 ACS 示例：

```json
{
  "aic": "",
  "active": true,
  "lastModifiedTime": "2025-10-11T14:21:05.906801+08:00",
  "protocolVersion": "01.00",
  "name": "北京美食推荐智能体",
  "description": "北京美食推荐智能体（Partner）。职责：根据用户口味偏好、行程和（可选）交通锚点推荐北京全境（城区+郊区）餐饮与特色美食，可补充文化背景。范围：仅限北京餐饮。能力延伸：当用户提供交通/站点/地标信息，可基于该位置信息做就近餐饮匹配，但不提供交通路线/时刻表/出行建议。明确拒绝：① 景点/行程/交通规划请求；② 城际或外地餐饮请求；③ 纯交通咨询；④ 与餐饮无关的通用问答。若请求混合含景点或交通规划需求，仅提取可分离的餐饮部分并声明拒绝其余。",
  "version": "1.0.0",
  "provider": {
    "organization": "北京邮电大学",
    "department": "人工智能学院",
    "url": "https://ai.bupt.edu.cn",
    "license": "京ICP备14033833号-1"
  },
  "securitySchemes": {
    "mtls": {
      "type": "mutualTLS",
      "description": "智能体间mTLS双向认证",
      "x-caChallengeBaseUrl": "http://localhost:8004/acps-atr-v2"
    }
  },
  "endPoints": [
    {
      "url": "http://localhost:8011/partners/beijing_food/rpc",
      "transport": "HTTP",
      "security": [
        {
          "mtls": []
        }
      ]
    }
  ],
  "capabilities": {
    "streaming": false,
    "notification": false,
    "messageQueue": []
  },
  "defaultInputModes": ["text/plain"],
  "defaultOutputModes": ["text/plain", "application/json"],
  "skills": [
    {
      "id": "beijing_catering.traditional-food-recommendation",
      "name": "传统美食推荐",
      "description": "推荐北京传统美食和老字号餐厅，包括烤鸭、炸酱面、豆汁、爆肚等经典北京菜品。拒绝与北京无关的餐饮请求。",
      "version": "1.0.0",
      "tags": ["传统美食", "老字号", "北京烤鸭", "经典菜品"],
      "examples": [
        "我想在北京品尝最正宗的烤鸭，请推荐几家历史悠久的老字号餐厅，最好能介绍一下它们的特色和价格区间",
        "除了烤鸭，北京还有哪些必吃的传统小吃？请推荐一些能品尝到炸酱面、豆汁焦圈、爆肚的正宗店铺",
        "我对北京的老字号餐厅很感兴趣，请推荐几家有百年历史的餐厅，并介绍它们的招牌菜和用餐环境"
      ],
      "inputModes": ["text/plain"],
      "outputModes": ["text/plain", "application/json"]
    },
    {
      "id": "beijing_catering.location-based-restaurant-recommendation",
      "name": "位置匹配餐厅推荐",
      "description": "根据用户当前位置或旅游路线推荐附近的餐厅，优化用餐时间和路线安排。拒绝非北京范围位置请求。",
      "version": "1.0.0",
      "tags": ["位置匹配", "路线优化", "就近用餐", "时间安排"],
      "examples": [
        "我明天上午要去故宫游览，中午想在附近找一家不错的餐厅吃午饭，步行距离不超过500米，有什么推荐吗？",
        "今晚想去簋街体验北京的夜市文化，请推荐几家簋街上口碑好的特色餐厅，最好有小龙虾或烧烤",
        "我住在王府井附近，想找一家适合商务宴请的高档餐厅，环境要安静，有包间最好，请推荐几个选择"
      ],
      "inputModes": ["text/plain"],
      "outputModes": ["text/plain", "application/json"]
    },
    {
      "id": "beijing_catering.dietary-preference-matching",
      "name": "口味偏好匹配",
      "description": "根据用户的饮食偏好、忌口要求和口味特点推荐合适的美食和餐厅。支持素食、低辣、儿童友好需求。",
      "version": "1.0.0",
      "tags": ["口味偏好", "饮食禁忌", "个性化推荐", "特殊需求"],
      "examples": [
        "我不能吃辣也不能吃海鲜，有轻微的乳糖不耐受，请推荐一些适合我的北京特色美食和餐厅",
        "我是素食主义者，想在北京找几家做得不错的素食餐厅，最好有传统的素食版本北京菜",
        "我带着5岁的孩子来北京旅游，请推荐一些儿童友好的餐厅，菜品要清淡少盐，环境要适合亲子用餐"
      ],
      "inputModes": ["text/plain"],
      "outputModes": ["text/plain", "application/json"]
    },
    {
      "id": "beijing_catering.food-culture-experience",
      "name": "美食文化体验",
      "description": "介绍北京美食的历史文化背景，提供深度的文化体验和知识分享。拒绝与北京饮食文化无关的泛化问题。",
      "version": "1.0.0",
      "tags": ["文化体验", "美食历史", "文化背景", "知识分享"],
      "examples": [
        "请详细介绍北京烤鸭的历史起源、制作工艺和文化意义，以及全聚德和便宜坊两家老店的区别",
        "我想了解老北京胡同里的传统小吃文化，比如豆汁焦圈、卤煮火烧背后的历史故事和制作传统",
        "京菜是如何形成的？它与宫廷菜、民间菜的关系是什么？请介绍几道代表性的京菜及其文化背景"
      ],
      "inputModes": ["text/plain"],
      "outputModes": ["text/plain", "application/json"]
    }
  ]
}
```

### 2.4 注册智能体本体
然后您需要在下图的注册页面中填写 ACS ，然后智能体能力注册服务商会对您的 ACS 进行审核，审核通过后，您的智能体会得到智能体身份码 AIC。

同时您可以选择您的智能体是否可派生(协议细节请参考[ATR协议](../../04-ACPs-spec-ATR/ACPs-spec-ATR.md)),当您只想注册一个智能体本体时，请选择 `否`；当您想注册一个可以派生多个智能体实体的智能体本体时，请选择 `是`。
![4](./tutorials/pic/4.png)

提交完成后即可进入审核流程，当审核通过后您的智能体将获得其独一无二的 AIC。


### 2.5 获取证书

#### (1) 创建ca-client.conf

```bash
vim ca-client.conf

# ca-client配置示例，按需修改
CA_SERVER_BASE_URL = http://bupt.ioa.pub:8003/acps-atr-v2
CHALLENGE_SERVER_BASE_URL = http://10.106.130.104:8004/acps-atr-v1
ACCOUNT_KEY_PATH = ./private/account.key
CERTS_DIR = ./certs
PRIVATE_KEYS_DIR = ./private
CSR_DIR = ./csr
TRUST_BUNDLE_PATH = ./certs/trust-bundle.pem
```

#### (2) 根据 ATR 申请 CA 证书
```bash
# 默认使用运行目录下的ca-client配置文件
ca-client new-cert --aic 1.2.156.3088.xxxx.xxxx.xxxxx.xxxxx.1.xxx
```

#### (3) 密钥轮换，根据 ATR 设计轮换 ACME 账户密钥 (可选)
```bash
ca-client key-rollover --new-key ./private/account-new.key
```
这样您就成功为您的智能体申请到一个证书。

### 2.6 注册智能体实体
智能体实体的注册可自动审核完成，可通过注册服务器的API进行实体注册，API文档通常在 http://your-registry-server-host-port/docs 。如下图所示：

接口说明
![5](./tutorials/pic/5.png)

请求体
![6](./tutorials/pic/6.png)

响应体
![7](./tutorials/pic/7.png)

当注册成功后，响应体中会返回实体的AIC，然后我们可以用实体的 AIC 重新进行第2.5步获取证书，为实体申请证书。

---

## 3. 体验基于ACPs的智能体互联应用（demo）

### 3.1 ACPs协议细节
所有ACPs协议文档均可通过 [协议文档汇总](../README.md) 访问。

### 3.2 demo使用

为了您快速体验ACPs协议，我们提供了一个demo，其代码仓库位于 [demo-app]()。这里我们提供了能快速体验demo的流程。

**首先克隆demo代码仓库**

```bash
git clone [demo-url]
cd demo-apps
```
**创建虚拟环境并安装依赖**

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

**配置RabbitMQ（可选）**
如果想体验AIP的群组模式，还需要配置RabbitMQ服务，可参考[此文档](gettingstarted/配置MQ.md)

**准备配置文件：**

```bash
# 准备leader的配置文件
cp leader/config.example.toml leader/config.toml
# 准备partner的配置文件(在这里直接创建所有partner的配置文件)
find partners/online -name "config.example.toml" -type f | while read file; do cp "$file" "${file%.example.toml}.toml"; done
# 重要：需要在创建的config.toml中根据实际情况对LLM Profile, Server Port, Discovery URL等进行配置
```

**启动全部 Agent 与演示服务（首次运行可先确认证书路径等）：**

```bash
./start.sh
```

**在浏览器验证 Leader：**

用浏览器访问 `http://localhost:3000`。您可以看到如下界面，您可以体验其中的所有demo演示。

![](../gettingstarted/pic/demo.png)


**查看运行日志：**

- 每个子服务在 `logs/` 下记录日志。
- 若需查看所有服务的日志，可运行 `tail -f logs/*.log`。


