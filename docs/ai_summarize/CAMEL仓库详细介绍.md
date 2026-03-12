# CAMEL 仓库详细介绍

## 一、仓库整体结构与文件作用

### 1.1 顶层目录结构

| 路径                                                         | 作用                                                    |
| ------------------------------------------------------------ | ------------------------------------------------------- |
| [README.md](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/README.md) | 项目主文档（英文），介绍CAMEL框架的定位、功能和快速开始 |
| [README.zh.md](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/README.zh.md) | 中文文档                                                |
| [pyproject.toml](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/pyproject.toml) | Python项目配置，定义依赖、版本（0.2.90a5）              |
| [camel/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/) | **核心源代码包** - 框架的主要实现                       |
| [examples/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/examples/) | **示例代码** - 各种使用场景示例                         |
| [docs/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/docs/) | **文档** - Sphinx文档源文件                             |
| [apps/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/apps/) | **应用程序** - 基于CAMEL构建的应用                      |
| [test/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/test/) | **测试代码**                                            |
| [data/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/data/) | 数据目录                                                |

### 1.2 核心模块详解（camel/目录）

#### 1.2.1 Agents 模块 ([camel/agents/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/agents/))

这是框架最核心的模块，包含各种智能体实现：

| 文件                                                         | 类/作用                                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [base.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/agents/base.py) | `BaseAgent` - 所有Agent的抽象基类                            |
| [chat_agent.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/agents/chat_agent.py) | `ChatAgent` - **核心对话智能体**（265KB，框架核心）          |
| [critic_agent.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/agents/critic_agent.py) | `CriticAgent` - 批评/评估智能体                              |
| [task_agent.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/agents/task_agent.py) | **任务规划相关智能体**：`TaskPlannerAgent`、`TaskCreationAgent`、`TaskPrioritizationAgent`、`TaskSpecifyAgent` |
| [embodied_agent.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/agents/embodied_agent.py) | `EmbodiedAgent` - 具身智能体                                 |
| [knowledge_graph_agent.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/agents/knowledge_graph_agent.py) | `KnowledgeGraphAgent` - 知识图谱智能体                       |
| [role_assignment_agent.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/agents/role_assignment_agent.py) | `RoleAssignmentAgent` - 角色分配智能体                       |
| [search_agent.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/agents/search_agent.py) | `SearchAgent` - 搜索智能体                                   |

#### 1.2.2 Models 模块 ([camel/models/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/models/))

支持**50+种LLM后端**的统一接口：

| 文件                                                         | 作用                          |
| ------------------------------------------------------------ | ----------------------------- |
| [base_model.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/models/base_model.py) | `BaseModelBackend` 模型基类   |
| [model_factory.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/models/model_factory.py) | `ModelFactory` - 模型工厂模式 |
| [openai_model.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/models/openai_model.py) | OpenAI GPT系列                |
| [anthropic_model.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/models/anthropic_model.py) | Anthropic Claude              |
| [gemini_model.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/models/gemini_model.py) | Google Gemini                 |
| [deepseek_model.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/models/deepseek_model.py) | DeepSeek                      |
| [qwen_model.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/models/qwen_model.py) | 通义千问                      |
| [zhipuai_model.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/models/zhipuai_model.py) | 智谱AI                        |
| [moonshot_model.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/models/moonshot_model.py) | 月之暗面                      |
| [ollama_model.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/models/ollama_model.py) | Ollama本地模型                |
| [litellm_model.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/models/litellm_model.py) | LiteLLM统一接口               |

#### 1.2.3 Societies 模块 ([camel/societies/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/societies/))

**多智能体社会协作框架** - 这是多智能体系统的核心：

| 文件/目录                                                    | 作用                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [role_playing.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/societies/role_playing.py) | `RolePlaying` - **双Agent角色扮演**（CAMEL最早的功能）       |
| [babyagi_playing.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/societies/babyagi_playing.py) | `BabyAGI` - 自主任务执行                                     |
| [workforce/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/societies/workforce/) | **`Workforce` - 多Agent工作力协作框架**（最新、最强大的多智能体系统） |

#### 1.2.4 Workforce 详细结构 ([camel/societies/workforce/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/societies/workforce/))



```
workforce/
├── __init__.py
├── base.py                  # BaseNode 基类
├── workforce.py            # Workforce 主类（核心协调器）
├── worker.py               # Worker 基类
├── single_agent_worker.py  # 单智能体工作节点
├── role_playing_worker.py  # 角色扮演工作节点
├── task_channel.py         # TaskChannel - 智能体间通信通道
├── prompts.py              # 提示词模板（ASSIGN_TASK_PROMPT、TASK_DECOMPOSE_PROMPT等）
├── utils.py                # 工具类（PipelineTaskBuilder、RecoveryStrategy等）
├── events.py               # 事件定义
└── ...
```

#### 1.2.5 Toolkits 模块 ([camel/toolkits/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/toolkits/))

**80+工具**覆盖各类场景：

| 文件                                                         | 功能                         |
| ------------------------------------------------------------ | ---------------------------- |
| [search_toolkit.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/toolkits/search_toolkit.py) | 搜索（DuckDuckGo、Google等） |
| [browser_toolkit.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/toolkits/browser_toolkit.py) | 浏览器自动化                 |
| [file_toolkit.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/toolkits/file_toolkit.py) | 文件操作                     |
| [github_toolkit.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/toolkits/github_toolkit.py) | GitHub API                   |
| [sql_toolkit.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/toolkits/sql_toolkit.py) | SQL数据库                    |
| [code_execution.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/toolkits/code_execution.py) | 代码执行                     |
| [memory_toolkit.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/toolkits/memory_toolkit.py) | 记忆工具                     |
| [task_planning_toolkit.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/toolkits/task_planning_toolkit.py) | **任务规划工具**             |

#### 1.2.6 其他重要模块

| 模块                                                         | 作用                             |
| ------------------------------------------------------------ | -------------------------------- |
| [messages/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/messages/) | 消息定义（`BaseMessage`）        |
| [memories/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/memories/) | 记忆系统                         |
| [storages/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/storages/) | 存储（向量DB、图DB等）           |
| [tasks/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/tasks/) | **任务定义（`Task`类）**         |
| [prompts/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/prompts/) | 提示词模板                       |
| [types/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/types/) | 类型定义和枚举                   |
| [datagen/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/datagen/) | 数据生成（CoT、Self-Instruct等） |
| [environments/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/environments/) | 环境模拟                         |
| [benchmarks/](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/benchmarks/) | 评估基准                         |

------

## 二、CAMEL 多智能体交互机制详解

### 2.1 核心架构

CAMEL的多智能体交互主要通过 **`Workforce`（工作力）** 框架实现，它采用**分层协作架构**：



```
Workforce (协调层)
├── Coordinator Agent (协调智能体) - 负责任务分配
├── Task Planner Agent (任务规划智能体) - 负责任务分解
└── Workers (工作节点层)
    ├── SingleAgentWorker (单智能体工作者)
    ├── RolePlayingWorker (角色扮演工作者)
    └── 嵌套的 Workforce (子工作力)
```

### 2.2 智能体通信机制 - TaskChannel

智能体之间通过 **`TaskChannel`（任务通道）** 进行**异步消息传递**：

#### 核心通信组件 ([task_channel.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/societies/workforce/task_channel.py:85))



```python
class TaskChannel:
    r"""An internal class used by Workforce to manage tasks.
    
    This implementation uses a hybrid data structure approach:
    - Hash map (_task_dict) for O(1) task lookup by ID
    - Status-based index (_task_by_status) for efficient filtering by status
    - Assignee/publisher queues for ordered task processing
    """
```

#### 通信流程

1. **任务发布** (`_post_task`): Workforce将任务包装为Packet发布到通道
2. **任务获取** (`get_assigned_task_by_assignee`): Worker原子性地获取分配给它的任务
3. **任务返回** (`return_task`): Worker完成任务后更新状态
4. **依赖管理** (`post_dependency`): 通过ARCHIVED状态的Packet存储依赖任务结果

#### 关键代码 ([task_channel.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/societies/workforce/task_channel.py:174-201))



```python
async def get_assigned_task_by_assignee(self, assignee_id: str) -> Task:
    r"""Atomically get and claim a task from the channel that has been
    assigned to the assignee. This prevents race conditions where multiple
    concurrent calls might retrieve the same task.
    """
    async with self._condition:
        while True:
            task_ids = self._task_by_assignee.get(assignee_id, deque())
            # Process all available tasks until we find a valid one
            while task_ids:
                task_id = task_ids.popleft()
                if task_id in self._task_dict:
                    packet = self._task_dict[task_id]
                    if (
                        packet.status == PacketStatus.SENT
                        and packet.assignee_id == assignee_id
                    ):
                        # Use helper method to properly update status
                        self._update_task_status(
                            task_id, PacketStatus.PROCESSING
                        )
                        self._condition.notify_all()
                        return packet.task
            await self._condition.wait()
```

### 2.3 任务状态流转



```
OPEN → RUNNING → DONE
  ↓      ↓        ↓
  └──────┴─────→ FAILED
```

------

## 三、规划策略：完整规划 vs 单步规划

### 3.1 关键结论：**先生成完整规划，再执行**

CAMEL的规划智能体（Task Planner Agent）采用的是**先完整分解，后统一执行**的策略，而不是执行一步再规划下一步。

### 3.2 证据分析

#### 证据1：任务处理流程 ([workforce.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/societies/workforce/workforce.py:2593-2637))



```python
async def process_task_async(
    self, task: Task, interactive: bool = False
) -> Task:
    # Handle different execution modes
    if self.mode == WorkforceMode.PIPELINE:
        return await self._process_task_with_pipeline(task)
    else:
        # AUTO_DECOMPOSE mode (default)
        subtasks = await self.handle_decompose_append_task(task)
        
        self.set_channel(TaskChannel())
        await self.start()
        
        if subtasks:
            task.result = "\n\n".join(
                f"--- Subtask {sub.id} Result ---\n{sub.result}"
                for sub in task.subtasks
                if sub.result
            )
```

#### 证据2：任务分解与执行分离 ([workforce.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/societies/workforce/workforce.py:2510-2591))



```python
async def handle_decompose_append_task(
    self, task: Task, reset: bool = True
) -> List[Task]:
    # ... validation ...
    
    # The agent tend to be overconfident on the whole task, so we
    # decompose the task into subtasks first
    subtasks_result = self._decompose_task(task, self._on_stream_callback)
    
    # Handle both streaming and non-streaming results
    if isinstance(subtasks_result, Generator):
        # This is a generator (streaming mode)
        subtasks = []
        for new_tasks in subtasks_result:
            subtasks.extend(new_tasks)
    else:
        # This is a regular list (non-streaming mode)
        subtasks = subtasks_result
    
    if subtasks:
        # _pending_tasks will contain both undecomposed
        # and decomposed tasks, so we use additional_info
        # to mark the tasks that need decomposition instead
        self._pending_tasks.extendleft(reversed(subtasks))
    else:
        # If no decomposition, execute the original task.
        self._pending_tasks.append(task)
    
    return subtasks
```

**关键点**：

1. 首先调用 `_decompose_task` 将任务分解为**完整的子任务列表**
2. 将所有子任务添加到 `_pending_tasks` 队列
3. 然后调用 `start()` 开始**统一执行**

### 3.3 执行流程详细说明



```
┌─────────────────────────────────────────────────────────────┐
│                     规划阶段 (Planning)                       │
├─────────────────────────────────────────────────────────────┤
│  1. 接收原始任务                                              │
│  2. Task Agent 调用 TASK_DECOMPOSE_PROMPT                    │
│  3. 生成完整的子任务列表 (包含所有步骤)                         │
│  4. 确定任务间的依赖关系                                       │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     执行阶段 (Execution)                      │
├─────────────────────────────────────────────────────────────┤
│  1. 将所有子任务加入 _pending_tasks 队列                       │
│  2. Coordinator Agent 根据Worker能力和任务需求进行分配          │
│  3. 启动 _listen_to_channel 主循环                            │
│  4. 按依赖关系逐步调度任务执行                                 │
│  5. Worker通过TaskChannel异步通信                             │
│  6. 处理完成的任务或失败的任务                                 │
└─────────────────────────────────────────────────────────────┘
```

### 3.4 任务分解提示词 ([prompts.py](vscode-webview://077q7kfc6bas63jfamemai8al610oa55aajt3l21ncgtuutj0r2f/camel/societies/workforce/prompts.py:194-309))



```python
TASK_DECOMPOSE_PROMPT = r"""You need to either decompose a complex task or enhance a simple one, following these important principles to maximize efficiency and clarity for the executing agents:

0.  **Analyze Task Complexity**: First, evaluate if the task is a single, straightforward action or a complex one.
    *   **If the task is complex or could be decomposed into multiple subtasks run in parallel, decompose it.** A task is considered complex if it involves multiple distinct steps, requires different skills, or can be significantly sped up by running parts in parallel.
    *   **If the task is simple, do not decompose it.** Instead, **rewrite and enhance** it to produce a high-quality task with a clear, specific deliverable.
...
You must output all subtasks strictly as individual <task> elements enclosed within a single <tasks> root.
"""
```

### 3.5 两种执行模式

CAMEL的Workforce支持两种执行模式：

| 模式               | 说明                                           | 适用场景           |
| ------------------ | ---------------------------------------------- | ------------------ |
| **AUTO_DECOMPOSE** | 自动分解模式，智能恢复策略（分解、重新规划等） | 复杂、不确定的任务 |
| **PIPELINE**       | 预定义管道模式，简单重试逻辑                   | 固定流程的任务     |



```python
class WorkforceMode(Enum):
    AUTO_DECOMPOSE = "auto_decompose"  # 自动任务分解模式
    PIPELINE = "pipeline"  # 预定义管道模式
```

### 3.6 与ReAct/Reflexion等范式的区别

| 范式                | 规划方式                        | 特点                           |
| ------------------- | ------------------------------- | ------------------------------ |
| **CAMEL Workforce** | **先完整分解，再执行**          | 适合复杂多智能体协作，并行执行 |
| ReAct               | 单步规划 → 执行 → 观察 → 下一步 | 适合单智能体逐步推理           |
| Reflexion           | 执行 → 反思 → 重新规划          | 强调自我改进                   |

------

## 四、总结

### 4.1 仓库定位

CAMEL是一个**大规模多智能体系统研究框架**，核心定位：

1. **多智能体系统研究与模拟** - 支持百万级Agent
2. **合成数据生成** - CoT、Self-Instruct等方法
3. **任务自动化** - 角色扮演、工作力协作、RAG管道
4. **世界模拟** - 多智能体环境交互和涌现行为研究

### 4.2 多智能体交互核心机制

1. **Workforce架构** - 分层协作（Coordinator + Task Planner + Workers）
2. **TaskChannel通信** - 异步消息传递，支持依赖管理
3. **完整规划策略** - 先生成完整子任务列表，再统一调度执行
4. **丰富的恢复策略** - retry、replan、decompose、create_worker、reassign

### 4.3 规划智能体工作方式

**回答你的核心问题**：CAMEL的规划智能体是**先生成完整规划，然后让后续智能体执行**，而不是生成单步规划、执行、观察、再生成下一步。

这种设计的好处：

- 可以**并行执行**独立的子任务
- 可以提前识别**任务依赖关系**
- 适合**多智能体协作**场景
- 支持**故障恢复和重新规划**（当某个子任务失败时）