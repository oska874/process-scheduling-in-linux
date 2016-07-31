# 6 Short Scheduling Algorithm History

This is a short history of the development of the scheduling algorithm used in Linux.

- 1995 1.2 Circular Runqueue with processes scheduled in a Round Robin system
- 1999 2.2 introduced scheduling classes and with that rt-tasks, non-preemptible tasks and
non-rt-tasks, introduced support for SMP
- 2001 2.4 O(N) scheduler, split time into epochs where each task was allowed a certain time
slice, iterating through N runnable tasks and applying goodness function to determine next
task
- 2003 2.6 O(1) scheduler used multiple runqueues for each priority, it was a more efficient
and scalable version of O(N), it introduced a bonus system for interactive vs. batch tasks
- 2008 2.6.23 Completely Fair Scheduler