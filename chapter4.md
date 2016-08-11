# 4 Main Runqueue

The main per-CPU runqueue data structure is defined in kernel/sched.c. It keeps track of all runnable tasks assigned to a particular CPU and manages various scheduling statistics about CPU load or scheduling domains for load balancing. Furthermore it has:

- a lock to synchronize scheduling operations for this CPU   
  ```
  raw_spinlock_t lock;
  ```
- pointers to the task_structs of the currently running, the idle and the stop task 
  ```
  struct task_struct *curr, *idle, *stop;
  ```
- runqueue data structures for fair and real time scheduling classes
  ```
  struct cfs_rq cfs;
  struct rt_rq rt;
  ```
  
---
  
# 4. 主运行队列

主运行队列在每个 CPU 上都有一份，数据结构定义在 `kernel/sched.c`。它记录了特定 CPU 上所有的可运行的任务，并且还管理和每个处理器负载或域负载相关的不同调度的统计信息。更进一步来说，它包括下面几项：
  
- 一个锁，用来同步当前 CPU 的调度操作
  ```
  raw_spinlock_t lock;
  ```
- 分别指向当前运行的任务、空闲任务、停止任务的 `task_struct` 结构体指针
  ```
  struct task_struct *curr, *idle, *stop;
  ```
- 公平调度和实时调度类的运行队列数据结构
  ```
  struct cfs_rq cfs;
  struct rt_rq rt;
  ```



