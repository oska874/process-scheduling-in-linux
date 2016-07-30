# 2. Task Classification

## 2.1. CPU-bound vs. I/O bound

Tasks tend to be either CPU-bound or I/O bound. That is, some threads spend a lot of time using the CPU to do computations, and others spend a lot of time waiting for relatively slow I/O operations to complete. I/O operations, in that context, can be waiting for user input, disk or network accesses.

This also strongly depends on the system the tasks are running on. A server or HPC (high performance computing) workload generally consists mostly of CPU-bound tasks whereas desktop or mobile workloads are mainly I/O bound.

The Linux OS runs on all those system and is therefore designed to handle all types of tasks. To cope with that, its scheduler needs to be responsive for I/O bound tasks and efficient for CPU-bound ones. If tasks run for longer periods of time, they can accomplish more work but responsiveness suffers. If time periods per task get shorter, the system can react faster on I/O events but more time is spent on running the scheduling algorithm between task switches and efficiency suffers. This puts the scheduler in a give and take situation which needs it to find a balance betweenthose two requirements.

## 2.2. Real Time vs. Normal

Furthermore, tasks running on Linux can explicitly be classified as real time (RT) or normal tasks.Real time tasks have strict timing requirements and therefore priority over any other task on the system. Different scheduling policies are used for RT and normal tasks which are implemented in the scheduler by using scheduling classes.

## 2.3. Task Priority Values

Higher priorities in the kernel have a numerical smaller value. Real time priorities range from 1 (highest) – 99 whereas normal priorities range from 100 – 139 (lowest). There can, however, be confusion when using system calls or scheduler library functions to set priorities. There, the numerical order can be reversed and/or mapped to different values (nice values).

# 2. 任务分类

## 2.1. CPU 限制 vs. I/O 限制

任务要么是限制在 CPU 要么限制在 I/O。也就是说，一些线程会使用 CPU 进行大量的运算，而另一些则会花费很多时间等待相对较慢的 I/O 操作完成。在本文， I/O 操作可能是等待用户输入，磁盘或网络访问。

同时这种分类也强烈依赖于任务运行的系统。一个服务器或者 HPC （超级计算机）的负载通常是 CPU 限制任务，而桌面或者移动平台的负载主要是 I/O 限制任务。

Linux 操作系统运行在这些系统之上，也因此设计时就是要应对所有类型的任务。要解决这个问题， Linux 的调度器需要负责及时响应 I/O 限制型任务，提高 CPU 限制型任务的效率。如果任务要运行更长的时间周期，这样他们呢就能完成更多的工作，但是响应性会受影响。如果每个任务的时间周期短一些，则系统能更快的相应 I/O 事件，但是这又会导致花费更多的时间在运行调度算法在任务切换上，效率会会受到影响。这就要求调度器需要有取有舍，并且要在两个需求之间取得平衡。

## 2.2. 实时 vs. 普通

更进一步，运行在 Linux 上的任务可以明确的分为实时（RT）任务和普通任务。实时任务有严格的时序要求，也因此比系统上的其他任务优先级要高。通过调度分类，针对 RT 任务和普通任务采用了不同的调度策略实现调度器。

## 2.3. 任务优先级值

在内核里，高的优先级在数值上更小。实时任务的优先级从 1（最高）到 99，而普通任务的优先级从 100 到 139（最低）。然而这样又会在使用系统调用或者调度器库函数修改任务优先级时产生混乱。因为数字的顺序可以被反转和/或被映射到不同的值（nice 值）




