# ClickHouse PipelineExecutor 调度机制分析

## 概述

`PipelineExecutor` 是 ClickHouse 查询引擎中真正负责**调度算子（Processor）**的核心组件。它将查询计划表示为一个**有向无环图（DAG）**，每个节点是一个 Processor，边是端口之间的连接，并通过多线程协作高效地执行这个图。

本文分析的代码位于 `executeStepImpl` 函数（`src/Processors/Executors/PipelineExecutor.cpp`），这是每个工作线程真正执行的核心循环。

---

## 关键数据结构

### 1. ExecutingGraph::Node — 图中的节点

| 字段 | 说明 |
|------|------|
| `processor` | 指向实际的 `IProcessor` 对象 |
| `processors_id` | 处理器在图中的唯一 ID |
| `direct_edges` | 输出边（OutputPort → 下游 InputPort） |
| `back_edges` | 反向边（InputPort ← 上游 OutputPort） |
| `status` | 当前状态：`Idle` / `Preparing` / `Executing` / `Finished` / `Async` |
| `status_mutex` | 并发访问保护 |
| `updated_input_ports` / `updated_output_ports` | 自上次 `prepare()` 后发生变化的端口列表 |

**状态机流转：**

```
Idle ──→ Preparing ──→ Executing ──→ Idle/Finished/Async
  ↑                        │
  └────────────────────────┘ (再次就绪后重新调度)
```

### 2. ExecutionThreadContext — 单个线程的执行上下文

| 字段 | 说明 |
|------|------|
| `async_tasks` | 该线程的异步任务队列（IO 完成回调） |
| `condvar` / `mutex` / `wake_flag` | 等待/唤醒机制 |
| `node` | 当前正在执行的 Processor 节点 |
| `thread_number` | 线程编号 |
| `num_scheduled_local_tasks` | 连续本地调度次数（优化用） |

**本地调度优化：**

当一个线程执行完一个 Processor 后，如果下游节点已经就绪，可以直接在当前线程继续执行，而**不经过全局队列**，减少锁竞争。但每线程限制最多连续 128 次本地调度，防止饥饿（starvation）。

### 3. ExecutorTasks — 全局任务管理器

| 字段 | 说明 |
|------|------|
| `finished` | 是否已完成或取消 |
| `task_queue` | 全局就绪任务队列（`Preparing` 状态的节点） |
| `async_task_queue` | 异步任务队列（基于 epoll） |
| `threads_queue` | 等待中的线程集合 |
| `num_threads` | 最大线程数 |
| `use_threads` | 当前实际使用的线程数 |

---

## 执行流程详解（executeStepImpl）

```
┌─────────────────────────────────────────────────────┐
│               executeStepImpl                        │
│             (每个工作线程循环运行)                     │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
    ┌──────────────────────────────────────┐
    │  Phase 1: tryGetTask (获取任务)       │
    │  ┌────────────────────────────────┐   │
    │  │ ① 尝试本线程的异步任务          │   │
    │  │ ② 尝试全局任务队列              │   │
    │  │ ③ 有任务 → 唤醒其他线程         │   │
    │  │ ④ 无任务且是最后线程 → 结束     │   │
    │  │ ⑤ 无任务 → 阻塞等待            │   │
    │  └────────────────────────────────┘   │
    └──────────────────────────────────────┘
                          │
                          ▼
    ┌──────────────────────────────────────┐
    │  Phase 2: executeTask (执行任务)      │
    │  ┌────────────────────────────────┐   │
    │  │ 调用 processor->work()         │   │
    │  │ 实际处理数据（读/写/计算等）    │   │
    │  │ 失败则取消整个 Pipeline        │   │
    │  └────────────────────────────────┘   │
    └──────────────────────────────────────┘
                          │
                          ▼
    ┌──────────────────────────────────────┐
    │  Phase 3: updateNode (更新图状态)    │
    │  ┌────────────────────────────────┐   │
    │  │ 调用 processor->prepare()      │   │
    │  │ 根据返回状态决定后续：          │   │
    │  │  Ready     → 放入就绪队列       │   │
    │  │  NeedData  → 等上游推数据       │   │
    │  │  PortFull  → 等下游消费数据     │   │
    │  │  Finished  → 标记完成           │   │
    │  │  Async     → 等待异步 IO        │   │
    │  │  ExpandPipeline → 扩展图结构    │   │
    │  │  Exception → 取消执行           │   │
    │  └────────────────────────────────┘   │
    │  同时根据端口变化更新邻居节点状态       │
    └──────────────────────────────────────┘
                          │
                          ▼
    ┌──────────────────────────────────────┐
    │  Phase 4: pushTasks (推送任务)        │
    │  ┌────────────────────────────────┐   │
    │  │ ① 优先尝试本地调度（优化）      │   │
    │  │ ② 剩余任务放回全局队列          │   │
    │  │ ③ 唤醒等待线程                 │   │
    │  └────────────────────────────────┘   │
    └──────────────────────────────────────┘
                          │
                          ▼
    ┌──────────────────────────────────────┐
    │  Phase 5: spawnThreads (扩容)        │
    │  ┌────────────────────────────────┐   │
    │  │ 动态增加工作线程数              │   │
    │  │ 适应不同阶段的并行度需求        │   │
    │  └────────────────────────────────┘   │
    └──────────────────────────────────────┘
                          │
                          ▼
    ┌──────────────────────────────────────┐
    │  Phase 6: yield (让出控制权)         │
    │  ┌────────────────────────────────┐   │
    │  │ 外部设置 yield_flag 则让出      │   │
    │  │ 用于超时处理或优雅取消          │   │
    │  └────────────────────────────────┘   │
    └──────────────────────────────────────┘
                          │
                          ▼
                    (回到 Phase 1)
```

---

## 各阶段深入分析

### Phase 1: tryGetTask（`ExecutorTasks::tryGetTask`）

这是线程获取任务的核心逻辑：

```cpp
void ExecutorTasks::tryGetTask(ExecutionThreadContext & context)
{
    std::unique_lock lock(mutex);

    // 1. 优先处理本线程的异步任务（IO 完成回调）
    if (auto * async_task = context.tryPopAsyncTask())
    {
        context.setTask(async_task);
        --num_waiting_async_tasks;
        return;
    }

    // 2. 从全局队列取任务
    if (!task_queue.empty())
        context.setTask(task_queue.pop(context.thread_number));

    if (context.hasTask())
    {
        // 3. 唤醒其他线程处理剩余任务
        tryWakeUpAnyOtherThreadWithTasks(context, lock);
        return;
    }

    // 4. 没有任务：判断是否可以结束
    if (threads_queue.size() + 1 == use_threads  // 当前是唯一活跃线程
        && async_task_queue.empty()              // 没有异步任务等待
        && num_waiting_async_tasks == 0)         // 没有已分发的异步任务
    {
        finish();  // 所有工作都完成了，结束
        return;
    }

    // 5. 加入等待队列，阻塞等待被唤醒
    threads_queue.push(context.thread_number);
    context.wait(finished);
}
```

**关键设计**：
- 最后活跃线程负责检测是否所有工作都已完成。
- `num_waiting_async_tasks` 跟踪已分发但未完成的异步任务，防止在异步 IO 还在进行时就结束。

### Phase 2-3: executeTask + updateNode — 执行与就绪检查

这是 ClickHouse 调度模型的核心：

**两个阶段交替进行：**

```
                   work() ──→ 实际处理数据
                      │
                      ▼
              ┌───────────────┐
              │   端口状态变化    │
              │  (消费输入/产生输出)│
              └───────────────┘
                      │
                      ▼
                 prepare() ──→ 检查端口状态
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
      Ready         NeedData      Finished
    (可继续)       (等待上游)    (完成)
        │
        ▼
   再次 work()
```

`updateNode` 做的事情：
1. 调用 `processor->prepare()` 检查当前节点的就绪状态。
2. 根据 `prepare()` 返回的状态分类处理。
3. 检查端口 `update_info` 的变化，找出哪些邻居节点的端口状态也发生了变化，将受影响的邻居也标记为需要调度。

**为什么需要 `updateNode`？**

ClickHouse 的算子模型是**拉取与推送混合**：
- 当一个 Processor 执行 `work()` 后，它可能消费了输入端口的数据（产生更多空间给上游），也可能产生了输出数据（提供更多数据给下游）。
- 这些端口状态变化需要传播给邻居节点，让它们重新评估自己的就绪状态。

### Phase 4: pushTasks — 本地调度优化

```cpp
void ExecutorTasks::pushTasks(Queue & queue, Queue & async_queue, ExecutionThreadContext & context)
{
    context.setTask(nullptr);  // 清空当前任务

    // 如果 queue 非空，且连续本地调度次数未达上限
    if (!queue.empty() && !context.hasAsyncTasks()
        && context.num_scheduled_local_tasks < max_scheduled_local_tasks)
    {
        ++context.num_scheduled_local_tasks;
        context.setTask(queue.front());  // 本地调度
        queue.pop();
    }
    else
        context.num_scheduled_local_tasks = 0;  // 重置计数器

    // 剩余任务放入全局队列
    if (!queue.empty() || !async_queue.empty())
    {
        std::unique_lock lock(mutex);
        // ... 放回全局队列
        tryWakeUpAnyOtherThreadWithTasks(context, lock);
    }
}
```

**优化效果**：如果 `updateNode` 发现当前节点的下游节点已经就绪（例如 Merge 算子的输入都有了数据），这个下游节点会被放入 queue。通过本地调度，当前线程直接继续执行下游节点，避免了：
- 一次全局队列的 push（放回）
- 一次全局队列的 pop（重新获取）
- 两次锁操作

但限制 128 次的目的是：**如果一个线程"太强"（吃掉了所有机会），其他线程会饿死**。这是以轻微性能损失换取公平性。

### Phase 5: spawnThreads — 动态扩容

ClickHouse 的 Pipeline 可以在执行过程中动态增加并行度。例如：
- 初始阶段单线程读取
- 读取到数据后展开为多线程处理

`spawnThreads()` 会根据当前的并行度需求和可用资源，启动更多工作线程。

### 异步任务机制

Linux 上使用 epoll 管理异步 IO：

```
Processor 发起异步 IO 请求
        │
        ▼
返回 Async 状态，被放入 async_task_queue
        │
        ▼
epoll 等待 IO 完成事件
        │
        ▼
IO 完成后，将 Processor 推回给对应线程
        │
        ▼
线程从 tryPopAsyncTask() 获取任务，继续执行
```

这个机制使得 ClickHouse 可以在等待网络 IO（如远程读取）时不阻塞线程，线程可以去处理其他就绪的 Processor。

---

## 核心设计思想总结

### 1. 工作窃取（Work Stealing）

多个工作线程从一个全局就绪队列中取任务。空闲线程会尝试从队列中取任务，而不是固定分配给某个线程。这是经典的生产者-消费者模型。

### 2. 图状态驱动调度

ClickHouse 不是简单地"按顺序执行算子"，而是通过 `prepare()` 实时评估每个算子的就绪状态：
- 一个算子的输出端口变满后，它的上游（生产数据的算子）会被阻塞。
- 一个算子的输入端口被消费后，它的下游（消费数据的算子）可能会饥饿。
- `prepare()` 负责检测这些条件，决定谁可以继续执行。

### 3. 本地调度优化

通过在当前线程直接执行下一个就绪节点，减少了全局队列的竞争。但限制 128 次以防止饥饿。

### 4. 异步 IO 集成

基于 epoll 的异步事件驱动，使得 IO 密集型算子不会阻塞线程，提高了 CPU 利用率。

### 5. 动态扩容

执行过程中可以根据需要增加线程数，适应不同阶段的并行度需求。

### 6. 确定性终止

最后活跃线程负责检测终止条件：所有工作完成 + 没有异步任务等待 → 调用 `finish()` 通知所有线程退出。

---

## 参考代码位置

| 文件 | 说明 |
|------|------|
| `src/Processors/Executors/PipelineExecutor.cpp` | 主执行器，包含 `executeStepImpl` |
| `src/Processors/Executors/ExecutorTasks.h/.cpp` | 全局任务管理器 |
| `src/Processors/Executors/ExecutionThreadContext.h` | 线程执行上下文 |
| `src/Processors/Executors/ExecutingGraph.h` | 执行图（节点、边、状态管理） |
| `src/Processors/Executors/PollingQueue.h` | 异步任务队列（epoll 封装） |
| `src/Processors/Executors/ThreadsQueue.h` | 等待线程管理 |
| `src/Processors/Executors/TasksQueue.h` | 全局任务队列（NUMA 感知） |
