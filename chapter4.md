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