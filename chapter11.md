# 11. Real Time Load Balancing

The active and idle balancing implementation in the CFS class makes sure that only CFS tasks are affected. Real time tasks are handled by the real time scheduler. It applies push and pull operations for overloaded RT queues considering the following cases:

- 1. A task wakes up and needs to be scheduled: This scenario is handled by the select_task_rq() implementation of the RT algorithm.
- 2. A lower prio task wakes up on a runqueue with a higher prio task running: a push operation is applied for the lower prio task.
- 3. A high prio task wakes up on a runqueue with a lower prio task running and pre-empts it: also a push operation is applied for the lower prio task.
- 4. Priorities change on a runqueue because a task lowers its prio and thereby causes a previously lower-prio task to have the higher prio: A pull operation will look for higher prio tasks to pull to this runqueue.

The design goal the real time load balancing pursues is that of a system-wide strict real-time priority scheduling. That means, the real time scheduler needs to make sure that the N highest-priority RT tasks are running at any given point in time, where N is the number of CPUs.

## 11.1. Root Domains and CPU Priorities

The given design goal requires the scheduler to get a quick and efficient overview of all runqueues in the system to make scheduling decisions. This leads to scalability issues with a growing number of CPUs. Therefore, the notion of a root domain was introduced which partitions CPUs into subsets 
that are used by a process or a group of processes. All real-time scheduling decisions are made only within the scope of a root domain.

Another concept that was introduced to deal with the given design goal are CPU priorities. CPU priorities mirror the priority of the highest priority RT task in a root domain. A two-dimensional bitmap is used to manage the CPU priorities. One dimension for the different priorities and one for the CPUs in that priority. CPU priorities are implemented in /kernel/sched_cpupri.c and /kernel/sched_cpupri.h.

A root_domain struct has a bit array for overloaded CPUs it contains and a cpupri struct with a bitmap of CPU priorities. A CPU counts as overloaded with RT task if it has more than one runnable RT task in its runqueue.

```
/*
 * We add the notion of a root-domain which will be used to define per-domain
 * variables. Each exclusive cpuset essentially defines an island domain by
 * fully partitioning the member cpus from any other cpuset. Whenever a new
 * exclusive cpuset is created, we also create and attach a new root-domain
 * object.
 *
 */
struct root_domain {
    atomic_t refcount;
    atomic_t rto_count;
    struct rcu_head rcu;
    cpumask_var_t span;
    cpumask_var_t online;
    /*
     * The "RT overload" flag: it gets set if a CPU has more than
     * one runnable RT task.
     */
    cpumask_var_t rto_mask;
    struct cpupri cpupri;
};
struct cpupri_vec {
    raw_spinlock_t lock;
    int count;
    cpumask_var_t mask;
};
struct cpupri {
    struct cpupri_vec pri_to_cpu[CPUPRI_NR_PRIORITIES];
    long pri_active[CPUPRI_NR_PRI_WORDS];
    int cpu_to_pri[NR_CPUS];
};
```

With the cpupri bitmask above properly maintained, the function cpupri_find() can be used to quickly find a low priority CPU to push a higher priority task to. If a priority level is non-empty and lower than the priority of the task being pushed, the lowest_mask is set to the mask corresponding to the priority level selected. This mask is then used by the push algorithm to compute the best CPU to which to push the task, based on affinity, topology and cache characteristics.

```
/**
 * cpupri_find - find the best (lowest-pri) CPU in the system
 * @cp: The cpupri context
 * @p: The task
 * @lowest_mask: A mask to fill in with selected CPUs (or NULL)
 *
 * Note: This function returns the recommended CPUs as calculated during the
 * current invocation. By the time the call returns, the CPUs may have in
 * fact changed priorities any number of times. While not ideal, it is not
 * an issue of correctness since the normal rebalancer logic will correct
 * any discrepancies created by racing against the uncertainty of the current
 * priority configuration.
 *
 * Returns: (int)bool - CPUs were found
 */
int cpupri_find(struct cpupri *cp, struct task_struct *p,
                struct cpumask *lowest_mask)
{
    int idx = 0;
    int task_pri = convert_prio(p->prio);
    for_each_cpupri_active(cp-> pri_active

                           , idx)

    {
        struct cpupri_vec *vec = &cp->pri_to_cpu[idx];
        if (idx >= task_pri)
            break;
        if (cpumask_any_and(&p->cpus_allowed, vec->mask) >= nr_cpu_ids)
            continue;
        if (lowest_mask) {
            cpumask_and(lowest_mask, &p->cpus_allowed, vec->mask);
            /*
             * We have to ensure that we have at least one bit
             * still set in the array, since the map could have
             * been concurrently emptied between the first and
             * second reads of vec->mask. If we hit this
             * condition, simply act as though we never hit this
             * priority level and continue on.
             */
            if (cpumask_any(lowest_mask) >= nr_cpu_ids)
                continue;
        }
        return 1;

    }
    return 0;

}
```

The function electing a final cpu to push a task to from the lowest mask is called find_lowest_rq(). It is implemented in kernel/sched_rt.c. If cpupri_find() returns a mask with possible CPU targets, find_lowest_rq() would first look if the CPU is among them that last executed the task to be pushed. This CPU is most likely to have the hottest cache and therefore the best choice.

If not, find_lowest_rq() walks up the scheduling domain hierarchy to find a CPU that is logically closest to the hot cache data of the current CPU and also in the lowest prio map. 

If this search also does not return any usable results, just any of the CPUs in the mask is selected and returned.

```
static int find_lowest_rq(struct task_struct *task)
{
    struct sched_domain *sd;
    struct cpumask *lowest_mask = __get_cpu_var(local_cpu_mask);
    int this_cpu = smp_processor_id();
    int cpu = task_cpu(task);
    /* Make sure the mask is initialized first */
    if (unlikely(!lowest_mask))
        return -1;
    if (task->rt.nr_cpus_allowed == 1)
        return -1; /* No other targets possible */
    if (!cpupri_find(&task_rq(task)->rd->cpupri, task, lowest_mask))
        return -1; /* No targets found */
    /*
     * At this point we have built a mask of cpus representing the
     * lowest priority tasks in the system. Now we want to elect
     * the best one based on our affinity and topology.
     *
     * We prioritize the last cpu that the task executed on since
     * it is most likely cache-hot in that location.
     */
    if (cpumask_test_cpu(cpu, lowest_mask))
        return cpu;
    /*
     * Otherwise, we consult the sched_domains span maps to figure
     * out which cpu is logically closest to our hot cache data.
     */
    if (!cpumask_test_cpu(this_cpu, lowest_mask))
        this_cpu = -1; /* Skip this_cpu opt if not among lowest */
    rcu_read_lock();
    for_each_domain(cpu, sd) {
        if (sd->flags & SD_WAKE_AFFINE) {
            int best_cpu;
            /*
             * "this_cpu" is cheaper to preempt than a
             * remote processor.
             */
            if (this_cpu != -1 &&
                cpumask_test_cpu(this_cpu, sched_domain_span(sd))) {
                rcu_read_unlock();
                return this_cpu;
            }
            best_cpu = cpumask_first_and(lowest_mask,
                                         sched_domain_span(sd));
            if (best_cpu < nr_cpu_ids) {
                rcu_read_unlock();
                return best_cpu;
            }
        }
    }
    rcu_read_unlock();
    /*
     * And finally, if there were no matches within the domains
     * just give the caller *something* to work with from the compatible
     * locations.
     */
    if (this_cpu != -1)
        return this_cpu;
    cpu = cpumask_any(lowest_mask);
    if (cpu < nr_cpu_ids)
        return cpu;
    return -1;

}    
```

## 11.2 Push Operation

The task push operation in the RT scheduler is invoked after the scheduler skeleton scheduled a new task on the current CPU. The scheduling class hook post_schedule() is called which is currently only implemented by the RT scheduling class. Internally, it calls push_rt_task() for all tasks on the current CPU's runqueue.

This function looks if the runqueue is overloaded and, if so, tries to move non running RT tasks to another applicable CPU until moving a task fails. If a task was moved, a future call to schedule() is invoked by setting the need_resched flag. 

It uses find_lock_lowest_rq() which internally calls find_lowest_rq() but additionally locks the queue if found.

```
/*
 * If the current CPU has more than one RT task, see if the non
 * running task can migrate over to a CPU that is running a task
 * of lesser priority.
 */
static int push_rt_task(struct rq *rq)
{
    struct task_struct *next_task;
    struct rq *lowest_rq;
    if (!rq->rt.overloaded)
        return 0;
    next_task = pick_next_pushable_task(rq);
    if (!next_task)
        return 0;
retry:
    if (unlikely(next_task == rq->curr)) {
        WARN_ON(1);
        return 0;
    }
    /*
     * It's possible that the next_task slipped in of
     * higher priority than current. If that's the case
     * just reschedule current.
     */
    if (unlikely(next_task->prio < rq->curr->prio)) {
        resched_task(rq->curr);
        return 0;
    }
    /* We might release rq lock */
    get_task_struct(next_task);
    /* find_lock_lowest_rq locks the rq if found */
    lowest_rq = find_lock_lowest_rq(next_task, rq);
    if (!lowest_rq) {
        struct task_struct *task;
        /*
         * find lock_lowest_rq releases rq->lock
         * so it is possible that next_task has migrated.
         *
         * We need to make sure that the task is still on the same
         * run-queue and is also still the next task eligible for
         * pushing.
         */
        task = pick_next_pushable_task(rq);
        if (task_cpu(next_task) == rq->cpu && task == next_task) {
            /*
             * If we get here, the task hasn't moved at all, but
             * it has failed to push. We will not try again,
             * since the other cpus will pull from us when they
             * are ready.
             */
            dequeue_pushable_task(rq, next_task);
            goto out;
        }
        if (!task)
            /* No more tasks, just exit */
            goto out;
        /*
         * Something has shifted, try again.
         */
        put_task_struct(next_task);
        next_task = task;
        goto retry;
    }
    deactivate_task(rq, next_task, 0);
    set_task_cpu(next_task, lowest_rq->cpu);
    activate_task(lowest_rq, next_task, 0);
    resched_task(lowest_rq->curr);
    double_unlock_balance(rq, lowest_rq);
out:
    put_task_struct(next_task);
    return 1;
}         
```

## 11.3 Pull Operation

The pull operation is called by the pre_schedule() hook. It is currently also only implemented by the RT class and invoked at the beginning of the schedule() function. It will check if the highest priority of the tasks in the current CPU's runqueue is smaller than the priority of the task than ran last. If that is the case, it will call pull_rt_task() to pull another RT task with a higher priority than the to-be-scheduled task from a different CPU.

```
static void pre_schedule_rt(struct rq *rq, struct task_struct *prev)
{
    /* Try to pull RT tasks here if we lower this rq's prio */
    if (rq->rt.highest_prio.curr > prev->prio)
        pull_rt_task(rq);
}
```

pull_rt_task() will go through each CPU of the same root domain as the current CPU. It looks for tasks that are the next highest priority task on the potential source CPU to pull them over to the current CPU. If a task is found, it is pulled over. Schedule() is not invoked since it is about to be executed anyway.

```
static int pull_rt_task(struct rq *this_rq)
{
    int this_cpu = this_rq->cpu, ret = 0, cpu;
    struct task_struct *p;
    struct rq *src_rq;
    if (likely(!rt_overloaded(this_rq)))
        return 0;
    for_each_cpu(cpu, this_rq->rd->rto_mask) {
        if (this_cpu == cpu)
            continue;
        src_rq = cpu_rq(cpu);
        /*
         * Don't bother taking the src_rq->lock if the next highest
         * task is known to be lower-priority than our current task.
         * This may look racy, but if this value is about to go
         * logically higher, the src_rq will push this task away.
         * And if its going logically lower, we do not care
         */
        if (src_rq->rt.highest_prio.next >=
            this_rq->rt.highest_prio.curr)
            continue;
        /*
         * We can potentially drop this_rq's lock in
         * double_lock_balance, and another CPU could
         * alter this_rq
         */
        double_lock_balance(this_rq, src_rq);
        /*
         * Are there still pullable RT tasks?
         */
        if (src_rq->rt.rt_nr_running <= 1)
            goto skip;
        p = pick_next_highest_task_rt(src_rq, this_cpu);
        /*
         * Do we have an RT task that preempts
         * the to-be-scheduled task?
         */
        if (p && (p->prio < this_rq->rt.highest_prio.curr)) {
            WARN_ON(p == src_rq->curr);
            WARN_ON(!p->on_rq);
            /*
             * There's a chance that p is higher in priority
             * than what's currently running on its cpu.
             * This is just that p is wakeing up and hasn't
             * had a chance to schedule. We only pull
             * p if it is lower in priority than the
             * current task on the run queue
             */
            if (p->prio < src_rq->curr->prio)
                goto skip;
            ret = 1;
            deactivate_task(src_rq, p, 0);
            set_task_cpu(p, this_cpu);
            activate_task(this_rq, p, 0);
            /*
             * We continue with the search, just in
             * case there's an even higher prio task
             * in another runqueue. (low likelihood
             * but possible)
             */
        }

skip:

        double_unlock_balance(this_rq, src_rq);

    }
    return ret;

}             
```

## 11.4 Select a Runqueue for a Waking Task

As described in the load balancing chapter for CFS tasks, the select_task_rq() scheduling class hook is called as soon as a task wakes up again or for the first time. Apart from the additional push and pull operations, this hook is also implemented by the RT scheduler.

If the currently running task on this CPU's runqueue is a RT task, if its priority is higher and if we can move the task to be woken up, then try to find another runqueue we can wake the task up on. If not, wake it up on the same CPU and let the RT scheduler do the rest.

This function also uses find_lowest_rq() to find a CPU applicable for the task.

```
static int select_task_rq_rt(struct task_struct *p, int sd_flag, int flags)
{
    struct task_struct *curr;
    struct rq *rq;
    int cpu;
    if (sd_flag != SD_BALANCE_WAKE)
        return smp_processor_id();
    cpu = task_cpu(p);
    rq = cpu_rq(cpu);
    rcu_read_lock();
    curr = ACCESS_ONCE(rq->curr); /* unlocked access */
    if (curr && unlikely(rt_task(curr)) &&
        (curr->rt.nr_cpus_allowed < 2 ||
         curr->prio <= p->prio) &&
        (p->rt.nr_cpus_allowed > 1)) {
        int target = find_lowest_rq(p);
        if (target != -1)
            cpu = target;
    }
    rcu_read_unlock();
    return cpu;
}
```

---

# 11. 实时负载均衡

CFS 调度类中实现的积极平衡和空闲平衡能确保只有 CFS 任务受影响。实时任务的负载均衡是由实时调度器处理的。在超载的 RT 队列上执行拉取和推送操作时要考虑下面几种情况：

- 1. 一个任务唤醒和需要被调度： 这种场景下是由实现 RT 调度算法的函数 `select_task_rq()` 处理的。
- 2. 队列中有一个高优先级的任务在运行，然后一个低优先级的任务被唤醒了： 会对这个低优先级任务执行推送操作。
- 3. 队列中有一个低优先级的任务在运行，然后一个高优先级的任务被唤醒了：同样也对低优先级任务执行推送操作。
- 4. 运行队里的优先级发生变化，因为一个任务降低了自己的优先级，从而之前一个更低优先级的任务变成相对高的优先级：会执行推送操作来寻找一个高优先级的任务并推送到这个运行队列。

实时负载均衡这种设计追求的目标是在整个系统中严格的执行实时优先级调度。这也就意味着实时任务调度器需要确保 N 个最高优先级的 RT 任务能够在任何时间点都能及时运行，其中 N 是 CPU 的个数。

## 11.1. 根域和 CPU 优先级

给定的设计目标要求调度器能快速且有效的获取一个关于系统中所有运行队列的综述，以此来做出调度决策。这就引出了随着 CPU 个数增而加造成的扩容问题。因此引入了根域这个概念，它就是把多个 CPU 划分成几个子集，每个子集包括一个或一组处理器。所有的实时任务调度决策都是在单个根域的范围生效的。

另一个为了处理给定的设计目标而引入的概念是 CPU 优先级。CPU 优先级反映的是根域中最高优先级的 RT 任务的优先级。是另一个维度的优先级，和处于这个优先级的一个 CPU。CPU 优先级的实现在 `/kernel/sched_cpupri.c` 和 `/kernel/sched_cpupri.h` 中。

每个 `root_domain` 结构体都有一个位数组（bit array）记录它对应的根域中有多少超载的 CPU ， `cpupri` 结构体含有一个 CPU 优先级位图（bitmap）。如果一个 CPU 的运行队列上有不止一个可运行的 RT 任务则它被认为是超载的。

```
/*
 * We add the notion of a root-domain which will be used to define per-domain
 * variables. Each exclusive cpuset essentially defines an island domain by
 * fully partitioning the member cpus from any other cpuset. Whenever a new
 * exclusive cpuset is created, we also create and attach a new root-domain
 * object.
 *
 */
struct root_domain {
    atomic_t refcount;
    atomic_t rto_count;
    struct rcu_head rcu;
    cpumask_var_t span;
    cpumask_var_t online;
    /*
     * The "RT overload" flag: it gets set if a CPU has more than
     * one runnable RT task.
     */
    cpumask_var_t rto_mask;
    struct cpupri cpupri;
};
struct cpupri_vec {
    raw_spinlock_t lock;
    int count;
    cpumask_var_t mask;
};
struct cpupri {
    struct cpupri_vec pri_to_cpu[CPUPRI_NR_PRIORITIES];
    long pri_active[CPUPRI_NR_PRI_WORDS];
    int cpu_to_pri[NR_CPUS];
};
```

函数 `cpupri_find()` 可以通过上面提到的 `cpupri` 的位图快速的找出一个可以将高优先级任务推送过去的低优先级的 CPU 。如果一个优先级等级非空，而且比这个被推送的任务优先级还要低，则标志 `lowest_mask` 会设置已经选择了的等级对应的掩码。这个掩码是被推送算法用来寻找最合适推送任务的 CPU，算法是基于亲和性，拓扑结构和缓存特性进行计算的。

```
/**
 * cpupri_find - find the best (lowest-pri) CPU in the system
 * @cp: The cpupri context
 * @p: The task
 * @lowest_mask: A mask to fill in with selected CPUs (or NULL)
 *
 * Note: This function returns the recommended CPUs as calculated during the
 * current invocation. By the time the call returns, the CPUs may have in
 * fact changed priorities any number of times. While not ideal, it is not
 * an issue of correctness since the normal rebalancer logic will correct
 * any discrepancies created by racing against the uncertainty of the current
 * priority configuration.
 *
 * Returns: (int)bool - CPUs were found
 */
int cpupri_find(struct cpupri *cp, struct task_struct *p,
                struct cpumask *lowest_mask)
{
    int idx = 0;
    int task_pri = convert_prio(p->prio);
    for_each_cpupri_active(cp-> pri_active, idx)
    {
        struct cpupri_vec *vec = &cp->pri_to_cpu[idx];
        if (idx >= task_pri)
            break;
        if (cpumask_any_and(&p->cpus_allowed, vec->mask) >= nr_cpu_ids)
            continue;
        if (lowest_mask) {
            cpumask_and(lowest_mask, &p->cpus_allowed, vec->mask);
            /*
             * We have to ensure that we have at least one bit
             * still set in the array, since the map could have
             * been concurrently emptied between the first and
             * second reads of vec->mask. If we hit this
             * condition, simply act as though we never hit this
             * priority level and continue on.
             */
            if (cpumask_any(lowest_mask) >= nr_cpu_ids)
                continue;
        }
        return 1;

    }
    return 0;

}
```

`find_lowest_rq()` 是用来挑选出最终要被推送任务的处理器的函数，而这个任务对应的是最低的掩码。它的实现位于 `kernel/sched_rt.c` 。如果 `cpupri_find()` 返回一个包含可能的目标 CPU 的掩码，`find_lowest_rq()` 将会首先检查这个 CPU 是否属于推送上次执行的任务的目标 CPU 。这个 CPU 通常有最热门的缓存（hottest cache），因此也是最佳选择。

如果不是的话，`find_lowest_rq()` 会遍历整个调度域层级来找出一个逻辑上最接近当前 CPU 的热门缓存数据，同时也位于最低优先级映射中的 CPU。

如果这次搜索同样没有返回任何可用结果，那么就返回掩码对应的任意一个 CPU 。

```
static int find_lowest_rq(struct task_struct *task)
{
    struct sched_domain *sd;
    struct cpumask *lowest_mask = __get_cpu_var(local_cpu_mask);
    int this_cpu = smp_processor_id();
    int cpu = task_cpu(task);
    /* Make sure the mask is initialized first */
    if (unlikely(!lowest_mask))
        return -1;
    if (task->rt.nr_cpus_allowed == 1)
        return -1; /* No other targets possible */
    if (!cpupri_find(&task_rq(task)->rd->cpupri, task, lowest_mask))
        return -1; /* No targets found */
    /*
     * At this point we have built a mask of cpus representing the
     * lowest priority tasks in the system. Now we want to elect
     * the best one based on our affinity and topology.
     *
     * We prioritize the last cpu that the task executed on since
     * it is most likely cache-hot in that location.
     */
    if (cpumask_test_cpu(cpu, lowest_mask))
        return cpu;
    /*
     * Otherwise, we consult the sched_domains span maps to figure
     * out which cpu is logically closest to our hot cache data.
     */
    if (!cpumask_test_cpu(this_cpu, lowest_mask))
        this_cpu = -1; /* Skip this_cpu opt if not among lowest */
    rcu_read_lock();
    for_each_domain(cpu, sd) {
        if (sd->flags & SD_WAKE_AFFINE) {
            int best_cpu;
            /*
             * "this_cpu" is cheaper to preempt than a
             * remote processor.
             */
            if (this_cpu != -1 &&
                cpumask_test_cpu(this_cpu, sched_domain_span(sd))) {
                rcu_read_unlock();
                return this_cpu;
            }
            best_cpu = cpumask_first_and(lowest_mask,
                                         sched_domain_span(sd));
            if (best_cpu < nr_cpu_ids) {
                rcu_read_unlock();
                return best_cpu;
            }
        }
    }
    rcu_read_unlock();
    /*
     * And finally, if there were no matches within the domains
     * just give the caller *something* to work with from the compatible
     * locations.
     */
    if (this_cpu != -1)
        return this_cpu;
    cpu = cpumask_any(lowest_mask);
    if (cpu < nr_cpu_ids)
        return cpu;
    return -1;

}    
```

## 11.2. 推送操作

RT 调度器中的任务推送操作是在调度框架调度新任务到当前 CPU 时执行的。调度类的钩子 `post_schedule()` 会被调用，目前它只在 RT 调度类中实现了。在函数内部，它会为当前 CPU 的运行队列中所有的任务调用 `push_rt_task()` 。

这个函数会检查运行队列是否超载，如果超载了就尝试将不是正在运行的 RT 任务迁移到另一个可用的 CPU 直到迁移失败。如果一个任务被迁移了，则之后都会通过设置标志 `need_resched` 来调用 `schedule()`。

内核会使用内部调用了 `find_lowest_rq()` 的 `find_lock_lowest_rq()` ，但是如果发现了队列就会对这个队列加锁。

```
/*
 * If the current CPU has more than one RT task, see if the non
 * running task can migrate over to a CPU that is running a task
 * of lesser priority.
 */
static int push_rt_task(struct rq *rq)
{
    struct task_struct *next_task;
    struct rq *lowest_rq;
    if (!rq->rt.overloaded)
        return 0;
    next_task = pick_next_pushable_task(rq);
    if (!next_task)
        return 0;
retry:
    if (unlikely(next_task == rq->curr)) {
        WARN_ON(1);
        return 0;
    }
    /*
     * It's possible that the next_task slipped in of
     * higher priority than current. If that's the case
     * just reschedule current.
     */
    if (unlikely(next_task->prio < rq->curr->prio)) {
        resched_task(rq->curr);
        return 0;
    }
    /* We might release rq lock */
    get_task_struct(next_task);
    /* find_lock_lowest_rq locks the rq if found */
    lowest_rq = find_lock_lowest_rq(next_task, rq);
    if (!lowest_rq) {
        struct task_struct *task;
        /*
         * find lock_lowest_rq releases rq->lock
         * so it is possible that next_task has migrated.
         *
         * We need to make sure that the task is still on the same
         * run-queue and is also still the next task eligible for
         * pushing.
         */
        task = pick_next_pushable_task(rq);
        if (task_cpu(next_task) == rq->cpu && task == next_task) {
            /*
             * If we get here, the task hasn't moved at all, but
             * it has failed to push. We will not try again,
             * since the other cpus will pull from us when they
             * are ready.
             */
            dequeue_pushable_task(rq, next_task);
            goto out;
        }
        if (!task)
            /* No more tasks, just exit */
            goto out;
        /*
         * Something has shifted, try again.
         */
        put_task_struct(next_task);
        next_task = task;
        goto retry;
    }
    deactivate_task(rq, next_task, 0);
    set_task_cpu(next_task, lowest_rq->cpu);
    activate_task(lowest_rq, next_task, 0);
    resched_task(lowest_rq->curr);
    double_unlock_balance(rq, lowest_rq);
out:
    put_task_struct(next_task);
    return 1;
}         
```

## 11.3. 拉取操作

拉取操作是通过调用钩子 `pre_schedule()` 执行的。这个函数目前也只是在 RT 调度类里面实现了，并且在函数 `schedule()` 的开始被调用。它会去检查当前 CPU 的运行队列中的任务的最高优先级是否小于上次运行的任务的优先级。如果是的话，内核就会调用 `pull_rt_task()` 从另一个 CPU 上拉取一个比（当前 CPU 上）将被调度的任务优先级高的任务。

```
static void pre_schedule_rt(struct rq *rq, struct task_struct *prev)
{
    /* Try to pull RT tasks here if we lower this rq's prio */
    if (rq->rt.highest_prio.curr > prev->prio)
        pull_rt_task(rq);
}
```

`pull_rt_task()` 会检查和当前 CPU 处于同一个根域的所有 CPU。它会在潜在的源 CPU 上的寻找第二高优先级任务，然后拉取到当前的 CPU。如果找到一个任务就拉过来。 因为 `Schedule()` 会在很多情况下被执行，所以此处没有调用。

```
static int pull_rt_task(struct rq *this_rq)
{
    int this_cpu = this_rq->cpu, ret = 0, cpu;
    struct task_struct *p;
    struct rq *src_rq;
    if (likely(!rt_overloaded(this_rq)))
        return 0;
    for_each_cpu(cpu, this_rq->rd->rto_mask) {
        if (this_cpu == cpu)
            continue;
        src_rq = cpu_rq(cpu);
        /*
         * Don't bother taking the src_rq->lock if the next highest
         * task is known to be lower-priority than our current task.
         * This may look racy, but if this value is about to go
         * logically higher, the src_rq will push this task away.
         * And if its going logically lower, we do not care
         */
        if (src_rq->rt.highest_prio.next >=
            this_rq->rt.highest_prio.curr)
            continue;
        /*
         * We can potentially drop this_rq's lock in
         * double_lock_balance, and another CPU could
         * alter this_rq
         */
        double_lock_balance(this_rq, src_rq);
        /*
         * Are there still pullable RT tasks?
         */
        if (src_rq->rt.rt_nr_running <= 1)
            goto skip;
        p = pick_next_highest_task_rt(src_rq, this_cpu);
        /*
         * Do we have an RT task that preempts
         * the to-be-scheduled task?
         */
        if (p && (p->prio < this_rq->rt.highest_prio.curr)) {
            WARN_ON(p == src_rq->curr);
            WARN_ON(!p->on_rq);
            /*
             * There's a chance that p is higher in priority
             * than what's currently running on its cpu.
             * This is just that p is wakeing up and hasn't
             * had a chance to schedule. We only pull
             * p if it is lower in priority than the
             * current task on the run queue
             */
            if (p->prio < src_rq->curr->prio)
                goto skip;
            ret = 1;
            deactivate_task(src_rq, p, 0);
            set_task_cpu(p, this_cpu);
            activate_task(this_rq, p, 0);
            /*
             * We continue with the search, just in
             * case there's an even higher prio task
             * in another runqueue. (low likelihood
             * but possible)
             */
        }

skip:

        double_unlock_balance(this_rq, src_rq);

    }
    return ret;

}             
```

## 11.4. 为唤醒的任务挑选一个运行队列

如之前在介绍 CFS 任务负载均衡的章节中描述的，一旦一个任务再次被唤醒或者是首次被创建内核就会立即调用 `select_task_rq()`。除了额外的推送和拉取操作，这个钩子函数也在 RT 调度器中实现了。

如果当前 CPU 的运行队列中正在运行的是 RT 任务，加入他的优先级更高，并且我们可以移动一个将要被唤醒的任务，然后尝试寻找另一个运行队列可以安置我们唤醒的任务。如果不是的话，就在同一个 CPU 上唤醒任务，然后让调度器完成剩余的工作。

这个函数同时也使用 `find_lowest_rq()` 寻找一个适合这个任务的 CPU。

```
static int select_task_rq_rt(struct task_struct *p, int sd_flag, int flags)
{
    struct task_struct *curr;
    struct rq *rq;
    int cpu;
    if (sd_flag != SD_BALANCE_WAKE)
        return smp_processor_id();
    cpu = task_cpu(p);
    rq = cpu_rq(cpu);
    rcu_read_lock();
    curr = ACCESS_ONCE(rq->curr); /* unlocked access */
    if (curr && unlikely(rt_task(curr)) &&
        (curr->rt.nr_cpus_allowed < 2 ||
         curr->prio <= p->prio) &&
        (p->rt.nr_cpus_allowed > 1)) {
        int target = find_lowest_rq(p);
        if (target != -1)
            cpu = target;
    }
    rcu_read_unlock();
    return cpu;
}
```
