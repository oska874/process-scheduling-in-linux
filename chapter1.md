# 1. Process Scheduling

The current Linux kernel is a multi-tasking kernel. Therefore, more than one process are allowed to exist at any given time and every process is allowed to run as if it were the only process on the system. The process scheduler coordinates which process runs when. In that context, it has the following tasks:

* share CPU equally among all currently running processes 
* pick appropriate process to run next if required, considering scheduling class/policy and process priorities
* balance processes between multiple cores in SMP systems

## 1.2 Linux Processes/Threads

Processes in Linux are a group of threads that share a thread group ID (TGID) and whatever resources necessary and does not differentiate between the two. The kernel schedules individual threads, not processes. Therefore, the term “task” will be used for the remainder of the document to refer to a thread.

task_struct (include/linux/sched.h) is the data structure used in Linux that contains all the information about a specific task.



# 1 进程调度

现在的 Linux 内核是一个多任务内核。因此，任何时刻都可以有不止一个进程存在，并且每个进程运行起来就像系统只存在它自己一个进程。进程调度器负责何时运行那个进程。这种情况下，调度其要完成下面的任务：

- 为所有运行的进程公平的分配处理器资源
- 如果需要的话挑选合适的进程作在下一次进程切换后运行，这需要考虑调度类型和策略，以及进程优先级
- 在 SMP 系统中要在多个处理器核心之间均衡进程运行

## 1.2. Linux 的进程和线程

在 Linux 中进程就是一组共享了线程组 ID （TGID）的线程和所需的必要资源，并且两者之间并没有什么不同。内核调度的是不同的线程而不是进程。因此名词 “任务（task）” 在本文中表示的是线程。

在 Linux 中 `task_struct`（`include/linux/sched.h`）是用来表示一个特定任务的全部信息的结构体。









