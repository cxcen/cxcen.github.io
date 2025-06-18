---
title: Tokio runtime 设计与实现
type: post
categories:
  - rust
  - tokio
tags:
  - tokio
  - runtime
  - asynchronous
---

### Runtime 核心数据

#### Work

每个 worker 线程对应一个 Worker 实例，它包含：

- **本地任务队列 local**：

  - 类型：LifoDeque<Task>，最大容量 256。
  - 用于存储由本线程创建或唤醒的任务。
  - 优先被调度执行，只有在空时才访问全局队列。
  - 支持任务的 *工作窃取*（work stealing）。
  
- **LIFO Slot（后进先出槽）**：

  - 类型：Option<Task>。
  - 用于快速调度刚被当前线程唤醒的任务，避免任务落入队列从而节省调度开销。
  - 最多只能存一个任务；若已有任务，会将旧任务退回本地队列。
  - 每个线程独享，**不可被其它线程窃取**。
  
- **运行中的任务计数和调度指标**：

  - 用于决定是否切换调度来源，例如是否从本地队列换到全局队列。

  

#### GlobalQueue: 跨线程任务共享队列

- 类型：Injector<Task>（来自 crossbeam 的 work-stealing 队列实现）。

- 用途：所有线程可并发向此队列推入任务；当某线程本地队列为空时，会尝试从此队列中拉取任务。

- 特殊用例：来自非 worker 线程的 tokio::spawn 会将任务直接插入全局队列。

- 

### Scheduler: 调度器结构

每个 Tokio 多线程 runtime 启动时会创建一个 Scheduler，它持有：

- 所有的 Worker 实例（例如一个数组 [Worker; N]）；
- 一个全局队列 GlobalQueue；
- 控制调度行为的参数（如 global_queue_interval、event_interval）；
- Parker 与 Unparker，用于线程休眠与唤醒；
- 用于任务 stealing 的原子引用等辅助数据结构。

调度流程简略：

```
loop {
    // 优先从 lifo slot 取任务
    // 然后尝试从本地队列 pop
    // 如果空了，每隔 global_queue_interval 次尝试拉取全局任务
    // 若还空，尝试从其他 worker 的本地队列偷任务（steal）
    // 若全部失败，则线程休眠
}
```



#### Task：可调度的异步任务

- Tokio 内部封装了每个 Future 为 Task。

- Task 实际是指针封装（Arc<TaskCell>），里面包含：

  - RawTask 的状态控制器；
  - Waker；
  - Future 对象；
  - 执行上下文等。
  
- Task 被压入队列、从队列取出并 poll，是 Tokio 调度的最小单元。



#### Shared：所有线程共享的调度上下文

- 多线程调度器中的某些结构（如全局队列、metrics、配置项）被封装在一个共享的 Shared 结构中，所有 Worker 持有其引用。

- 类型典型是 Arc<Shared>

  

#### 可视化结构关系（简化）

```
Runtime
│
├─ Scheduler
│  ├─ GlobalQueue (Injector<Task>)
│  ├─ [Worker; N]
│      ├─ local: LifoDeque<Task>
│      ├─ lifo_slot: Option<Task>
│      ├─ metrics
│
└─ Shared
   ├─ runtime config
   ├─ event loop driver
```



Tokio 多线程调度器通过 **本地队列 + 全局队列 + LIFO slot + 工作窃取** 的组合设计，实现了：

- 高并发、低延迟的任务调度；
- 局部性优化（lifo + 本地队列）；
- 线程饥饿避免与全局公平性（全局队列 + stealing）；
- IO / timer 驱动集成（driver + event loop）；

它是一个 lightweight 且高度优化的任务调度器，适合处理大规模的异步负载。



### Task Steal流程分析

 steal_into2 是 Tokio 多线程调度器中工作窃取逻辑的核心实现之一。这个函数通过原子操作安全地从一个 worker 的本地任务队列中“偷”任务，并把它们搬运到另一个线程的本地缓冲区中。下面我将逐步拆解并解释它的关键逻辑与数据结构含义。

#### 数据结构

```rust
pub(crate) struct Inner<T> {
    head: AtomicUnsignedLong,  // 两个 UnsignedShort，打包成一个 u32/u64
    tail: AtomicUnsignedShort, // 本地 thread 使用
    buffer: Box<[UnsafeCell<MaybeUninit<Notified<T>>>; LOCAL_QUEUE_CAPACITY]>
}
```

其中 head 被编码为一个打包结构：(steal_head, real_head)，防止 ABA 问题 + 多线程冲突：
• real_head: 当前任务头部位置（被 local 线程或 stealer 消费）
• steal_head: 如果两者不同，说明一个 stealing 过程正在进行，其他偷线程必须等待



#### steal_into2 步骤详解

**Step 1：尝试原子声明“我要偷任务了”**

```rust
let (src_head_steal, src_head_real) = unpack(prev_packed);
let src_tail = self.0.tail.load(Acquire);
```

•	若 src_head_steal != src_head_real：说明别人已经在偷，不能同时进行。
•	计算可偷数量 n = (tail - head) / 2，防止过度搬运。

然后构造新的 head：

```rust
let steal_to = src_head_real.wrapping_add(n);
let next_packed = pack(src_head_steal, steal_to);
```

• 注意 steal_to 是你要“抢占到”的尾部，接下来用 CAS 尝试写入。



**Step 2：原子地声明“我偷到了这些任务段”**

```rust
self.0.head.compare_exchange(prev_packed, next_packed, AcqRel, Acquire)
```

• 成功后，当前线程就“独占”了 [src_head_real, steal_to) 这段任务。
• 所有其他 stealer 会发现 head_steal != head_real，而自动跳过。



**Step 3：真正去 buffer 里读取任务、写入目标缓冲区**

```rust
for i in 0..n {
    let src_idx = src_pos & MASK;
    let dst_idx = dst_pos & MASK;

let task = self.0.buffer[src_idx].with(|ptr| unsafe { ptr::read((*ptr).as_ptr()) });

dst.inner.buffer[dst_idx]
    .with_mut(|ptr| unsafe { ptr::write((*ptr).as_mut_ptr(), task) });

}
```

​	这里读写都是 UnsafeCell，因为 Rust 的类型系统不允许 &T 下的修改。我们确保当前没有数据竞争，因此使用 unsafe 是合理的。



**Step 4：原子更新 head_steal = head_real，宣告“偷取完成”**

```rust
let head = unpack(prev_packed).1;
let next_packed = pack(head, head);
```

再次使用 compare_exchange 将 head_steal 设置为 head_real，其他线程就可以继续尝试偷了。



**注意的点**
• 如果偷取过程中失败了（CAS失败或被别人抢先偷），会重试或放弃；
• 每次只偷任务队列中一半（n = n - n / 2），让本地线程保留一半任务，提升缓存局部性；
• 用 AtomicUnsignedLong + pack/unpack 技巧避免使用两个原子变量，同时解决同步和 ABA 问题；
• 整体保证：无锁、多线程安全、零拷贝搬运


