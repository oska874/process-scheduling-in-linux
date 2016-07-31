# 6. Short Scheduling Algorithm History

This is a short history of the development of the scheduling algorithm used in Linux.

* 1995 1.2 Circular Runqueue with processes scheduled in a Round Robin system
* 1999 2.2 introduced scheduling classes and with that rt-tasks, non-preemptible tasks and
  non-rt-tasks, introduced support for SMP
* 2001 2.4 O\(N\) scheduler, split time into epochs where each task was allowed a certain time
  slice, iterating through N runnable tasks and applying goodness function to determine next
  task
* 2003 2.6 O\(1\) scheduler used multiple runqueues for each priority, it was a more efficient
  and scalable version of O\(N\), it introduced a bonus system for interactive vs. batch tasks
* 2008 2.6.23 Completely Fair Scheduler

---

# 6. 调度算法简史

这里是一份 Linux 的调度算法的开发历史：

* 1995 1.2 循环运行队列与采用循环调度进程调度算法
* 1999 2.2 引入了调度类和实时任务，不可抢占任务和非实时任务，开始支持 SMP
* 2001 2.4 O\(N\) 复杂度的调度器，将时间分片，每个任务只被允许一段特定时间片，遍历 N 可运行任务，使用 goodness 函数来决定下一个运行的任务
* 2003 2.6 O\(1\) 复杂度的调度器，针对每个优先级使用了多个运行队列，是一个更有效且可扩展的 O\(N\) 版调度算法，引入了奖励系统来应对交互式 vs 批处理任务
* 2008 2.6.23 完全公平调度器

