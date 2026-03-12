# CAMEL Workforce 代码结构指南

本文档简要说明 `camel/societies/workforce/workforce.py` 的代码结构,标注关键行号和函数名。

**文件总行数**: 约 6250 行

---

## 一、文件头部导入区 (第1-130行)

**范围**: 第1-130行

**主要内容**:
- 标准库导入(asyncio, json, time等)
- Workforce相关模块导入(Worker、TaskChannel、提示词等)
- 系统常量定义

**关键常量**:
- `MAX_TASK_RETRIES` (第132行): 任务最大重试次数=3
- `TASK_TIMEOUT_SECONDS` (第134行): 任务超时=600秒

---

## 二、枚举与状态定义 (第141-205行)

**范围**: 第141-205行

| 名称 | 行号 | 说明 |
|------|------|------|
| `WorkforceState` | 第141行 | 执行状态枚举(IDLE/RUNNING/PAUSED/STOPPED) |
| `WorkforceMode` | 第159行 | 执行模式枚举(AUTO_DECOMPOSE/PIPELINE) |
| `WorkforceSnapshot` | 第175行 | 状态快照类(用于暂停/恢复) |

---

## 三、Workforce 主类 (第206行起)

### 3.1 类文档字符串 (第206-460行)

**范围**: 第206-460行

**说明**: 详细的中文文档,介绍整体架构和三个核心Agent的职责。

### 3.2 __init__ 初始化方法 (第461-795行)

**范围**: 第461-795行

**分阶段初始化**:

| 阶段 | 行号范围 | 说明 |
|------|----------|------|
| 阶段一: 基础配置 | 第483-510行 | 子节点、模型、执行模式等配置 |
| 阶段二: 任务状态追踪 | 第530-545行 | _pending_tasks、_task_dependencies等 |
| 阶段三: 管道与辅助 | 第558-565行 | 管道构建器、任务时间追踪 |
| 阶段四: 人工干预 | 第572-590行 | 暂停/恢复/快照支持 |
| 阶段五: 流式与共享内存 | 第604-617行 | 流式回调、共享内存UUID |
| 阶段六: 三个核心Agent | 第625-795行 | Coordinator/ Task/ New Worker Agent初始化 |

**三个核心Agent**:
1. **Coordinator Agent** (第625行): 任务分配
2. **Task Agent** (第693行): 任务分解
3. **New Worker Agent** (第753行): 动态Worker模板

---

## 四、核心执行方法

### 4.1 handle_decompose_append_task (约第2720行)

**功能**: 任务分解与入队

**执行流程**:
1. 验证任务内容
2. 重置Workforce状态
3. 调用 `_decompose_task()` 分解任务
4. 将子任务添加到 `_pending_tasks` 队列

**被调用**: `process_task_async()`

### 4.2 _decompose_task (约第1690行)

**功能**: 实际调用Task Agent进行任务分解

**关键操作**:
- 构建 `TASK_DECOMPOSE_PROMPT`
- 重置Task Agent状态
- 调用 `task.decompose()`
- 更新依赖关系

### 4.3 start (第6115行)

**功能**: 启动Workforce执行

**执行流程**:
1. 同步共享内存(如启用)
2. 启动所有子Worker
3. 进入 `_listen_to_channel()` 主循环

**被调用**: `process_task_async()`

### 4.4 _listen_to_channel (第5460行)

**功能**: 主事件循环

**循环逻辑**:
1. 检查暂停/停止/跳过请求
2. 检查队首任务是否需要分解
3. 等待任务返回 `_get_returned_task()`
4. 处理返回的任务(DONE/FAILED)
5. 发布新就绪任务 `_post_ready_tasks()`

### 4.5 _post_ready_tasks (第4570行)

**功能**: 任务调度与发布

**两个步骤**:
1. **任务分配**: 调用 `_find_assignee()` 获取Coordinator分配
2. **任务发布**: 检查依赖满足,调用 `_post_task()` 发布到TaskChannel

### 4.6 _find_assignee (第4220行)

**功能**: 任务分配决策

**执行流程**:
1. 等待Worker就绪(指数退避)
2. 调用 `_call_coordinator_for_assignment()`
3. 验证分配结果
4. 处理无效分配(重试/回退)

### 4.7 _call_coordinator_for_assignment (第3955行)

**功能**: 实际调用Coordinator Agent(LLM推理)

**关键操作**:
- 构建 `ASSIGN_TASK_PROMPT`
- 添加重试反馈(如有无效ID)
- 调用Coordinator Agent
- 解析结构化输出

### 4.8 _post_task (第4360行)

**功能**: 将任务发布到TaskChannel

**执行流程**:
1. 记录开始时间
2. 记录 `TaskStartedEvent`
3. 调用 `_channel.post_task()`
4. 增加 `_in_flight_tasks` 计数

---

## 五、依赖管理方法

| 方法名 | 约行号 | 功能 |
|--------|--------|------|
| `_update_dependencies_for_decomposition` | 第1440行 | 任务分解后更新依赖继承关系 |
| `_update_task_dependencies_from_assignments` | 第4160行 | 根据分配结果更新依赖 |

---

## 六、结果处理方法

| 方法名 | 约行号 | 功能 |
|--------|--------|------|
| `_handle_completed_task` | 第2840行 | 处理成功完成的任务 |
| `_handle_failed_task` | 第3040行 | 处理失败任务(触发恢复策略) |
| `_analyze_task` | 第1800行 | Task Agent分析失败原因/质量 |

---

## 七、执行流程图

```
process_task_async(task)              # 第2800行
    ↓
handle_decompose_append_task(task)    # 第2720行
    ↓
_decompose_task(task)                 # 第1690行
    ↓
start()                               # 第6115行
    ↓
_listen_to_channel()                  # 第5460行 [主循环]
    ├── _post_ready_tasks()           # 第4570行
    │       ├── _find_assignee()      # 第4220行
    │       │       └── _call_coordinator_for_assignment()  # 第3955行
    │       └── _post_task()          # 第4360行
    ├── _get_returned_task()          # 第4480行
    └── 处理返回任务:
            ├── DONE → _handle_completed_task()   # 第2840行
            └── FAILED → _handle_failed_task()    # 第3040行
```

---

## 八、关键数据结构

| 变量名 | 类型 | 约行号 | 说明 |
|--------|------|--------|------|
| `_pending_tasks` | Deque[Task] | 第533行 | 待处理任务队列 |
| `_task_dependencies` | Dict[str, List[str]] | 第535行 | 依赖关系图 |
| `_assignees` | Dict[str, str] | 第537行 | 任务→Worker映射 |
| `_completed_tasks` | List[Task] | 第587行 | 已完成任务列表 |
| `_in_flight_tasks` | int | 第539行 | 正在执行的任务数 |

---

## 九、提示词模板 (从prompts.py导入)

| 提示词名 | 用途 |
|----------|------|
| `TASK_DECOMPOSE_PROMPT` | 任务分解 |
| `ASSIGN_TASK_PROMPT` | 任务分配 |
| `TASK_ANALYSIS_PROMPT` | 失败分析/质量评估 |
| `CREATE_NODE_PROMPT` | 创建Worker节点 |

---

## 十、回调事件类型 (从events.py导入)

| 事件名 | 触发时机 |
|--------|----------|
| `TaskCreatedEvent` | 任务创建时 |
| `TaskDecomposedEvent` | 任务分解时 |
| `TaskAssignedEvent` | 任务分配时 |
| `TaskStartedEvent` | 任务开始时 |
| `TaskCompletedEvent` | 任务完成时 |
| `TaskFailedEvent` | 任务失败时 |
| `WorkerCreatedEvent` | Worker创建时 |

---

**文档生成时间**: 2026-03-12
