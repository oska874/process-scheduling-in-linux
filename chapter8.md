# 8. CFS Implementation Details

## 8.1. Data Structures

With the CFS scheduler, a structure called sched_entity was introduced into the Linux scheduler. It is mainly used for time accounting for the single tasks and was added as the se member to each task's task_struct. Its defined in include/linux/sched.h:

```
struct sched_entity {

    struct load_weight load; /* for load-balancing */
    struct rb_node run_node;
    struct list_head group_node;
    unsigned int on_rq;
    u64 exec_start;
    u64 sum_exec_runtime;
    u64 vruntime;
    u64 prev_sum_exec_runtime;
    u64 nr_migrations;

#ifdef CONFIG_SCHEDSTATS
    struct sched_statistics statistics;
#endif

#ifdef CONFIG_FAIR_GROUP_SCHED
    struct sched_entity *parent;
    /* rq on which this entity is (to be) queued: */
    struct cfs_rq *cfs_rq;
    /* rq "owned" by this entity/group: */
    struct cfs_rq *my_q;
#endif

};
```

Another CFS related field, called cfs, was added to the main runqueue data structure. It is of the type cfs_rq, which is implemented in kernel/sched.c. It contains a list of pointers to all running CFS tasks, the root of CFS' red-black-tree, a pointer to the left most node, min_vruntime, pointers to previously and currently scheduled tasks and additional members for group and smp scheduling and load balancing. The priority of the task is encoded in the load_weight data structure.


```
/* CFS-related fields in a runqueue */

struct cfs_rq {
    struct load_weight load;
    unsigned long nr_running;
    u64 exec_clock;
    u64 min_vruntime;

#ifndef CONFIG_64BIT
    u64 min_vruntime_copy;
#endif

    struct rb_root tasks_timeline;
    struct rb_node *rb_leftmost;
    struct list_head tasks;
    struct list_head *balance_iterator;

    /*
    * 'curr' points to currently running entity on this cfs_rq.
    * It is set to NULL otherwise (i.e when none are currently running).
    */
    struct sched_entity *curr, *next, *last, *skip;

#ifdef CONFIG_SCHED_DEBUG
    unsigned int nr_spread_over;
#endif

#ifdef CONFIG_FAIR_GROUP_SCHED
    struct rq *rq; /* cpu runqueue to which this cfs_rq is attached */
    /*
    * leaf cfs_rqs are those that hold tasks (lowest schedulable entity in
    * a hierarchy). Non-leaf lrqs hold other higher schedulable entities
    * (like users, containers etc.)
    *
    * leaf_cfs_rq_list ties together list of leaf cfs_rq's in a cpu. This
    * list is used during load balance.
    */
    int on_list;
    struct list_head leaf_cfs_rq_list;
    struct task_group *tg;/* group that "owns" this runqueue */

#ifdef CONFIG_SMP
    /*
    * the part of load.weight contributed by tasks
    */
    unsigned long task_weight;

    /*
    * h_load = weight * f(tg)
    *
    * Where f(tg) is the recursive weight fraction assigned to
    * this group.
    */

    unsigned long h_load;

    /*
    * Maintaining per-cpu shares distribution for group scheduling
    *
    * load_stamp is the last time we updated the load average
    * load_last is the last time we updated the load average and saw load
    * load_unacc_exec_time is currently unaccounted execution time
    */
    u64 load_avg;
    u64 load_period;
    u64 load_stamp, load_last, load_unacc_exec_time;
    unsigned long load_contribution;
#endif
#endif

};
```

## 8.2 Time Accounting

As mentioned above, vruntime is used to track the virtual runtime of runnable tasks in CFS' redblack-tree. The scheduler_tick() function of the scheduler skeleton regularly calls the task_tick() hook into CFS. This hook internally calls task_tick_fair() which is the entry point into the CFS task update:

```
/*
* scheduler tick hitting a task of our scheduling class:
*/
static void task_tick_fair(struct rq *rq, struct task_struct *curr, int queued)
{
    struct cfs_rq *cfs_rq;
    struct sched_entity *se = &curr->se;
    for_each_sched_entity(se) {
        cfs_rq = cfs_rq_of(se);
        entity_tick(cfs_rq, se, queued);
    }

}
```

task_tick_fair() calls entity_tick() for the tasks scheduling entity and corresponding runqueue. entity_tick() executes two main tasks: First, it updates runtime statistics for the currently scheduled task and secondly, it checks if the current task needs to be pre-empted.

```
entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr, int queued)
{
    /*
    * Update run-time statistics of the 'current'.
    */
    update_curr(cfs_rq);
    /*
    * Update share accounting for long-running entities.
    */
    update_entity_shares_tick(cfs_rq);
#ifdef CONFIG_SCHED_HRTICK
    /*
    * queued ticks are scheduled to match the slice, so don't bother
    * validating it and just reschedule.
    */
    if (queued) {
        resched_task(rq_of(cfs_rq)->curr);
        return;
    }
    /*
    * don't let the period tick interfere with the hrtick preemption
    */
    if (!sched_feat(DOUBLE_TICK) &&
        hrtimer_active(&rq_of(cfs_rq)->hrtick_timer))
        return;
#endif
    if (cfs_rq->nr_running > 1 || !sched_feat(WAKEUP_PREEMPT))
        check_preempt_tick(cfs_rq, curr);
}
```

update_curr() is the responsible function to update the current task's runtime statistics. It calculates the elapsed time since the current task was scheduled last and passes the result, delta_exec, on to __update_curr().

```
static void update_curr(struct cfs_rq *cfs_rq)
{
    struct sched_entity *curr = cfs_rq->curr;
    u64 now = rq_of(cfs_rq)->clock_task;
    unsigned long delta_exec;
    if (unlikely(!curr))
        return;
    /*
    * Get the amount of time the current task was running
    * since the last time we changed load (this cannot
    * overflow on 32 bits):
    */
    delta_exec = (unsigned long)(now - curr->exec_start);
    if (!delta_exec)
        return;
    __update_curr(cfs_rq, curr, delta_exec);
    curr->exec_start = now;
    if (entity_is_task(curr)) {
        struct task_struct *curtask = task_of(curr);
        trace_sched_stat_runtime(curtask, delta_exec, curr->vruntime);
        cpuacct_charge(curtask, delta_exec);
        account_group_exec_runtime(curtask, delta_exec);
    }
}
```

The runtime delta is weighted by current task's priority, which is encoded in load_weigth, and the result is added to the current task's vruntime. This is the location where vruntime grows faster or slower, depending on the task' priority. You can also see that __update_curr() updates min_vruntime.


```
/*
 * Update the current task's runtime statistics. Skip current tasks that
 * are not in our scheduling class.
*/
static inline void __update_curr(struct cfs_rq *cfs_rq, struct sched_entity *curr,unsigned long delta_exec)
{
    unsigned long delta_exec_weighted;
    schedstat_set(curr->statistics.exec_max,max((u64)delta_exec, curr->statistics.exec_max));
    curr->sum_exec_runtime += delta_exec;
    schedstat_add(cfs_rq, exec_clock, delta_exec);
    delta_exec_weighted = calc_delta_fair(delta_exec, curr);
    curr->vruntime += delta_exec_weighted;
    update_min_vruntime(cfs_rq);
#if defined CONFIG_SMP && defined  CONFIG_FAIR_GROUP_SCHED
    cfs_rq->load_unacc_exec_time += delta_exec;
#endif
}
```

In addition to this regular update, update_current() is also called if the corresponding task becomes runnable or goes to sleep.

Back to entity_tick(). After the current task was updated, check_preempt_tick() is called to satisfy CFS' concept of giving each task a fair share. vruntime of the current process is checked against vruntime of the left most task in the red-black-tree to decide if a process switch is necessary.


```
/*
* Preempt the current task with a newly woken task if needed:
*/
static void
check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
    unsigned long ideal_runtime, delta_exec;
    ideal_runtime = sched_slice(cfs_rq, curr);
    delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;
    if (delta_exec > ideal_runtime) {
        resched_task(rq_of(cfs_rq)->curr);
        /*
        * The current task ran long enough, ensure it doesn't get
        * re-elected due to buddy favours.
        */
        clear_buddies(cfs_rq, curr);
        return;
    }
    /*
    * Ensure that a task that missed wakeup preemption by a
    * narrow margin doesn't have to wait for a full slice.
    * This also mitigates buddy induced latencies under load.
    */
    if (!sched_feat(WAKEUP_PREEMPT))
        return;
    if (delta_exec < sysctl_sched_min_granularity)
        return;
    if (cfs_rq->nr_running > 1) {
        struct sched_entity *se = __pick_first_entity(cfs_rq);
        s64 delta = curr->vruntime - se->vruntime;
        if (delta < 0)
            return;
        if (delta > ideal_runtime)
            resched_task(rq_of(cfs_rq)->curr);
    }
}
```

sched_slice() returns the target latency runtime for the current task, depending on the number of runnable processes. If its delta runtime is larger than this amount, the need_resched flag is set for this task.

If not, the runtime is checked against the minimum granularity. If the task ran longer than that and more than one tasks are in total in the red-black-tree, a comparison with the left most node is done.If the vruntime difference between those two is positive, that means the current task ran longer than the left most one, and larger than the target latency runtime calculated above, need_resched flag is set as well to reschedule ASAP.

## 8.3 Modifying the Red-Black-Tree

In Scheduler Skeleton you could see how tasks where deactivated when they were removed from the runqueue or activated when woken up in try_to_wake_up(). For the CFS scheduling class, enqueue_task_fair() and dequeue_task_fair() are called which end up in enqueue_entity() and dequeue_entity() to modify the red-black-tree.

In enqueue_entity(), you can see how the tasks vruntime is updated, as mentioned above, and that __enqueue_entity() is called. This function is responsible for the red-black-tree insert magic that fulfils the data structures properties. You can also find caching operations of the left most node there.

```
static void enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
    /*
    * Update the normalized vruntime before updating min_vruntime
    * through callig update_curr().
    */
    if (!(flags & ENQUEUE_WAKEUP) || (flags & ENQUEUE_WAKING))
        se->vruntime += cfs_rq->min_vruntime;
    /*
    * Update run-time statistics of the 'current'.
    */
    update_curr(cfs_rq);
    update_cfs_load(cfs_rq, 0);
    account_entity_enqueue(cfs_rq, se);
    update_cfs_shares(cfs_rq);
    if (flags & ENQUEUE_WAKEUP) {
        place_entity(cfs_rq, se, 0);
        enqueue_sleeper(cfs_rq, se);
    }
    update_stats_enqueue(cfs_rq, se);
    check_spread(cfs_rq, se);
    if (se != cfs_rq->curr)
        __enqueue_entity(cfs_rq, se);
    se->on_rq = 1;
    if (cfs_rq->nr_running == 1)
        list_add_leaf_cfs_rq(cfs_rq);
}
```

Removing a node works in a similar way. First, vruntime is updated and then the task is removed from the tree using __dequeue_entity().

```
static void dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
    /*
    * Update run-time statistics of the 'current'.
    */
    update_curr(cfs_rq);
    update_stats_dequeue(cfs_rq, se);
    if (flags & DEQUEUE_SLEEP) {
#ifdef CONFIG_SCHEDSTATS
        if (entity_is_task(se)) {
            struct task_struct *tsk = task_of(se);
            if (tsk->state & TASK_INTERRUPTIBLE)
                se->statistics.sleep_start = rq_of(cfs_rq)->clock;
            if (tsk->state & TASK_UNINTERRUPTIBLE)
                se->statistics.block_start = rq_of(cfs_rq)->clock;
        }
#endif
    }
    clear_buddies(cfs_rq, se);
    if (se != cfs_rq->curr)
        __dequeue_entity(cfs_rq, se);
    se->on_rq = 0;
    update_cfs_load(cfs_rq, 0);
    account_entity_dequeue(cfs_rq, se);
    /*
    * Normalize the entity after updating the min_vruntime because the
    * update can refer to the ->curr item and we need to reflect this
    * movement in our normalized position.
    */
    if (!(flags & DEQUEUE_SLEEP))
        se->vruntime -= cfs_rq->min_vruntime;
    update_min_vruntime(cfs_rq);
    update_cfs_shares(cfs_rq);
}
```

The additional calls you see are necessary for group scheduling, scheduling statistic updates and the buddy system which is used for picking the next task to run.

## 8.4 Picking the next runnable task

The main scheduler function schedule() calls pick_next_task() of the scheduling class with the highest priority that has runnable tasks. If this is called for the CFS class, the class hook calls pick_next_task_fair().

If no tasks are in this class, NULL is returned immediately. Otherwise pick_next_entity() is called to select the next task from the tree. This is then forwarded to set_next_entity() which removes it from the tree since scheduled processes are not allowed in there. The while loop is used to account for fair group scheduling.

```
static struct task_struct *pick_next_task_fair(struct rq *rq)
{
    struct task_struct *p;
    struct cfs_rq *cfs_rq = &rq->cfs;
    struct sched_entity *se;
    if (!cfs_rq->nr_running)
        return NULL;
    do {
        se = pick_next_entity(cfs_rq);
        set_next_entity(cfs_rq, se);
        cfs_rq = group_cfs_rq(se);
    } while (cfs_rq);
    p = task_of(se);
    hrtick_start_fair(rq, p);
    return p;
}
```

In pick_next_entity() not necessarily the left most task is picked to run next. A buddy system gives a certain degree of freedom in selecting the next task to run. This is supposed to benefit cache locality and group scheduling.

```
/*
* Pick the next process, keeping these things in mind, in this order:
* 1) keep things fair between processes/task groups
* 2) pick the "next" process, since someone really wants that to run
* 3) pick the "last" process, for cache locality
* 4) do not run the "skip" process, if something else is available
*/
static struct sched_entity *pick_next_entity(struct cfs_rq *cfs_rq)
{
    struct sched_entity *se = __pick_first_entity(cfs_rq);
    struct sched_entity *left = se;
    /*
    * Avoid running the skip buddy, if running something else can
    * be done without getting too unfair.
    */
    if (cfs_rq->skip == se) {
        struct sched_entity *second = __pick_next_entity(se);
        if (second && wakeup_preempt_entity(second, left) < 1)
            se = second;
    }
    /*
    * Prefer last buddy, try to return the CPU to a preempted task.
    */
    if (cfs_rq->last && wakeup_preempt_entity(cfs_rq->last, left) < 1)
        se = cfs_rq->last;
    /*
    * Someone really wants this to run. If it's not unfair, run it.
    */
    if (cfs_rq->next && wakeup_preempt_entity(cfs_rq->next, left) < 1)
        se = cfs_rq->next;
    clear_buddies(cfs_rq, se);
    return se;
}
```

__pick_first_entity() picks the left most node from the tree, which is very easy and fast as you can see below:

```
static struct sched_entity *__pick_first_entity(struct cfs_rq *cfs_rq)
{
    struct rb_node *left = cfs_rq->rb_leftmost;
    if (!left)
        return NULL;
    return rb_entry(left, struct sched_entity, run_node);
}
```


---

# 8. CFS 实现细节

## 8.1. 数据结构

CFS 调度器为 Linux 的进程调度器引入了一个名叫 `sched_entity` 的结构体。它最主要的用处是审计单个任务的时间，并且作为一个成员添加到了每个任务的 `task_struct` 结构体中。它定义在文件 `include/linux/sched.h` 中：

```
struct sched_entity {

    struct load_weight load; /* for load-balancing */
    struct rb_node run_node;
    struct list_head group_node;
    unsigned int on_rq;
    u64 exec_start;
    u64 sum_exec_runtime;
    u64 vruntime;
    u64 prev_sum_exec_runtime;
    u64 nr_migrations;

#ifdef CONFIG_SCHEDSTATS
    struct sched_statistics statistics;
#endif

#ifdef CONFIG_FAIR_GROUP_SCHED
    struct sched_entity *parent;
    /* rq on which this entity is (to be) queued: */
    struct cfs_rq *cfs_rq;
    /* rq "owned" by this entity/group: */
    struct cfs_rq *my_q;
#endif

};
```

另一个和 CFS 相关的领域是一个添加到运行队列的数据结构的成员变量 `cfs`。它的类型是 `cfs_rq` ，在文件 `kernel/sched.c` 中实现。它包含一个由指向全部运行中的 CFS 任务的指针的链表，CFS 调度器的红黑树的根，一个只想最左边节点的指针，最小虚拟时间（min_vruntime)，指向前一个任务和当前调度的任务的指针以及额外的用于公平组调度和 SMP 调度和负载运行的成员变量。任务的优先级已经编码到了 `load_weight` 数据结构中。

```
/* CFS-related fields in a runqueue */

struct cfs_rq {
    struct load_weight load;
    unsigned long nr_running;
    u64 exec_clock;
    u64 min_vruntime;

#ifndef CONFIG_64BIT
    u64 min_vruntime_copy;
#endif

    struct rb_root tasks_timeline;
    struct rb_node *rb_leftmost;
    struct list_head tasks;
    struct list_head *balance_iterator;

    /*
    * 'curr' points to currently running entity on this cfs_rq.
    * It is set to NULL otherwise (i.e when none are currently running).
    */
    struct sched_entity *curr, *next, *last, *skip;

#ifdef CONFIG_SCHED_DEBUG
    unsigned int nr_spread_over;
#endif

#ifdef CONFIG_FAIR_GROUP_SCHED
    struct rq *rq; /* cpu runqueue to which this cfs_rq is attached */
    /*
    * leaf cfs_rqs are those that hold tasks (lowest schedulable entity in
    * a hierarchy). Non-leaf lrqs hold other higher schedulable entities
    * (like users, containers etc.)
    *
    * leaf_cfs_rq_list ties together list of leaf cfs_rq's in a cpu. This
    * list is used during load balance.
    */
    int on_list;
    struct list_head leaf_cfs_rq_list;
    struct task_group *tg;/* group that "owns" this runqueue */

#ifdef CONFIG_SMP
    /*
    * the part of load.weight contributed by tasks
    */
    unsigned long task_weight;

    /*
    * h_load = weight * f(tg)
    *
    * Where f(tg) is the recursive weight fraction assigned to
    * this group.
    */

    unsigned long h_load;

    /*
    * Maintaining per-cpu shares distribution for group scheduling
    *
    * load_stamp is the last time we updated the load average
    * load_last is the last time we updated the load average and saw load
    * load_unacc_exec_time is currently unaccounted execution time
    */
    u64 load_avg;
    u64 load_period;
    u64 load_stamp, load_last, load_unacc_exec_time;
    unsigned long load_contribution;
#endif
#endif

};
```

## 8.2. 时间审计

如上所述， vruntime 是用来记录 CFS 的红黑树中的可运行任务的虚拟时间的。调度器框架的函数 `scheduler_tick()` 周期性调用 CFS 跟新任务的钩子函数 `task_tick()` ：


```
/*
* scheduler tick hitting a task of our scheduling class:
*/
static void task_tick_fair(struct rq *rq, struct task_struct *curr, int queued)
{
    struct cfs_rq *cfs_rq;
    struct sched_entity *se = &curr->se;
    for_each_sched_entity(se) {
        cfs_rq = cfs_rq_of(se);
        entity_tick(cfs_rq, se, queued);
    }

}
```

`task_tick_fair()` 调用 `entity_tick()` 是为了任务调度实体和相应的运行队列。`entity_tick()` 有两个主要任务：第一是为当前调度的任务更新运行时数据，第二是检查当前任务是否需要被抢占。

```
entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr, int queued)
{
    /*
    * Update run-time statistics of the 'current'.
    */
    update_curr(cfs_rq);
    /*
    * Update share accounting for long-running entities.
    */
    update_entity_shares_tick(cfs_rq);
#ifdef CONFIG_SCHED_HRTICK
    /*
    * queued ticks are scheduled to match the slice, so don't bother
    * validating it and just reschedule.
    */
    if (queued) {
        resched_task(rq_of(cfs_rq)->curr);
        return;
    }
    /*
    * don't let the period tick interfere with the hrtick preemption
    */
    if (!sched_feat(DOUBLE_TICK) &&
        hrtimer_active(&rq_of(cfs_rq)->hrtick_timer))
        return;
#endif
    if (cfs_rq->nr_running > 1 || !sched_feat(WAKEUP_PREEMPT))
        check_preempt_tick(cfs_rq, curr);
}
```

函数 `update_curr()` 负责更新当前任务的运行时数据。它会计算当前任务自从上次被调用以后使用了的时间，然后使用函数 `__update_curr()` 传递结果 `delta_exec` 。

```
static void update_curr(struct cfs_rq *cfs_rq)
{
    struct sched_entity *curr = cfs_rq->curr;
    u64 now = rq_of(cfs_rq)->clock_task;
    unsigned long delta_exec;
    if (unlikely(!curr))
        return;
    /*
    * Get the amount of time the current task was running
    * since the last time we changed load (this cannot
    * overflow on 32 bits):
    */
    delta_exec = (unsigned long)(now - curr->exec_start);
    if (!delta_exec)
        return;
    __update_curr(cfs_rq, curr, delta_exec);
    curr->exec_start = now;
    if (entity_is_task(curr)) {
        struct task_struct *curtask = task_of(curr);
        trace_sched_stat_runtime(curtask, delta_exec, curr->vruntime);
        cpuacct_charge(curtask, delta_exec);
        account_group_exec_runtime(curtask, delta_exec);
    }
}
```

运行时差值可以加权当前任务的优先级，而这个值是编码在 `load_weigth` 当中的，而且这个结果会加到当前任务的 `vruntime` 。这就是 `vruntime` 增长的更快或者更慢的地方，这依赖于任务的优先级，同时你也可以看到 `__update_curr()` 更新了 `min_vruntime` 。


```
/*
 * Update the current task's runtime statistics. Skip current tasks that
 * are not in our scheduling class.
*/
static inline void __update_curr(struct cfs_rq *cfs_rq, struct sched_entity *curr,unsigned long delta_exec)
{
    unsigned long delta_exec_weighted;
    schedstat_set(curr->statistics.exec_max,max((u64)delta_exec, curr->statistics.exec_max));
    curr->sum_exec_runtime += delta_exec;
    schedstat_add(cfs_rq, exec_clock, delta_exec);
    delta_exec_weighted = calc_delta_fair(delta_exec, curr);
    curr->vruntime += delta_exec_weighted;
    update_min_vruntime(cfs_rq);
#if defined CONFIG_SMP && defined  CONFIG_FAIR_GROUP_SCHED
    cfs_rq->load_unacc_exec_time += delta_exec;
#endif
}
```

除了周期性更新外， 在对应的任务变成可运行或者将要睡眠的时候 `update_current()` 也会被调用。

回到 `entity_tick()`。在更新了当前任务之后， 会调用 `check_preempt_tick()` 来满足 CFS 的基本原则：为每个任务提供一个公平的 CPU 份额。当前进程的 `vruntime` 会和红黑树中最左边的任务进行比较，以此来决定进程切换是否是必须的。

```
/*
* Preempt the current task with a newly woken task if needed:
*/
static void
check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
    unsigned long ideal_runtime, delta_exec;
    ideal_runtime = sched_slice(cfs_rq, curr);
    delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;
    if (delta_exec > ideal_runtime) {
        resched_task(rq_of(cfs_rq)->curr);
        /*
        * The current task ran long enough, ensure it doesn't get
        * re-elected due to buddy favours.
        */
        clear_buddies(cfs_rq, curr);
        return;
    }
    /*
    * Ensure that a task that missed wakeup preemption by a
    * narrow margin doesn't have to wait for a full slice.
    * This also mitigates buddy induced latencies under load.
    */
    if (!sched_feat(WAKEUP_PREEMPT))
        return;
    if (delta_exec < sysctl_sched_min_granularity)
        return;
    if (cfs_rq->nr_running > 1) {
        struct sched_entity *se = __pick_first_entity(cfs_rq);
        s64 delta = curr->vruntime - se->vruntime;
        if (delta < 0)
            return;
        if (delta > ideal_runtime)
            resched_task(rq_of(cfs_rq)->curr);
    }
}
```

删除一个节点的方法也是类似的。首先更新 `vruntime` 使用然后 `__dequeue_entity()` 将任务从树上删除掉。

```
static void dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
    /*
    * Update run-time statistics of the 'current'.
    */
    update_curr(cfs_rq);
    update_stats_dequeue(cfs_rq, se);
    if (flags & DEQUEUE_SLEEP) {
#ifdef CONFIG_SCHEDSTATS
        if (entity_is_task(se)) {
            struct task_struct *tsk = task_of(se);
            if (tsk->state & TASK_INTERRUPTIBLE)
                se->statistics.sleep_start = rq_of(cfs_rq)->clock;
            if (tsk->state & TASK_UNINTERRUPTIBLE)
                se->statistics.block_start = rq_of(cfs_rq)->clock;
        }
#endif
    }
    clear_buddies(cfs_rq, se);
    if (se != cfs_rq->curr)
        __dequeue_entity(cfs_rq, se);
    se->on_rq = 0;
    update_cfs_load(cfs_rq, 0);
    account_entity_dequeue(cfs_rq, se);
    /*
    * Normalize the entity after updating the min_vruntime because the
    * update can refer to the ->curr item and we need to reflect this
    * movement in our normalized position.
    */
    if (!(flags & DEQUEUE_SLEEP))
        se->vruntime -= cfs_rq->min_vruntime;
    update_min_vruntime(cfs_rq);
    update_cfs_shares(cfs_rq);
}
```

你所看到的其它函数调用是组调度，更新调度数据和伙伴系统所必须的，伙伴系统用来挑选下一个要运行的任务。

## 8.4. 挑选下一个可运行的任务

主要的调度函数 `schedule()` 会调用拥有可运行任务的最高优先级的调度类的 `pick_next_task()` 。如果调用的是 CFS 调度类的这个函数，调度类钩子函数会调用 `pick_next_task_fair()`。

如果这个调度类中没有任务，则直接返回 `NULL`。否则调用 `pick_next_entity()` 从树中挑选下一个任务。之后会转发给 `set_next_entity()` 来将任务从树上删除，因为调度了的进程是不允许继续存在这里。这个 `while` 循环使用来进行公平组调度的。

```
static struct task_struct *pick_next_task_fair(struct rq *rq)
{
    struct task_struct *p;
    struct cfs_rq *cfs_rq = &rq->cfs;
    struct sched_entity *se;
    if (!cfs_rq->nr_running)
        return NULL;
    do {
        se = pick_next_entity(cfs_rq);
        set_next_entity(cfs_rq, se);
        cfs_rq = group_cfs_rq(se);
    } while (cfs_rq);
    p = task_of(se);
    hrtick_start_fair(rq, p);
    return p;
}
```

在 `pick_next_entity()` 中，树中最左边的任务并不一定会被选出来运行。伙伴系统会给出选择下一个要运行的任务的等级。这被认为是有益于缓存位置和组调度。

```
/*
* Pick the next process, keeping these things in mind, in this order:
* 1) keep things fair between processes/task groups
* 2) pick the "next" process, since someone really wants that to run
* 3) pick the "last" process, for cache locality
* 4) do not run the "skip" process, if something else is available
*/
static struct sched_entity *pick_next_entity(struct cfs_rq *cfs_rq)
{
    struct sched_entity *se = __pick_first_entity(cfs_rq);
    struct sched_entity *left = se;
    /*
    * Avoid running the skip buddy, if running something else can
    * be done without getting too unfair.
    */
    if (cfs_rq->skip == se) {
        struct sched_entity *second = __pick_next_entity(se);
        if (second && wakeup_preempt_entity(second, left) < 1)
            se = second;
    }
    /*
    * Prefer last buddy, try to return the CPU to a preempted task.
    */
    if (cfs_rq->last && wakeup_preempt_entity(cfs_rq->last, left) < 1)
        se = cfs_rq->last;
    /*
    * Someone really wants this to run. If it's not unfair, run it.
    */
    if (cfs_rq->next && wakeup_preempt_entity(cfs_rq->next, left) < 1)
        se = cfs_rq->next;
    clear_buddies(cfs_rq, se);
    return se;
}
```

`__pick_first_entity()` 从树中挑选出最左边的节点，你可以从下面的代码看出来这是非常简单和快速的：

```
static struct sched_entity *__pick_first_entity(struct cfs_rq *cfs_rq)
{
    struct rb_node *left = cfs_rq->rb_leftmost;
    if (!left)
        return NULL;
    return rb_entry(left, struct sched_entity, run_node);
}
```

