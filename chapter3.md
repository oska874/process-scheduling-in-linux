# 3 Scheduling Classes

3 Scheduling Classes
The Linux scheduler is modular, enabling different algorithms/policies to schedule different types
of tasks. An algorithm's implementation is wrapped in a so called scheduling class. A scheduling
class offers an interface to the main scheduler skeleton which it can use to handle tasks according to
the implemented algorithm.
The sched_class data structure can be found in include/linux/sched.h:

```
struct sched_class {
const struct sched_class *next;
void (*enqueue_task) (struct rq *rq, struct task_struct *p, int flags);
void (*dequeue_task) (struct rq *rq, struct task_struct *p, int flags);
void (*yield_task) (struct rq *rq);
bool (*yield_to_task) (struct rq *rq, struct task_struct *p, bool preempt);
void (*check_preempt_curr) (struct rq *rq, struct task_struct *p, int flags);
struct task_struct * (*pick_next_task) (struct rq *rq);
void (*put_prev_task) (struct rq *rq, struct task_struct *p);
#ifdef CONFIG_SMP
int (*select_task_rq)(struct task_struct *p, int sd_flag, int flags);
void (*pre_schedule) (struct rq *this_rq, struct task_struct *task);
void (*post_schedule) (struct rq *this_rq);
void (*task_waking) (struct task_struct *task);
void (*task_woken) (struct rq *this_rq, struct task_struct *task);
void (*set_cpus_allowed)(struct task_struct *p,
const struct cpumask *newmask);
void (*rq_online)(struct rq *rq);
void (*rq_offline)(struct rq *rq);
#endif
void (*set_curr_task) (struct rq *rq);
void (*task_tick) (struct rq *rq, struct task_struct *p, int queued);
void (*task_fork) (struct task_struct *p);
void (*switched_from) (struct rq *this_rq, struct task_struct *task);
void (*switched_to) (struct rq *this_rq, struct task_struct *task);
void (*prio_changed) (struct rq *this_rq, struct task_struct *task,
int oldprio);
unsigned int (*get_rr_interval) (struct rq *rq,
struct task_struct *task);
#ifdef CONFIG_FAIR_GROUP_SCHED
void (*task_move_group) (struct task_struct *p, int on_rq);
#endif
};
```

Except for the first one, all members of this struct are function pointers which are used by the
scheduler skeleton to call the corresponding policy implementation hook.
All existing scheduling classes in the kernel are in a list which is ordered by the priority of the
scheduling class. The first member in the struct called next is a pointer to the next scheduling class
with a lower priority in that list.
The list is used to prioritise tasks of different types before others. In the Linux versions described in
this document, the complete list looks like the following:
stop_sched_class → rt_sched_class → fair_sched_class → idle_sched_class → NULL
Stop and Idle are special scheduling classes. Stop is used to schedule the per-cpu stop task which