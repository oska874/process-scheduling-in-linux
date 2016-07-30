# 5 Scheduler Skeleton

## 5.1 The Scheduler Entry Point

The main entry point into the process scheduler is the function schedule(), defined in
kernel/sched.c. This is the function that the rest of the kernel uses to invoke the process scheduler,deciding which process to run and then running it.

Its main goal is to find the next task to be run and assign it to the local variable next. In the end, it then executes a context switch to that new task. If no other task than prev is found and prev is still runnable, it is rescheduled which basically means schedule() changes nothing.

Lets look at it in more detail:

```
/*
* __schedule() is the main scheduler function.
*/
static void __sched __schedule(void)
{
    struct task_struct *prev, *next;
    unsigned long *switch_count;
    struct rq *rq;
    int cpu;
need_resched:
    preempt_disable();
    cpu = smp_processor_id();
    rq = cpu_rq(cpu);
    rcu_note_context_switch(cpu);
    prev = rq->curr;
    schedule_debug(prev);
    if (sched_feat(HRTICK))
        hrtick_clear(rq);
    raw_spin_lock_irq(&rq->lock);
    switch_count = &prev->nivcsw;
    if (prev->state && !(preempt_count() & PREEMPT_ACTIVE)) {
        if (unlikely(signal_pending_state(prev->state, prev))) {
            prev->state = TASK_RUNNING;
        } else {
            deactivate_task(rq, prev, DEQUEUE_SLEEP);
            prev->on_rq = 0;
            /*
            * If a worker went to sleep, notify and ask workqueue
            * whether it wants to wake up a task to maintain
            * concurrency.
            */
            if (prev->flags & PF_WQ_WORKER) {
                struct task_struct *to_wakeup;
                to_wakeup = wq_worker_sleeping(prev, cpu);
                if (to_wakeup)
                    try_to_wake_up_local(to_wakeup);
            }
        }
        switch_count = &prev->nvcsw;
    }
    pre_schedule(rq, prev);
    if (unlikely(!rq->nr_running))
        idle_balance(cpu, rq);
    put_prev_task(rq, prev);
    next = pick_next_task(rq);
    clear_tsk_need_resched(prev);
    rq->skip_clock_update = 0;
    if (likely(prev != next)) {
        rq->nr_switches++;
        rq->curr = next;
        ++*switch_count;
        context_switch(rq, prev, next); /* unlocks the rq */
        /*
        * The context switch have flipped the stack from under us
        * and restored the local variables which were saved when
        * this task called schedule() in the past. prev == current
        * is still correct, but it can be moved to another cpu/rq.
        */
        cpu = smp_processor_id();
        rq = cpu_rq(cpu);
    } else
        raw_spin_unlock_irq(&rq->lock);
    post_schedule(rq);
    preempt_enable_no_resched();
    if (need_resched())
        goto need_resched;
}
```

Since the Linux kernel is pre-emptive, it can happen that a task executing code in kernel space is involuntarily pre-empted by a higher priority task. This pauses the pre-empted task within an unfinished kernel space operation that is only continued when it is scheduled next. Therefore, the first thing, the schedule function does is disabling pre-emption by calling preempt_disable() so the scheduling thread can not be pre-empted during critical operations.

Secondly, it establishes another protection by locking the current CPU's runqueue lock since only one thread at a time is allowed to modify the runqueue.

Next, schedule() examines the state of the previously executed task in prev. If it is not runnable and has not been pre-empted in kernel mode, then it should be removed from the runqueue. However, if it has nonblocked pending signals, its state is set to TASK_RUNNING and it is left in the runqueue.This means prev gets another chance to be selected for execution.

To remove a task from the runqueue, deactivate_task() is called which internally calls the
dequeue_task() hook of the task's scheduling class.

```
static void dequeue_task(struct rq *rq, struct task_struct *p, int flags)
{
    update_rq_clock(rq);
    sched_info_dequeued(p);
    p->sched_class->dequeue_task(rq, p, flags);
}
```
The next action is to check if any runnable tasks exist in the CPU's runqueue. If not, idle_balance() is called to get some runnable tasks from other CPUs (see Load Balancing).

put_prev_task() is a scheduling class hook that informs the task's class that the given task is about to be switched out of the CPU.

Now the corresponding class is asked to pick the next suitable task to be scheduled on the CPU by calling the hook pick_next_task(). This is followed by clearing the need_resched flag which might have been set previously to invoke the schedule() function call in the first place.

The need_resched is a flag in the task_struct and regularly checked by the kernel. It is a way of telling the kernel that another task deserves to run and schedule() should be executed as soon as possible.

```
/*
* Pick up the highest-prio task:
*/
static inline struct task_struct *
pick_next_task(struct rq *rq)
{
    const struct sched_class *class;
    struct task_struct *p;
    /*
    * Optimization: we know that if all tasks are in
    * the fair class we can call that function directly:
    */
    if (likely(rq->nr_running == rq->cfs.nr_running)) {
        p = fair_sched_class.pick_next_task(rq);
        if (likely(p))
            return p;
    }
    for_each_class(class) {
        p = class->pick_next_task(rq);
        if (p)
            return p;
    }
    BUG(); /* the idle class will always have a runnable task */
}
```

pick_next_task() is also implemented in sched.c. It iterates through our list of scheduling classes to find the class with the highest priority that has a runnable task (see Scheduling Classes above). If the class is found, the scheduling class hook is called. Since most tasks are handled by the sched_fair class, a short cut to this class is implemented in the beginning of the function.

Now schedule() checks if pick_next_task() found a new task or if it picked the same task again that was running before. If the latter is the case, no task switch is performed and the current task just keeps running. If a new task is found, which is the more likely case, the actual task switch is executed by calling context_switch(). Internally, context_switch() switches to the new task's memory map and swaps register state and stack.

To finish up, the runqueue is unlocked and pre-emption is reenabled. In case pre-emption was requested during the time in which it was disabled, schedule() is run again right away.

---
