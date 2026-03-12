# CAMEL 规划-执行分离设计模式深度解析

本文档详细分析 CAMEL 框架中"规划智能体生成多步骤规划，执行智能体负责执行"的设计模式，为实验设计提供参考。

---

## 一、总体架构概述

CAMEL 框架实现了多种规划-执行分离模式，主要分为两大类：

1. **Workforce 架构**：静态/动态多智能体协作，先完整规划后执行
2. **BabyAGI 架构**：交替进行规划与执行，动态生成任务

---

## 二、Workforce 架构详解

### 2.1 核心设计思想

Workforce 采用**三层分离架构**：

| 层级 | 智能体类型 | 职责 |
|------|-----------|------|
| 规划层 | Task Planner Agent | 任务分解、质量评估、失败分析 |
| 协调层 | Coordinator Agent | 任务分配、Worker选择 |
| 执行层 | Worker (SingleAgentWorker/RolePlayingWorker) | 具体任务执行 |

### 2.2 核心文件路径

- **主实现**: `camel/societies/workforce/workforce.py`
- **Worker 基类**: `camel/societies/workforce/worker.py`
- **单智能体Worker**: `camel/societies/workforce/single_agent_worker.py`
- **角色扮演Worker**: `camel/societies/workforce/role_playing_worker.py`
- **任务通道**: `camel/societies/workforce/task_channel.py`
- **提示词模板**: `camel/societies/workforce/prompts.py`
- **工具类**: `camel/societies/workforce/utils.py`
- **Task 定义**: `camel/tasks/task.py`

### 2.3 规划智能体 (Task Planner Agent)

#### 2.3.1 初始化与配置

```python
# workforce.py 第471-530行
if task_agent is None:
    self.task_agent = ChatAgent(
        system_message=BaseMessage.make_assistant_message(
            role_name="task_agent",
            content=TASK_AGENT_SYSTEM_MESSAGE,
        ),
        model=self.default_model,
    )
```

**系统消息定义** (`prompts.py` 第449-457行)：

```python
TASK_AGENT_SYSTEM_MESSAGE = """You are an intelligent task management assistant responsible for planning, analyzing, and quality control.

Your responsibilities include:
1. **Task Decomposition**: Breaking down complex tasks into manageable subtasks
2. **Failure Analysis**: Analyzing task failures to determine root cause
3. **Quality Evaluation**: Assessing completed task results
"""
```

#### 2.3.2 任务分解核心流程

**入口方法**: `handle_decompose_append_task()` (`workforce.py` 第2510-2591行)

```python
async def handle_decompose_append_task(self, task: Task, reset: bool = True) -> List[Task]:
    # 1. 验证任务
    if not validate_task_content(task.content, task.id):
        task.state = TaskState.FAILED
        return [task]

    # 2. 调用任务分解
    subtasks_result = self._decompose_task(task, self._on_stream_callback)

    # 3. 处理流式/非流式结果
    if isinstance(subtasks_result, Generator):
        subtasks = []
        for new_tasks in subtasks_result:
            subtasks.extend(new_tasks)
    else:
        subtasks = subtasks_result

    # 4. 添加到待处理队列
    if subtasks:
        self._pending_tasks.extendleft(reversed(subtasks))
    else:
        self._pending_tasks.append(task)

    return subtasks
```

**分解方法**: `_decompose_task()` (`workforce.py` 第1474-1532行)

```python
def _decompose_task(self, task: Task, stream_callback=None):
    # PIPELINE 模式下不分解
    if self.mode == WorkforceMode.PIPELINE:
        return []

    decompose_prompt = str(TASK_DECOMPOSE_PROMPT.format(
        content=task.content,
        child_nodes_info=self._get_child_nodes_info(),
        additional_info=task.additional_info,
    ))

    self.task_agent.reset()
    result = task.decompose(self.task_agent, decompose_prompt, stream_callback=stream_callback)

    # 处理依赖关系
    if isinstance(result, Generator):
        # 流式处理...
    else:
        subtasks = result
        if subtasks:
            self._update_dependencies_for_decomposition(task, subtasks)
        return subtasks
```

### 2.4 任务分解提示词设计

**文件**: `camel/societies/workforce/prompts.py` 第194-309行

```python
TASK_DECOMPOSE_PROMPT = r"""You need to either decompose a complex task or enhance a simple one.

0. **Analyze Task Complexity**:
    * **If the task is complex, decompose it.**
    * **If the task is simple, do not decompose it.** Instead, rewrite and enhance it.

1. **Self-Contained Subtasks**: Each subtask must be fully self-sufficient.

2. **Define Clear Deliverables**: Each task must specify a clear, concrete deliverable.

4. **Aggressive Parallelization**: Decompose into parallel subtasks when possible.

6. **Skill-Aware Decomposition**: If a task explicitly requires using a skill, DO NOT decompose it.

Output format:
<tasks>
<task>Subtask 1</task>
<task>Subtask 2</task>
</tasks>
"""
```

### 2.5 Task 类的分解实现

**文件**: `camel/tasks/task.py` 第408-458行

```python
def decompose(
    self,
    agent: "ChatAgent",
    prompt: Optional[str] = None,
    task_parser: Callable[[str, str], List["Task"]] = parse_response,
    stream_callback: Optional[Callable[["ChatAgentResponse"], None]] = None,
) -> Union[List["Task"], Generator[List["Task"], None, None]]:
    """Decompose a task to a list of sub-tasks."""
    role_name = agent.role_name
    content = prompt or TASK_DECOMPOSE_PROMPT.format(
        role_name=role_name,
        content=self.content,
    )
    msg = BaseMessage.make_user_message(role_name=role_name, content=content)
    response = agent.step(msg)

    # 自动检测流式响应
    from camel.utils import is_streaming_response
    is_streaming = is_streaming_response(response)

    if is_streaming:
        return self._decompose_streaming(response, task_parser, stream_callback=stream_callback)
    return self._decompose_non_streaming(response, task_parser)
```

### 2.6 依赖关系管理

**方法**: `_update_dependencies_for_decomposition()` (`workforce.py` 第1391-1427行)

```python
def _update_dependencies_for_decomposition(
    self, original_task: Task, subtasks: List[Task]
) -> None:
    """Update dependency tracking when a task is decomposed."""
    if not subtasks:
        return

    original_task_id = original_task.id
    subtask_ids = [subtask.id for subtask in subtasks]

    # 查找依赖于原始任务的任务
    dependent_task_ids = [
        task_id for task_id, deps in self._task_dependencies.items()
        if original_task_id in deps
    ]

    # 更新依赖任务使其依赖所有子任务
    for task_id in dependent_task_ids:
        dependencies = self._task_dependencies[task_id]
        dependencies.remove(original_task_id)
        dependencies.extend(subtask_ids)

    # 最后一个子任务继承原始任务的依赖
    if original_task_id in self._task_dependencies:
        original_dependencies = self._task_dependencies[original_task_id]
        if original_dependencies:
            self._task_dependencies[subtask_ids[-1]] = original_dependencies.copy()
        del self._task_dependencies[original_task_id]
```

### 2.7 协调智能体 (Coordinator Agent)

#### 2.7.1 任务分配流程

**入口**: `_call_coordinator_for_assignment()` (`workforce.py` 第3693-3806行)

```python
def _call_coordinator_for_assignment(
    self, tasks: List[Task], invalid_ids: Optional[List[str]] = None
) -> TaskAssignResult:
    # 格式化任务信息
    tasks_info = ""
    for task in tasks:
        tasks_info += f"Task ID: {task.id}\n"
        tasks_info += f"Content: {task.content}\n"
        if task.additional_info:
            tasks_info += f"Additional Info: {task.additional_info}\n"
        tasks_info += "---\n"

    prompt = str(ASSIGN_TASK_PROMPT.format(
        tasks_info=tasks_info,
        child_nodes_info=self._get_child_nodes_info(),
    ))

    # 使用结构化输出
    response = self.coordinator_agent.step(prompt, response_format=TaskAssignResult)
    result_dict = json.loads(response_content, parse_int=str)
    return TaskAssignResult(**result_dict)
```

#### 2.7.2 任务分配提示词

**文件**: `prompts.py` 第52-85行

```python
ASSIGN_TASK_PROMPT = TextPrompt("""You need to assign multiple tasks to worker nodes.

For each task, you need to:
1. Choose the most capable worker node ID for that task.
2. Identify any dependencies between tasks.

Your response MUST be a valid JSON object containing an 'assignments' field.

Each assignment dictionary should have:
- "task_id": the ID of the task
- "assignee_id": the ID of the chosen worker node
- "dependencies": list of task IDs that this task depends on

Example:
{"assignments": [
    {"task_id": "task_1", "assignee_id": "node_12345", "dependencies": []},
    {"task_id": "task_2", "assignee_id": "node_67890", "dependencies": ["task_1"]}
]}
""")
```

#### 2.7.3 任务分配数据结构

**文件**: `utils.py` 第169-209行

```python
class TaskAssignment(BaseModel):
    task_id: str = Field(description="The ID of the task to be assigned.")
    assignee_id: str = Field(description="The ID of the worker to assign the task to.")
    dependencies: List[str] = Field(
        default_factory=list,
        description="List of task IDs that must complete before this task."
    )

class TaskAssignResult(BaseModel):
    assignments: List[TaskAssignment] = Field(description="List of task assignments.")
```

### 2.8 执行智能体 (Worker)

#### 2.8.1 Worker 抽象基类

**文件**: `worker.py` 第34-77行

```python
class Worker(BaseNode, ABC):
    """A worker node that works on tasks."""

    @abstractmethod
    async def _process_task(
        self,
        task: Task,
        dependencies: List[Task],
        stream_callback: Optional[Callable[["ChatAgentResponse"], Optional[Awaitable[None]]]] = None,
    ) -> TaskState:
        """Processes a task based on its dependencies.

        Returns:
            'DONE' if the task is successfully processed,
            'FAILED' if the processing fails.
        """
        pass
```

#### 2.8.2 SingleAgentWorker 实现

**文件**: `single_agent_worker.py` 第349-584行

```python
async def _process_task(
    self,
    task: Task,
    dependencies: List[Task],
    stream_callback: Optional[Callable[["ChatAgentResponse"], Optional[Awaitable[None]]]] = None,
) -> TaskState:
    # 从代理池获取智能体
    worker_agent = await self._get_worker_agent()

    try:
        # 构建依赖信息
        dependency_tasks_info = self._get_dep_tasks_info(dependencies)
        prompt = str(PROCESS_TASK_PROMPT.format(
            content=task.content,
            parent_task_content=task.parent.content if task.parent else "",
            dependency_tasks_info=dependency_tasks_info,
            additional_info=task.additional_info,
        ))

        # 执行任务
        response = await worker_agent.astep(prompt, response_format=TaskResult)
        task_result = response.msg.parsed

        # 设置结果
        task.result = task_result.content

        if task_result.failed:
            return TaskState.FAILED
        return TaskState.DONE

    finally:
        # 归还代理到池中
        await self._return_worker_agent(worker_agent)
```

### 2.9 Workforce 完整执行流程

```
process_task_async(task)
    │
    ▼
handle_decompose_append_task(task)  [规划阶段: Task Planner Agent分解任务]
    │
    ▼
_decompose_task(task)
    │
    ├──> task.decompose(task_agent)  [调用Task类的分解方法]
    │
    ▼
[Subtasks added to _pending_tasks]
    │
    ▼
start() -> _listen_to_channel()  [进入执行阶段]
    │
    ▼
_post_ready_tasks()  [检查依赖并发布就绪任务]
    │
    ▼
_find_assignee() -> _call_coordinator_for_assignment()  [协调阶段: Coordinator Agent分配任务]
    │
    ▼
_post_task() -> TaskChannel  [发布到任务通道]
    │
    ▼
Worker._listen_to_channel()  [Worker监听通道]
    │
    ▼
Worker._process_task()  [执行阶段: Worker执行任务]
    │
    ▼
return_task()  [返回结果到TaskChannel]
```

### 2.10 执行模式对比

```python
class WorkforceMode(Enum):
    AUTO_DECOMPOSE = "auto_decompose"  # 自动任务分解模式
    PIPELINE = "pipeline"  # 预定义管道模式
```

| 模式 | 规划方式 | 执行方式 | 适用场景 |
|------|----------|----------|----------|
| AUTO_DECOMPOSE | 动态分解 | 智能恢复策略 | 复杂、不确定的任务 |
| PIPELINE | 预定义 | Fork-Join并行 | 固定流程的任务 |

---

## 三、BabyAGI 架构详解

### 3.1 核心设计思想

BabyAGI 采用**交替式规划-执行架构**：

| 组件 | 类型 | 职责 |
|------|------|------|
| assistant_agent | ChatAgent | **执行** - 实际完成任务 |
| task_creation_agent | TaskCreationAgent | **规划** - 基于上下文创建新任务 |
| task_prioritization_agent | TaskPrioritizationAgent | **规划** - 对任务列表排序 |

### 3.2 核心文件路径

- **主实现**: `camel/societies/babyagi_playing.py`
- **Task Agent**: `camel/agents/task_agent.py`
- **示例**: `examples/ai_society/babyagi_playing.py`

### 3.3 核心组件初始化

**文件**: `babyagi_playing.py` 第111-126行

```python
self.assistant_agent: ChatAgent           # 执行代理
self.assistant_sys_msg: Optional[BaseMessage]
self.task_creation_agent: TaskCreationAgent      # 规划 - 任务创建
self.task_prioritization_agent: TaskPrioritizationAgent  # 规划 - 任务排序

self.subtasks: deque = deque([])          # 待执行的任务队列
self.solved_subtasks: List[str] = []      # 已完成的任务列表
self.MAX_TASK_HISTORY = max_task_history  # 最大历史任务数（默认10）
```

### 3.4 TaskCreationAgent 实现

**文件**: `task_agent.py` 第200-312行

```python
class TaskCreationAgent(ChatAgent):
    def __init__(
        self,
        role_name: str,
        objective: Union[str, TextPrompt],
        max_task_num: Optional[int] = 3,  # 每次最多创建3个任务
    ):
        task_creation_prompt = TextPrompt("""
Create new tasks with the following objective: {objective}.
You must consider past solved tasks and in-progress tasks: {task_list}.
The new created tasks must not overlap with these past tasks.

Output format (numbered list):
    #. First Task
    #. Second Task
    #. Third Task

You can only give up to {max_task_num} tasks at a time.
If no new tasks are needed, write "No tasks to add."
""")
```

**关键特性**:
- 接收 `task_list` 参数（已解决和进行中的任务）
- 确保新任务不与已有任务重叠
- 每次最多创建 `max_task_num` (默认3) 个任务
- 如果不需要新任务，返回 "No tasks to add."

### 3.5 TaskPrioritizationAgent 实现

**文件**: `task_agent.py` 第315-410行

```python
class TaskPrioritizationAgent(ChatAgent):
    def __init__(self, objective: Union[str, TextPrompt], ...):
        task_prioritization_prompt = TextPrompt("""
Prioritize the following tasks: {task_list}.
Consider the ultimate objective: {objective}.

Tasks should be sorted from highest to lowest priority.
Higher-priority tasks are prerequisites or more essential.

Return one task per line:
    #. First task
    #. Second task

Do not remove or modify any tasks, only re-sort them.
""")
```

### 3.6 BabyAGI 主执行流程

**文件**: `babyagi_playing.py` 第224-284行

```python
def step(self) -> ChatAgentResponse:
    # 1. 如果任务队列为空，初始化创建任务
    if not self.subtasks:
        new_subtask_list = self.task_creation_agent.run(task_list=[])
        prioritized_subtask_list = self.task_prioritization_agent.run(new_subtask_list)
        self.subtasks = deque(prioritized_subtask_list)

    # 2. 取出最高优先级任务
    task_name = self.subtasks.popleft()
    assistant_msg_msg = BaseMessage.make_user_message(
        role_name="assistant",
        content=f"{task_name}",
    )

    # 3. 执行该任务
    assistant_response = self.assistant_agent.step(assistant_msg_msg)
    assistant_msg = assistant_response.msgs[0]

    # 4. 记录已解决的任务
    self.solved_subtasks.append(task_name)
    past_tasks = self.solved_subtasks + list(self.subtasks)

    # 5. 基于执行结果创建新任务（规划阶段）
    new_subtask_list = self.task_creation_agent.run(
        task_list=past_tasks[-self.MAX_TASK_HISTORY:]
    )

    # 6. 如果有新任务，添加到队列并重新排序
    if new_subtask_list:
        self.subtasks.extend(new_subtask_list)
        prioritized_subtask_list = self.task_prioritization_agent.run(
            task_list=list(self.subtasks)[-self.MAX_TASK_HISTORY:]
        )
        self.subtasks = deque(prioritized_subtask_list)

    # 7. 检查是否所有任务完成
    if not self.subtasks:
        terminated = True
        return ChatAgentResponse(...)

    return ChatAgentResponse(...)
```

### 3.7 BabyAGI 执行流程图

```
┌─────────────────────────────────────────────────────────────┐
│                         step() 方法                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  subtasks为空?   │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              ▼                              ▼
        ┌─────────────┐                ┌─────────────┐
        │  是 - 初始化 │                │  否 - 继续   │
        │  创建初始任务 │                │  执行现有任务 │
        └──────┬──────┘                └──────┬──────┘
               │                              │
               ▼                              ▼
    ┌─────────────────────┐          ┌─────────────────────┐
    │ TaskCreationAgent   │          │ task_name =         │
    │ .run(task_list=[])  │          │ subtasks.popleft()  │
    └──────────┬──────────┘          └──────────┬──────────┘
               │                                │
               ▼                                ▼
    ┌─────────────────────┐          ┌─────────────────────┐
    │ TaskPrioritization  │          │ assistant_agent     │
    │ Agent.run()         │          │ .step(assistant_msg)│
    └──────────┬──────────┘          └──────────┬──────────┘
               │                                │
               ▼                                ▼
    ┌─────────────────────┐          ┌─────────────────────┐
    │ subtasks = deque(   │          │ solved_subtasks     │
    │   prioritized_list) │          │ .append(task_name)  │
    └─────────────────────┘          └──────────┬──────────┘
                                                │
                                                ▼
                                     ┌─────────────────────┐
                                     │ TaskCreationAgent   │
                                     │ .run(task_list=     │
                                     │   past_tasks)       │
                                     └──────────┬──────────┘
                                                │
                                                ▼
                                     ┌─────────────────────┐
                                     │ 有新任务?            │
                                     └──────────┬──────────┘
                                                │
                                   ┌────────────┴────────────┐
                                   ▼                         ▼
                            ┌─────────────┐            ┌─────────────┐
                            │  是 - 添加   │            │  否 - 记录   │
                            │  并重新排序  │            │  "no new    │
                            │             │            │  tasks"     │
                            └──────┬──────┘            └──────┬──────┘
                                   │                          │
                                   ▼                          ▼
                        ┌─────────────────────┐      ┌─────────────────┐
                        │ TaskPrioritization  │      │ 检查subtasks    │
                        │ Agent.run()         │      │ 是否为空?        │
                        └──────────┬──────────┘      └────────┬────────┘
                                   │                          │
                                   ▼                          ▼
                        ┌─────────────────────┐      ┌─────────────────┐
                        │ subtasks = deque(   │      │  是 → 终止      │
                        │   prioritized_list) │      │  否 → 继续循环  │
                        └─────────────────────┘      └─────────────────┘
```

### 3.8 BabyAGI 与 Workforce 对比

| 特性 | BabyAGI | Workforce |
|------|---------|-----------|
| 规划时机 | 执行后动态规划 | 执行前完整规划 |
| 任务数量 | 逐步生成 | 一次性分解 |
| 执行方式 | 串行 | 并行（支持依赖） |
| 规划粒度 | 单步规划 | 多步完整规划 |
| 适用场景 | 探索性任务 | 复杂协作任务 |

---

## 四、TaskPlanningToolkit 工具包

### 4.1 工具包设计

**文件**: `camel/toolkits/task_planning_toolkit.py`

```python
class TaskPlanningToolkit(BaseToolkit):
    def decompose_task(
        self,
        original_task_content: str,
        sub_task_contents: List[str],
        original_task_id: Optional[str] = None,
    ) -> List[Task]:
        """将原始任务分解为多个子任务"""

    def replan_tasks(
        self,
        original_task_content: str,
        sub_task_contents: List[str],
        original_task_id: Optional[str] = None,
    ) -> List[Task]:
        """当子任务不相关或不符合目标时重新规划任务"""
```

### 4.2 使用示例

**文件**: `examples/toolkits/task_planning_toolkit.py`

```python
# 创建任务规划工具包
tools = TaskPlanningToolkit().get_tools()

# 创建智能体并绑定工具
agent = ChatAgent(sys_prompt, model=model, tools=tools)

# 示例：任务分解
user_msg = BaseMessage.make_user_message(
    role_name="User",
    content="Decompose this task: Write a research paper on AI ethics"
)
response = agent.step(user_msg)

# 输出：将任务分解为10个子任务
# 1. Research the history of AI ethics
# 2. Identify key ethical issues in AI
# 3. Review existing literature on AI ethics
# ...
```

---

## 五、实验设计建议

### 5.1 规划-执行分离的实验设计方案

基于 CAMEL 的设计，推荐以下实验架构：

#### 方案一：Workforce 模式（完整规划后执行）

```python
from camel.societies.workforce import Workforce
from camel.agents import ChatAgent
from camel.tasks import Task

# 1. 创建规划智能体
planner_agent = ChatAgent(
    system_message=BaseMessage.make_assistant_message(
        role_name="planner",
        content="You are a task planner that breaks down complex tasks into subtasks."
    ),
    model=model,
)

# 2. 创建执行智能体
executor_agent = ChatAgent(
    system_message=BaseMessage.make_assistant_message(
        role_name="executor",
        content="You are a task executor that completes assigned tasks."
    ),
    model=model,
)

# 3. 创建 Workforce
workforce = Workforce(
    'Experiment Workforce',
    task_agent=planner_agent,  # 规划智能体
)
workforce.add_single_agent_worker('executor', executor_agent)  # 执行智能体

# 4. 执行任务（自动：规划 -> 分配 -> 执行）
task = Task(content="Your complex task here", id="0")
result = workforce.process_task(task)
```

#### 方案二：BabyAGI 模式（交替规划执行）

```python
from camel.societies.babyagi_playing import BabyAGI

# 创建 BabyAGI 实例
babyagi = BabyAGI(
    assistant_role_name="executor",
    user_role_name="user",
    task_prompt="Your task here",
    model=model,
)

# 执行（自动交替规划和执行）
for _ in range(chat_turn_limit):
    response = babyagi.step()
    if response.terminated:
        break
```

#### 方案三：自定义分离架构

```python
from camel.agents import ChatAgent
from camel.tasks import Task

class PlannerExecutorSystem:
    def __init__(self, planner_model, executor_model):
        self.planner = ChatAgent(
            system_message=self.get_planner_sysmsg(),
            model=planner_model,
        )
        self.executor = ChatAgent(
            system_message=self.get_executor_sysmsg(),
            model=executor_model,
        )

    def plan(self, task: str) -> List[str]:
        """规划阶段：生成多步骤规划"""
        response = self.planner.step(f"Create a plan for: {task}")
        # 解析响应为步骤列表
        return self.parse_plan(response.msgs[0].content)

    def execute(self, steps: List[str]) -> List[str]:
        """执行阶段：逐条执行规划"""
        results = []
        for step in steps:
            response = self.executor.step(f"Execute this step: {step}")
            results.append(response.msgs[0].content)
        return results

    def run(self, task: str):
        # 先完整规划
        plan = self.plan(task)
        # 再执行
        results = self.execute(plan)
        return plan, results
```

### 5.2 关键实验变量

| 变量 | 说明 | 可选值 |
|------|------|--------|
| 规划粒度 | 规划详细程度 | 粗粒度/细粒度 |
| 规划时机 | 何时进行规划 | 执行前/执行中/混合 |
| 反馈机制 | 执行结果是否反馈给规划 | 有/无 |
| 重规划策略 | 失败时如何处理 | 重试/重规划/放弃 |

### 5.3 评估指标

1. **任务成功率**: 完成任务的比例
2. **规划质量**: 规划的合理性、完整性
3. **执行效率**: 完成任务所需的步骤数/时间
4. **适应性**: 面对变化时的调整能力

---

## 六、关键代码引用汇总

| 功能 | 文件路径 | 关键类/方法 |
|------|----------|-------------|
| Workforce 主类 | `camel/societies/workforce/workforce.py` | `Workforce` |
| Worker 基类 | `camel/societies/workforce/worker.py` | `Worker` |
| 单智能体Worker | `camel/societies/workforce/single_agent_worker.py` | `SingleAgentWorker` |
| 任务通道 | `camel/societies/workforce/task_channel.py` | `TaskChannel` |
| 提示词模板 | `camel/societies/workforce/prompts.py` | `TASK_DECOMPOSE_PROMPT`, `ASSIGN_TASK_PROMPT` |
| 工具类 | `camel/societies/workforce/utils.py` | `TaskAssignment`, `TaskAssignResult` |
| Task 定义 | `camel/tasks/task.py` | `Task.decompose()` |
| BabyAGI | `camel/societies/babyagi_playing.py` | `BabyAGI` |
| Task Agent | `camel/agents/task_agent.py` | `TaskCreationAgent`, `TaskPrioritizationAgent` |
| 任务规划工具包 | `camel/toolkits/task_planning_toolkit.py` | `TaskPlanningToolkit` |

---

## 七、总结

CAMEL 框架提供了多种规划-执行分离的实现模式：

1. **Workforce 架构**适合需要完整规划、并行执行、复杂依赖管理的场景
2. **BabyAGI 架构**适合需要动态规划、逐步探索、适应性强的场景
3. **TaskPlanningToolkit** 提供了可组合的任务规划工具

实验设计时可根据任务特点选择合适的模式，或组合多种模式实现更复杂的智能体协作系统。
