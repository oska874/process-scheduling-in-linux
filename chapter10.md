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
