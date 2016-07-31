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

## 5.2 Calling the Scheduler

After seeing the entry point into the scheduler, lets now have a look at when the schedule() function is actually called. There are three main occasions when that happens in kernel code:

### 1. Regular runtime update of currently scheduled task
The function scheduler_tick() is called regularly by a timer interrupt. Its purpose is to update runqueue clock, CPU load and runtime counters of the currently running task.

```
/*
* This function gets called by the timer code, with HZ frequency.
* We call it with interrupts disabled.
*/
void scheduler_tick(void)
{
    int cpu = smp_processor_id();
    struct rq *rq = cpu_rq(cpu);
    struct task_struct *curr = rq->curr;
    sched_clock_tick();
    raw_spin_lock(&rq->lock);
    update_rq_clock(rq);
    update_cpu_load_active(rq);
    curr->sched_class->task_tick(rq, curr, 0);
    raw_spin_unlock(&rq->lock);
    perf_event_task_tick();
#ifdef CONFIG_SMP
    rq->idle_at_tick = idle_cpu(cpu);
    trigger_load_balance(rq, cpu);
#endif
}
```

You can see, that scheduler_tick() calls the scheduling class hook task_tick() which runs the regular task update for the corresponding class. Internally, the scheduling class can decide if a new task needs to be scheduled an would set the need_resched flag for the task which tells the kernel to invoke schedule() as soon as possible.

At the end of scheduler_tick() you can also see that load balancing is invoked if SMP is configured.

### 2. Currently running task goes to sleep

The process of going to sleep to wait for a specific event to happen is implemented in the Linux kernel for multiple occasions. It usually follows a certain pattern:

```
/* ‘q’ is the wait queue we wish to sleep on */
DEFINE_WAIT(wait);
add_wait_queue(q, &wait);
while (!condition)   /* condition is the event that we are waiting for */
{
    prepare_to_wait(&q, &wait, TASK_INTERRUPTIBLE);
    if (signal_pending(current))
        /* handle signal */
        schedule();
}
finish_wait(&q, &wait);
```

The calling task would create a wait queue and add itself to it. It would then start a loop that waits until a certain condition becomes true. In the loop, it would set its own task state to either TASK_INTERRUPTIBLE or TASK_UNITERRUPTIBLE. If the former is the case, it can be woken up for a pending signal which can then be handled.

If the needed event did not occur yet, it would call schedule() and go to sleep. schedule() would then remove the task from the runqueue (See Scheduler Entry Point). If the condition becomes true, the loop is exited and the task is removed from the wait queue.

You can see here, that schedule() is always called right before a task goes to sleep, to pick another task to run next.

### 3. Sleeping task wakes up

The code that causes the event the sleeping task is waiting for typically calls wake_up() on the corresponding wait queue which eventually ends up in the scheduler function try_to_wake_up() (ttwu).
This function does three things:

1. It puts the task to be woken back into the runqueue.
2. It wakes the task up by setting its state to TASK_RUNNING.
3. If the the awakened task has higher priority than the currently running task, the need_resched flag is set to invoke schedule().

```
/**
* try_to_wake_up - wake up a thread
* @p: the thread to be awakened
* @state: the mask of task states that can be woken
* @wake_flags: wake modifier flags (WF_*)
*
* Put it on the run-queue if it's not already there. The "current"
* thread is always on the run-queue (except when the actual
* re-schedule is in progress), and as such you're allowed to do
* the simpler "current->state = TASK_RUNNING" to mark yourself
* runnable without the overhead of this.
*
* Returns %true if @p was woken up, %false if it was already running
* or @state didn't match @p's state.
*/
static int
try_to_wake_up(struct task_struct *p, unsigned int state, int wake_flags)
{
    unsigned long flags;
    int cpu, success = 0;
    smp_wmb();
    raw_spin_lock_irqsave(&p->pi_lock, flags);
    if (!(p->state & state))
        goto out;
    success = 1; /* we're going to change ->state */
    cpu = task_cpu(p);
    if (p->on_rq && ttwu_remote(p, wake_flags))
        goto stat;
#ifdef CONFIG_SMP
    /*
    * If the owning (remote) cpu is still in the middle of schedule() with
    * this task as prev, wait until its done referencing the task.
    */
    while (p->on_cpu) {
#ifdef __ARCH_WANT_INTERRUPTS_ON_CTXSW
        /*
        * In case the architecture enables interrupts in
        * context_switch(), we cannot busy wait, since that
        * would lead to deadlocks when an interrupt hits and
        * tries to wake up @prev. So bail and do a complete
        * remote wakeup.
        */
        if (ttwu_activate_remote(p, wake_flags))
            goto stat;
#else
        cpu_relax();
#endif
    }
    /*
    * Pairs with the smp_wmb() in finish_lock_switch().
    */
    smp_rmb();
    p->sched_contributes_to_load = !!task_contributes_to_load(p);
    p->state = TASK_WAKING;
    if (p->sched_class->task_waking)
        p->sched_class->task_waking(p);
    cpu = select_task_rq(p, SD_BALANCE_WAKE, wake_flags);
    if (task_cpu(p) != cpu) {
        wake_flags |= WF_MIGRATED;
        set_task_cpu(p, cpu);
    }
#endif /* CONFIG_SMP */
    ttwu_queue(p, cpu);
stat:
    ttwu_stat(p, cpu, wake_flags);
out:
    raw_spin_unlock_irqrestore(&p->pi_lock, flags);
    return success;
}
```

After some initial error checking and some SMP magic, the function ttwu_queue() is called which does the wake up work.

```
static void ttwu_queue(struct task_struct *p, int cpu)
{
    struct rq *rq = cpu_rq(cpu);
#if defined(CONFIG_SMP)
    if (sched_feat(TTWU_QUEUE) && cpu != smp_processor_id()) {
        sched_clock_cpu(cpu); /* sync clocks x-cpu */
        ttwu_queue_remote(p, cpu);
        return;
    }
#endif
    raw_spin_lock(&rq->lock);
    ttwu_do_activate(rq, p, 0);
    raw_spin_unlock(&rq->lock);
}
static void
ttwu_do_activate(struct rq *rq, struct task_struct *p, int wake_flags)
{
#ifdef CONFIG_SMP
    if (p->sched_contributes_to_load)
        rq->nr_uninterruptible--;
#endif
    ttwu_activate(rq, p, ENQUEUE_WAKEUP | ENQUEUE_WAKING);
    ttwu_do_wakeup(rq, p, wake_flags);
}
```

This function locks the runqueue and calls ttwu_do_activate() which further calls ttwu_activate() to perform step 1 and ttwu_do_wakeup() to perform step 2 and 3.

```
static void ttwu_activate(struct rq *rq, struct task_struct *p, int en_flags)
{
    activate_task(rq, p, en_flags);
    p->on_rq = 1;
    /* if a worker is waking up, notify workqueue */
    if (p->flags & PF_WQ_WORKER)
        wq_worker_waking_up(p, cpu_of(rq));
}
/*
* activate_task - move a task to the runqueue.
*/
static void activate_task(struct rq *rq, struct task_struct *p, int flags)
{
    if (task_contributes_to_load(p))
        rq->nr_uninterruptible--;
    enqueue_task(rq, p, flags);
    inc_nr_running(rq);
}
static void enqueue_task(struct rq *rq, struct task_struct *p, int flags)
{
    update_rq_clock(rq);
    sched_info_queued(p);
    p->sched_class->enqueue_task(rq, p, flags);
}
```

If you follow the chain of ttwu_activate() you end up in a call to the corresponding scheduling class hook to enqueue_task() which we already saw in schedule() to put the task back into the runqueue.

```
/*
* Mark the task runnable and perform wakeup-preemption.
*/
static void
ttwu_do_wakeup(struct rq *rq, struct task_struct *p, int wake_flags)
{
    trace_sched_wakeup(p, true);
    check_preempt_curr(rq, p, wake_flags);
    p->state = TASK_RUNNING;
#ifdef CONFIG_SMP
    if (p->sched_class->task_woken)
        p->sched_class->task_woken(rq, p);
    if (rq->idle_stamp) {
        u64 delta = rq->clock - rq->idle_stamp;
        u64 max = 2*sysctl_sched_migration_cost;
        if (delta > max)
            rq->avg_idle = max;
        else
            update_avg(&rq->avg_idle, delta);
        rq->idle_stamp = 0;
    }
#endif
}
static void check_preempt_curr(struct rq *rq, struct task_struct *p, int flags)
{
    const struct sched_class *class;
    if (p->sched_class == rq->curr->sched_class) {
        rq->curr->sched_class->check_preempt_curr(rq, p, flags);
    } else {
        for_each_class(class) {
            if (class == rq->curr->sched_class)
                break;
            if (class == p->sched_class) {
                resched_task(rq->curr);
                break;
            }
        }
    }
    /*
    * A queue event has occurred, and we're going to schedule. In
    * this case, we can save a useless back to back clock update.
    */
    if (rq->curr->on_rq && test_tsk_need_resched(rq->curr))
        rq->skip_clock_update = 1;
}
```

ttwu_do_wakeup() checks if the current task needs to be pre-empted by the task being woken up which is now in the runqueue. The function check_preempt_curr() ends up calling the corresponding hook into the scheduling class internally might set the need_resched flag. Afterwards the task's state is set to TASK_RUNNING, which completes the wake up process.

---

# 5. 调度框架

## 5.1. 调度器入口

进入进程调度器的主入口是函数 `schedule()` ，定义在文件 `kernel/sched.c`。内核其余部分都要使用这个函数来调用进程调度器，决定那个进程运行以及接下来运行那个进程。

`schedule()` 的主要目标是挑选下一个运行的任务，并且把这个进程传给本地变量 `next`。最后它就执行上下文切换进入新的任务。如果只有 `prev` 并且 `prev` 还可以运行，那么重新调度基本上就意味着 `schedule()` 什么都没改变。

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

因为 Linux 内核是可以抢占的，所以在内核空间执行的任务代码是可以被高优先级任务随机打断的。这就会使得被抢占的任务在内核空间没有执行完的操作只能在下一次调度执行了。因此调度函数要做的第一件事是调用 `preempt_disable()` 禁止抢占，这样以来调度中的线程就不会在执行临界区操作时被打断。

第二步，系统会通过锁定当前 CPU 的运行队列锁来建立另一个保护，因为统一时刻只能有一个线程被允许修改运行队列。

接下来，`schedule()` 要检查 `prev` 指向的之前执行的任务。如果这个任务在内核模式是不可运行的并且还没有被抢占，那么它就应该被移出运行队列。然而如果该任务有非阻塞的等待信号，那就把它的运行状态设置为 TASK_RUNNING 然后继续留在运行队列里。这就意味着 `prev` 指向的任务有获取了一次被调度执行的机会。

要将一个任务移出运行队列需要在内部调用 `deactivate_task()` ，这个函数会调用该任务所属的调度类的钩子函数 `dequeue_task()`。

```
static void dequeue_task(struct rq *rq, struct task_struct *p, int flags)
{
    update_rq_clock(rq);
    sched_info_dequeued(p);
    p->sched_class->dequeue_task(rq, p, flags);
}
```

下一个动作是检查当前 CPU 的运行队列是否还有可以运行的任务。如果没有的话就调用 `idle_balance()` 从其他 CPU 获取一些可以运行的任务（参见负载均衡）。

`put_prev_task()` 是一个调度类的钩子函数，使用来告诉任务所属的类这个任务将要被切换出当前 CPU 。

现在就要通过调用 `pick_next_task()` 来命令相应的调度类挑选下一个适合在当前 CPU 上运行的任务。紧接着就是清掉之前最开始为了调用 `schedule()` 而设置的 `need_resched` 标志。

`need_resched` 标志位于结构体 `task_struct` ，并且定期内核会检查该标志。这是一种告诉内核其它任务应该运行了、`schedule()` 函数需要尽快被调用的途径。

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




















