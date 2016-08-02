# 7. Completely Fair Scheduler (CFS)

Lets now describe the scheduling algorithm used for the NORMAL category tasks: The current scheduler is the completely fair scheduler by Ingo Molnar based on the Rotating Staircase Deadline Scheduler (RSDL) by Con Kolivas. CFS' implementation hides behind the fair_sched_class in /kernel/sched_fair.c.

The CFS algorithm is based on the idea of an ideal multi-tasking processor. Such a processor would run all currently active tasks literally at the same time while each task gets a portion of its processing power. For example, if two tasks would be runnable, they would run at the same time with 50% processing power each. That means, assuming the two tasks started at the same time, the execution time or runtime of each task is at any moment in time exactly the same, therefore completely fair. You could also say, each tasks runs for an infinitesimal small amount of time but with full processing power.

Since it is physically not possible to drive current processors in that way and it is highly inefficient to run for very short times due to switching cost, CFS tries to approximate this behaviour as closely as possible. It keeps track of the runtime of each runnable task, called virtual runtime, and tries to maintain an overall balance between all runnable tasks at all times.

The concept is based on giving all runnable tasks a fair share of the processor. The total share grows and shrinks with the total number of running processes and the relative share is inflated or deflated based on the tasks' priorities.

## 7.1. Time Slices vs. Virtual Runtime

In CFS, the traditional model of an epoch divided into fixed time slice for each task does not exist any more. Instead, a virtual runtime (vruntime) was introduced for each task. In every call of scheduler_tick() for a scheduled CFS task, its vruntime is updated with the elapsed time since it was scheduled. As soon as the vruntime of another task in the runqueue is smaller than the current tasks', a rescheduling is executed and the task with the smallest vruntime is selected to run next.

To avoid an immense overhead of scheduling runs due to frequent task switches, a scheduling latency and a minimum task runtime granularity were introduced.

The target scheduling latency (TSL) is a constant which is used to calculate a minimum task runtime. Lets assume TSL is 20ms and we have 2 running tasks of the same priority. Each task would then run for a total of 10ms before pre-empting in favour of the other.

If the number of tasks grows, this portion of runtime would shrink towards zero which again would lead to an inefficient switching rate. To counter this, a constant minimum granularity is used to stretch the TSL depending on the amount of running tasks.

## 7.2. Priorities

A task is prioritised above another by weighting how fast its vruntime grows while being scheduled. The time delta since the task was scheduled is weighted by the task's priority. That means, a high priority tasks' vruntime grows slower than the vruntime of a low priority one.

## 7.3. Handling I/O and CPU bound tasks

The previous scheduling algorithm, the O(1) scheduler, tried to use heuristic of sleep and runtimes to determine if a task is I/O or CPU bound. It would then benefit one or the other. As it turned out later, this concept did not work quite as satisfying as expected due to the complex and error prone heuristics.

CFS' concept of giving tasks a fair share of the processor works quite well and is easy to apply. I would like to explain how this works using the example given in Robert Love's Linux Kernel Development:

Consider a system with two runnable tasks: a text editor and a video encoder. The text editor is I/Obound because it spends nearly all its time waiting for user key presses. However, when the text editor does receive a key press, the user expects the editor to respond immediately. Conversely, the video encoder is processor-bound.

Aside from reading the raw data stream from the disk and later writing the resulting video, the encoder spends all its time applying the video codec to the raw data, easily consuming 100% of the processor. The video encoder does not have any strong time constraints on when it runs—if it started running now or in half a second, the user could not tell and would not care. Of course, the sooner it finishes the better, but latency is not a primary concern. In this scenario, ideally the scheduler gives the text editor a larger proportion of the available processor than the video encoder, because the text editor is interactive. We have two goals for the text editor. First, we want it to have a large amount of processor time available to it; not because it needs a lot of processor (it does not) but because we want it to always have processor time available the moment it needs it. Second, we want the text editor to pre-empt the video encoder the moment it wakes up (say, when the user presses a key). This can ensure the text editor has good interactive performance and is responsive to user input.

CFS solves this issue in the following way: Instead of assigning the text editor a specific priority and timeslice, it guarantees the text editor a specific proportion of the processor. If the video encoder and text editor are the only running processes and both are at the same nice level, this proportion would be 50% — each would be guaranteed half of the processor’s time. Because the text editor spends most of its time blocked, waiting for user key presses, it does not use anywhere near 50% of the processor. Conversely, the video encoder is free to use more than its allotted 50%, enabling it to finish the encoding quickly.

The crucial concept is what happens when the text editor wakes up. Our primary goal is to ensure it runs immediately upon user input. In this case, when the editor wakes up, CFS notes that it is allotted 50% of the processor but has used considerably less. Specifically, CFS determines that the text editor has run for less time than the video encoder. Attempting to give all processes a fair share of the processor, it then pre-empts the video encoder and enables the text editor to run. The text editor runs, quickly processes the user’s key press, and again sleeps, waiting for more input. As the text editor has not consumed its allotted 50%, we continue in this manner, with CFS always enabling the text editor to run when it wants and the video encoder to run the rest of the time.

## 7.4 Spawning a new Task

CFS maintains a vruntime_min value, which is a monotonic increasing value tracking the smallest vruntime among all tasks in the runqueue. This value is used for new tasks that are added to the runqueue to give them a chance of being scheduled quickly.

## 7.5 The Runqueue – Red-Black-Tree

As a runqueue, CFS uses a Red-Black-Tree data structure. This data structure is a self balancing binary search tree. Following a certain set of rules, no path in the tree will ever be more than twice as long as any other. Furthermore, operations on the tree occur in O(log n) time, which allows insertion and deletion of nodes to be quick and efficient.

Each node represents a task and they are ordered by the task's vruntime. That means the left most node in the tree always points to the task with the smallest vruntime, say the task that has the gravest need for the processor. Caching of the left most node makes it quick and easy to access.

## 7.6 Fair Group Scheduling

Group scheduling is another way to bring fairness to scheduling, particularly in the face of tasks that spawn many other tasks. Consider a server that spawns many tasks to parallelise incoming connections. Instead of all tasks being treated fairly uniformly, CFS introduces groups to account for this behaviour. The server process that spawns tasks share their vruntimes across the group (in a hierarchy), while the single task maintains its own independent vruntime. In this way, the single task receives roughly the same scheduling time as the group.

Let's say that next to the server tasks another user has only one task running on the machine. Without group scheduling, the second user would be treated unfairly in opposition to the server. With group scheduling, CFS would first try to be fair to the group and then fair to the processes in the group. Therefore, both users would get 50% share of the CPU.

Groups can be configured via a /proc interface.

---

# 7. 完全公平调度器

现在让我们描述一下用于 NORMAL 类型任务的调度算法：目前的调度器是 Ingo Molnar 基于 Con Kolivas 开发的 Rotating Staircase Deadline Scheduler（RSDL） 开发的完全公平调度算法。 CFS 的实现就隐藏在 `/kernel/sched_fair.c` 的 `fair_sched_class` 调度类。

CFS 算法是基于一个完美的多任务处理器的想法。这样一个处理器表面上同时会运行当前所有活动的任务，而每个任务都会获取一份属于自己的处理运算能力。举个例子，如果两个任务都是可运行的，那么他们就可以同时运行，分别占有 50% 的处理器运算能力。这也就意味着，加入这两个任务同时启动，每个任务的执行时间或者运行时间在任何时候都是相同的，因此是完全公平的。你也可以说每个任务都运行一个无穷小的时间但是完全占用全部的处理器运算能力。
既然物理上是不可能以这种方式使用处理器，并且这样子也是极度没有效率的，因为任务切换的损耗会导致实际任务运行时间很短。调度器会记录每个可运行的任务的运行时，即所谓的虚拟运行时，并且尝试随时在全部可运行任务之间维护均衡。

这个原则是基于给定的全部可运行任务是公平分享处理器的。全部份额会随着运行中的处理器个数变化而增加或者减少，而相对份额会基于任务的优先级变化而增加或减少。

## 7.1. 时间片 vs. 虚拟运行时

在 CFS 调度算法中，将每个任务的时间划分成固定长度的时间片这种传统模型已经不再存在了。作为替代，每个任务都引入了虚拟运行时（vruntime）。在为每一个使用 CFS 调度的任务执行 `scheduler_tick()` 时，在任务被调度以后，它的 vruntime 会和流逝的时间一起更新。一旦运行队列中另一个任务的 vruntime 比当前任务的小，那么就会执行一次任务调度，接下来 vruntime 最小的任务就会得到执行。

为了避免因为频繁进行任务切换而造成任务调度开销过大， Linux 引入了调度延迟和最小任务运行时间粒度。

目标调度延迟（TSL）是一个用来计算最小任务运行时间的常量。假如 TSL 是 20 ms，而我们有两个运行中的同等优先级任务。每个任务都会在被对方抢占前运行 10ms。

如果任务的数量在增加，则这部分的运行时会减少，甚至为 0 ，这就会导致一个毫无效率的切换速率。要应对这点，引入了常量“最小粒度”，根据运行中任务的数量来扩大 TSL 。

## 7.2. 优先级

调度中任务的 vruntime 增长的速度来决定了一个任务是否比另一个任务优先级高。时间差产生的原因是任务的调度是根据任务的优先级决定的。这也意味着高优先级任务的 vruntime 增长的要比低优先级任务慢。

## 7.3. 应对 I/O 限制型和 CPU 限制型任务

在之前的调度算法中， O(1) 调度其尝试使用启发式的睡眠和运行时来决定一个任务是 I/O 限制型任务还是 CPU 限制型任务。这样就可以让某个比另一个更占优势。根据之后的结果发现这个原则并不能满足预期的工作要求，因为启发法太复杂了而且容易出错。

而 CFS 的原则是为每个任务提供公平的计算机使用，实际工作效果相当好，也容易应用。我打算用 Robert Love 的 Linux Kernel Development 中的例子来解释 CFS 是怎么工作的 ：

考虑一个系统有两个可运行的任务：一个文本编辑器和一个视频编码器。文本编辑器是 I/O 限制型任务因为它花费了几乎全部的时间在等待用户的按键输入。然而当文本编辑器捕获到键盘按下后，用户就期待它立即响应。对应的，视频编码器就是一个处理器限制型任务。

除了从磁盘读取原始的数据流和将结果写入视频文件，编码器几乎全部的时间都用在了编解码视频原始数据上，很轻松的就会使用了 100% 的处理器。视频编码对何时运行并没有很强的限制——用户是不可能区分出是立刻运行还是半秒之后运行的，他们也不关心这一点。当然了，越快完成越好，但是延迟并不一个主要考虑的因素。在这种场景下，理想的调度其会给文本编辑器相较于视频编码器更多的处理器时间，因为文本编辑器是交互型的。我们对文本编辑器有两个目标。第一，我们希望他有大量可用的处理器时间；不是因为他需要大量处理器时间（它并不需要）而是因为我们希望他在需要处理器时总可以使用处理器。第二，我们希望文本编辑器在被唤醒时可以抢占视频编码器（也就是在当用户按下按键时）。这一点可以保证文本编辑器能够提供很好的交互性能和及时的响应用户输入。

CFS 通过下面的办法解决了这个问题：调度器保证文本编辑器有特定的处理器时间比例而不是赋给他特定的优先级和时间片。如果视频编码器和文本编辑器是仅有的运行中的进程，并且有着相同的友好级（nice level），这个比例可以是 50% —— 每个任务都保证占有一半的处理器时间。因为文本编辑器把大部分的时间花费到了阻塞上，等待用输入，所以他它没有用尽 10% 的处理器。相对的，视频编码器能够自由使用超过分配给自己的 50% 的处理器，这就是他能够快速的完成编码工作。

一个重要的原则当文本编辑器被唤醒时。我们的主要目标是保证它能根据用户的输入立即运行。在这种情况下，当编辑器唤醒后， CFS 注意到它分配了 50% 的处理器但是相当大的部分并没有使用。特别的是， CFS 确定了文本编辑器已经运行的时间比视频编码器要少。调度器会尝试给全部进程分配公平的的处理器时间，然后抢占了视频编码器并让文本编辑器开始运行。文本编辑器开始运行，迅速的处理用户的按下的按键，然后就进入睡眠等待更多的输入。因为文本编辑器并没有消耗完分配给它的 50% 处理器，我们继续这个方式， CFS 总会让文本编辑器能够在它想运行的时刻运行，然后视频编码器会运行在剩余的时间。

## 7.4. 创建新任务

CFS 维护了一个 vruntime_min 值，这是个单调递增的值，记录了运行队列中全部任务最小的 vruntime 值。这个值会加到运行队列上以此来给新任务一个更快的调度机会。

## 7.5. 运行队列 - 红黑树

CFS 在运行队列中使用红黑树数据结构。这个数据结构是一个自平衡二叉搜索树。遵循一套确定的规则，树中不会有一条路径会是另一条的两倍长。更进一步，这种树的操作时间复杂度是 O(log n)，这就允许快速、高效的插入和删除节点的操作。

树中没一个节点代表一个而任务，他们根据任务的 vruntime 排序。这就意味着二叉树最左边的节点总是指向 vruntime 最小的任务，也就是说这个任务最需要处理器。缓存最左边的节点可以更快和更方便的访问节点。

## 7.6. 公平组调度

组调度是另一种给任务调度增加公平性的途径，特别是面对任务又生成很多任务的情况。考虑到服务器会生成很多任务来并行处理收到的接入连接。 CFS 引入了组来描述这种行为而不是统一的公平对待全部任务。生成任务的服务器进程在组内（根据层级）共享他们的 vruntime ，而单独任务会维护自己独立的 vruntime。在这种办法中，但任务接收和组粗略相等调度时间。

让我们假设机器上除了服务器任务，另一个用户只有一个运行中的任务。如果没有组调度，第二个用户相对于服务器会受到不公平的对待。通过组调度， CFS 会首先尝试对组公平，然后再对组内进程公平。因此两个用户都获得 50% 的 CPU。

组可以通过 `/proc` 接口进行配置。
