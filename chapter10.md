# 10. Load Balancing on SMP Systems

Load balancing was introduced with the main goal of improving the performance of SMP systems
by offloading tasks from busy CPUs to less busy or idle ones. The Linux scheduler checks regularly how the task load is spread throughout the system and performs load balancing if necessary.

The complexity lies in serving a variety of SMP system topologies. There are systems with multiple physical cores where tasks might suffer more from a cache flush due to being moved to a different CPU than from being scheduled on a more busy CPU. Other systems might support hyper threading with shared caches where this scenario is more flexible towards task migration due to shared caches. NUMA architectures create situations where different nodes have different access speeds to different areas of main memory.

In order to tackle this topology variety, scheduling domains were introduced in the 2.6 Linux kernel. They are a way of hierarchically grouping all available CPUs in the system which gives the kernel a way of characterising the underlying core topology.

## 10.1. Scheduling Domains and Groups

A scheduling domain is a set of CPUs which share properties and scheduling policies, and which can be balanced against each other.

Each domain can contain one ore more scheduling groups which are treated as a single unit by the domain. When the scheduler tries to balance the load within a domain, it tries to even out the load carried by each scheduling group without worrying directly about what is happening within the group.

![](2016-08-04-001245_1178x602_scrot.png)

Imagine a system with two physical CPUs which are both hyperthreaded which gives us in total
four logical CPUs. If the system starts up, the kernel would divide the logical cores into the two level domain hierarchy you can see in the picture.

Each hyperthreaded processor is put into exactly one group while either two are in a single domain.These two domains are then again put into a top level domain which holds the complete processor.

It contains two groups which both have two hyperthreaded cores each.

If this were a NUMA system, it would have multiple domains which look like the above diagram;each of those domains would represent one NUMA node. The hierarchy would have a third, systemlevel domain which contains all of the NUMA nodes.

## 10.2 Load Balancing

Each scheduling domain has a balancing policy set which is valid on that level of the hierarchy. The policy parameters include how often attempts should be made to balance loads across the domain, how far the loads on the component processors are allowed to get out of sync before a balancing attempt is made, how long a process can sit idle before it is considered to no longer have any significant cache affinity.

On top of that, various policy flags specify the behaviour for certain situations, such as: A CPU goes idle; should the balancing code look for a task to pull? A task wakes up or is spawned; which CPU should it be scheduled on?

An active load balancing is executed regularly which moves up the scheduling domain hierarchy and checks all groups along the way if they got out of balance. If so it does a balancing attempt considering the policy rules of the corresponding domain.

## 10.3 Implementation Details

### Data Structures

Two data structures were added to include/linux/sched.h to divide the cores into a balancing hierarchy. sched_domain represents a scheduling domain and sched_group a scheduling group:

```
struct sched_domain {
    /* These fields must be setup */
    struct sched_domain *parent; /* top domain must be null terminated */
    struct sched_domain *child; /* bottom domain must be null terminated */
    struct sched_group *groups; /* the balancing groups of the domain */
    unsigned long min_interval; /* Minimum balance interval ms */
    unsigned long max_interval; /* Maximum balance interval ms */
    unsigned int busy_factor; /* less balancing by factor if busy */
    unsigned int imbalance_pct; /* No balance until over watermark */
    unsigned int cache_nice_tries; /* Leave cache hot tasks for # tries */
    unsigned int busy_idx;
    unsigned int idle_idx;
    unsigned int newidle_idx;
    unsigned int wake_idx;
    unsigned int forkexec_idx;
    unsigned int smt_gain;
    int flags; /* See SD_* */
    int level;
    /* Runtime fields. */
    unsigned long last_balance; /* init to jiffies. units in jiffies */
    unsigned int balance_interval; /* initialise to 1. units in ms. */
    unsigned int nr_balance_failed; /* initialise to 0 */
    u64 last_update;
    ...
    unsigned int span_weight;
    /*
    * Span of all CPUs in this domain.
    *
    * NOTE: this field is variable length. (Allocated dynamically
    * by attaching extra space to the end of the structure,
    * depending on how many CPUs the kernel has booted up with)
    */
    unsigned long span[0];
};
struct sched_group {
    struct sched_group *next; /* Must be a circular list */
    atomic_t ref;
    unsigned int group_weight;
    struct sched_group_power *sgp;
    /*
    * The CPUs this group covers.
    *
    * NOTE: this field is variable length. (Allocated dynamically
    * by attaching extra space to the end of the structure,
    * depending on how many CPUs the kernel has booted up with)
    */
    unsigned long cpumask[0];
};
```

In include/linux/topology.h or the corresponding architecture version, you can see how flags and values for the scheduling domain policy are set.

In sched_group you can see a field called sgp which is of the type sched_group_power. The concept of total CPU power of a group was introduced to further specify the topology of a processor. Even though a hyperthreaded core appears as an independent unit, it has in reality significantly less processing power than a second physical core. Two separate processors would have a CPU power of two, while a hyperthreaded pair would have something closer to 1.1. During load balancing, the kernel tries to maximise the CPU power value to increase the overall throughput of the system.

### Active Balancing

Active balancing is performed regularly on each CPU. During active balancing, the kernel walks up the domain hierarchy, starting at the current CPU's domain, and checks each scheduling domain to see if it is due to be balanced, and initiates a balancing operation if so.

During scheduler initialisation, a soft irq handler is registered which performs the regular load balancing. It is triggered in scheduler_tick() with a call to trigger_load_balance() (see Scheduler Skeleton). trigger_load_balance() checks a timer and if balancing is due, it fires the soft irq with the corresponding flag SCHED_SOFTIRQ.

```
/*
* Trigger the SCHED_SOFTIRQ if it is time to do periodic load balancing.
*/
static inline void trigger_load_balance(struct rq *rq, int cpu)
{
    /* Don't need to rebalance while attached to NULL domain */
    if (time_after_eq(jiffies, rq->next_balance) &&
        likely(!on_null_domain(cpu)))
        raise_softirq(SCHED_SOFTIRQ);
#ifdef CONFIG_NO_HZ
    else if (nohz_kick_needed(rq, cpu) && likely(!on_null_domain(cpu)))
        nohz_balancer_kick(cpu);
#endif
}
```

The function registered as irq handler is run_rebalance_domains() which calls rebalance_domains() to perform the actual work.

```
/*
* run_rebalance_domains is triggered when needed from the scheduler tick.
* Also triggered for nohz idle balancing (with nohz_balancing_kick set).
*/
static void run_rebalance_domains(struct softirq_action *h)
{
    int this_cpu = smp_processor_id();
    struct rq *this_rq = cpu_rq(this_cpu);
    enum cpu_idle_type idle = this_rq->idle_at_tick ?
                              CPU_IDLE : CPU_NOT_IDLE;
    rebalance_domains(this_cpu, idle);
    /*
    * If this cpu has a pending nohz_balance_kick, then do the
    * balancing on behalf of the other idle cpus whose ticks are
    * stopped.
    */
    nohz_idle_balance(this_cpu, idle);
}
```

rebalance_domains() then walks up the domain hierarchy and calls load_balance() if the domain has the SD_LOAD_BALANCE flag set and its balancing interval is expired. The balancing interval of a domain is in jiffies and updated after each balancing run.

Note that active balancing is a pull operation for the executing CPU. It would pull a task from an overloaded CPU to the current one to rebalance tasks but it would not push one of. The function executing this pull operation is load_balance(). If it is able to find an imbalanced group, it moves one or more tasks over to the current CPU and returns a value larger than 0.

```
/*
 * It checks each scheduling domain to see if it is due to be balanced,
 * and initiates a balancing operation if so.
 *
 * Balancing parameters are set up in arch_init_sched_domains.
 */
static void rebalance_domains(int cpu, enum cpu_idle_type idle)
{
    int balance = 1;
    struct rq *rq = cpu_rq(cpu);
    unsigned long interval;
    struct sched_domain *sd;
    /* Earliest time when we have to do rebalance again */
    unsigned long next_balance = jiffies + 60*HZ;
    int update_next_balance = 0;
    int need_serialize;
    update_shares(cpu);
    rcu_read_lock();
    for_each_domain(cpu, sd) {
        if (!(sd->flags & SD_LOAD_BALANCE))
            continue;
        interval = sd->balance_interval;
        if (idle != CPU_IDLE)
            interval *= sd->busy_factor;
        /* scale ms to jiffies */
        interval = msecs_to_jiffies(interval);
        interval = clamp(interval, 1UL, max_load_balance_interval);
        need_serialize = sd->flags & SD_SERIALIZE;
        if (need_serialize) {
            if (!spin_trylock(&balancing))
                goto out;
        }
        if (time_after_eq(jiffies, sd->last_balance + interval)) {
            if (load_balance(cpu, rq, sd, idle, &balance)) {
                /*
                 * We've pulled tasks over so either we're no
                 * longer idle.
                 */
                idle = CPU_NOT_IDLE;
            }
            sd->last_balance = jiffies;
        }
        if (need_serialize)
            spin_unlock(&balancing);
out:
        if (time_after(next_balance, sd->last_balance + interval)) {
            next_balance = sd->last_balance + interval;
            update_next_balance = 1;
        }
        /*
         * Stop the load balance at this level. There is another
         * CPU in our sched group which is doing load balancing more
         * actively.
         */
        if (!balance)
            break;
    }
    rcu_read_unlock();
    /*
     * next_balance will be updated only when there is a need.
     * When the cpu is attached to null domain for ex, it will not be
     * updated.
     */
    if (likely(update_next_balance))
        rq->next_balance = next_balance;
}        
```

load_balance() calls find_busiest_group() which looks for an imbalance in the given sched_domain and returns the busiest group if it finds one. If the system is in balance and no group is found, load_balance() returns.

If a group was returned, it is passed on to find_busiest_queue() which returns the runqueue of the busiest logical CPU in that group.

load_balance() then searches the resulting runqueue for tasks to swap over to the current CPU's queue by calling move_tasks(). The imbalance parameter, which was set in find_busiest_group(), specifies the amount of tasks that should be moved. It can happen that all tasks on the queue are pinned to it due to cache affinity. In that case, load_balance() searches again but excludes the previously found CPU.

```
/*
 * Check this_cpu to ensure it is balanced within domain. Attempt to move
 * tasks if there is an imbalance.
 */
static int load_balance(int this_cpu, struct rq *this_rq,
                        struct sched_domain *sd, enum cpu_idle_type idle,
                        int *balance)
{
    int ld_moved, all_pinned = 0, active_balance = 0;
    struct sched_group *group;
    unsigned long imbalance;
    struct rq *busiest;
    unsigned long flags;
    struct cpumask *cpus = __get_cpu_var(load_balance_tmpmask);
    cpumask_copy(cpus, cpu_active_mask);
    schedstat_inc(sd, lb_count[idle]);
redo:
    group = find_busiest_group(sd, this_cpu, &imbalance, idle,
                               cpus, balance);
    if (*balance == 0)
        goto out_balanced;
    if (!group) {
        schedstat_inc(sd, lb_nobusyg[idle]);
        goto out_balanced;
    }
    busiest = find_busiest_queue(sd, group, idle, imbalance, cpus);
    if (!busiest) {
        schedstat_inc(sd, lb_nobusyq[idle]);
        goto out_balanced;
    }
    BUG_ON(busiest == this_rq);
    schedstat_add(sd, lb_imbalance[idle], imbalance);
    ld_moved = 0;
    if (busiest->nr_running > 1) {
        /*
         * Attempt to move tasks. If find_busiest_group has found
         * an imbalance but busiest->nr_running <= 1, the group is
         * still unbalanced. ld_moved simply stays zero, so it is
         * correctly treated as an imbalance.
         */
        all_pinned = 1;
        local_irq_save(flags);
        double_rq_lock(this_rq, busiest);
        ld_moved = move_tasks(this_rq, this_cpu, busiest,
                              imbalance, sd, idle, &all_pinned);
        double_rq_unlock(this_rq, busiest);
        local_irq_restore(flags);
        /*
         * some other cpu did the load balance for us.
         */
        if (ld_moved && this_cpu != smp_processor_id())
            resched_cpu(this_cpu);
        /* All tasks on this runqueue were pinned by CPU affinity */
        if (unlikely(all_pinned)) {
            cpumask_clear_cpu(cpu_of(busiest), cpus);
            if (!cpumask_empty(cpus))
                goto redo;
            goto out_balanced;
        }
    }
    ...
    goto out;
out_balanced:
    schedstat_inc(sd, lb_balanced[idle]);
    sd->nr_balance_failed = 0;
out_one_pinned:
    /* tune up the balancing interval */
    if ((all_pinned && sd->balance_interval < MAX_PINNED_INTERVAL) ||
        (sd->balance_interval < sd->max_interval))
        sd->balance_interval *= 2;
    ld_moved = 0;
out:
    return ld_moved;
}        
```

An energy saving related tweak is hidden in find_busiest_group(). If the SD_POWERSAVINGS_BALANCE flag is set in the domains policy and no busiest group is found, find_busiest_group() would look for the least loaded group in the sched_domain, so that it's CPUs can be put to idle. This feature, however, is not activated in the Android kernel and removed from the Ubuntu one.

### Idle Balancing

Idle balancing is invoked as soon as a CPU goes idle. Therefore, it is called by schedule() for the CPU executing the current scheduling thread if its runqueue becomes empty (see Scheduler Sekeleton).

Like active balancing, idle_balance() is implemented in sched_fair.c. First, it checks if the average idle period of the idle runqueue is larger than the cost of migrating a task over to it, that means it checks if it is worth getting a task from somewhere else or if it is better to just wait since the next task is likely to wake up soon anyway.

If migrating a task makes sense, idle_balance() pretty much works like rebalance_domains(). It walks up the domain hierarchy and calls idle_balance() for domains that have the SD_LOAD_BALANCE and the idle balance specific SD_BALANCE_NEWIDLE flag set.

If one or more tasks were pulled over, the hierarchy walk is terminated and idle_balance() returns.

```
/*
 * idle_balance is called by schedule() if this_cpu is about to become
 * idle. Attempts to pull tasks from other CPUs.
 */
static void idle_balance(int this_cpu, struct rq *this_rq)
{
    struct sched_domain *sd;
    int pulled_task = 0;
    unsigned long next_balance = jiffies + HZ;
    this_rq->idle_stamp = this_rq->clock;
    if (this_rq->avg_idle < sysctl_sched_migration_cost)
        return;
    /*
     * Drop the rq->lock, but keep IRQ/preempt disabled.
     */
    raw_spin_unlock(&this_rq->lock);
    update_shares(this_cpu);
    rcu_read_lock();
    for_each_domain(this_cpu, sd) {
        unsigned long interval;
        int balance = 1;
        if (!(sd->flags & SD_LOAD_BALANCE))
            continue;
        if (sd->flags & SD_BALANCE_NEWIDLE) {
            /* If we've pulled tasks over stop searching: */
            pulled_task = load_balance(this_cpu, this_rq,
                                       sd, CPU_NEWLY_IDLE, &balance);
        }
        interval = msecs_to_jiffies(sd->balance_interval);
        if (time_after(next_balance, sd->last_balance + interval))
            next_balance = sd->last_balance + interval;
        if (pulled_task) {
            this_rq->idle_stamp = 0;
            break;
        }
    }
    rcu_read_unlock();
    raw_spin_lock(&this_rq->lock);
    if (pulled_task || time_after(jiffies, this_rq->next_balance)) {
        /*
         * We are going idle. next_balance may be set based on
         * a busy processor. So reset next_balance.
         */
        this_rq->next_balance = next_balance;
    }
}

```

### Selecting a runqueue for a new task

A third spot where balancing decisions need to be made is when a task wakes up or is created and needs to be placed on a runqueue. This runqueue should be selected considering the overall task balance of the system.

Each scheduling class implements its own strategy to handle their tasks and provides a hook (select_task_rq()) that can be called by the scheduler skeleton (kernel/sched.c) to execute it. It is called for three different occasions which are each marked by a corresponding domain flag.

- 1. SD_BALANCE_EXEC flag is used in sched_exec(). This function is called if a task starts a new one by using the exec() system call. A new task at this point has a small memory and cache footprint which gives the kernel a good balancing opportunity.
- 2. SD_BALANCE_FORK flag is used in wake_up_new_task(). This function is called if a newly created task is woken up for the first time.
- 3. SD_BALANCE_WAKE flag is used in try_to_wake_up(). If a task that was running before wakes up it usually has some kind of cache affinity which needs to be considered while selecting a good queue to schedule it on.

---



