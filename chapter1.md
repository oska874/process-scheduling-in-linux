# 1 Process Scheduling

The current Linux kernel is a multi-tasking kernel. Therefore, more than one process are allowed to exist at any given time and every process is allowed to run as if it were the only process on the 
system. The process scheduler coordinates which process runs when. In that context, it has the following tasks:

* share CPU equally among all currently running processes 
* pick appropriate process to run next if required, considering scheduling class/policy and process priorities
* balance processes between multiple cores in SMP systems

## 1.2 Linux Processes/Threads

Processes in Linux are a group of threads that share a thread group ID (TGID) and whatever resources necessary and does not differentiate between the two. The kernel schedules individual threads, not processes. Therefore, the term “task” will be used for the remainder of the document to refer to a thread.

task_struct (include/linux/sched.h) is the data structure used in Linux that contains all the information about a specific task.









