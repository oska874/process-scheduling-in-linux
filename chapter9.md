# 9. Soft Real-Time Scheduling

The Linux scheduler supports soft real-time (RT) scheduling. This means that is can effectively schedule tasks that have strict timing requirements. However, while the kernel is usually capable of meeting very strict RT scheduling deadlines, it does not guarantee that deadlines will be met.

The corresponding scheduling class is rt_sched_class which is implemented in kernel/sched_rt.c. RT tasks have priority over CFS tasks.

## 9.1. Scheduling Modes

Tasks handled by the RT scheduler can be configured in two different scheduling modes:

- SCHED_FIFO: A scheduled FIFO task has no time slice and will be scheduled until it terminates,yields the processor voluntarily or a higher priority task becomes runnable.
- SCHED_RR: An RR task will be scheduled for a fixed time slice and then pre-empted in a round robin fashion by tasks with the same priority. That means, as soon as the task's time slice is over, it is set to the end of the queue and its slice is refilled. It can also be pre-empted by tasks with higher priority before the time slice is over.
-
## 9.2. Priorities

With its priority implementation, the RT class follows the same concept, the previous O(1) scheduler did. It uses multiple runqueues where one is reserved for each priority. That way,operations like adding, removing or finding task with highest priority can be achieved in O(1) time.

## 9.3 Real Time Throttling

Occasionally, you can see throttling or bandwith operations in the RT scheduler implementation.These were added to add some safety to SCHED_FIFO tasks. They assign an RT task group with FIFO tasks a certain bandwith for a processor (95% by default) before they are pre-empted if they want to or not. This is supposed to add more security to the kernel against blocking FIFO tasks. It, however, started a discussion among kernel developers and its future is not entirely set.

## 9.4. Implementation Details

### Data Structures

Like CFS, the RT scheduler has its own scheduling entity and runqueue data structure which are added as members to task_struct and the main runqueue.
sched_rt_entity is implemented in /include/linux/sched.h. It has fields for time slice accounting, a pointer to the priority list it belongs to and group scheduling related members.

```
struct sched_rt_entity {
    struct list_head run_list;
    unsigned long timeout;
    unsigned int time_slice;
    int nr_cpus_allowed;
    struct sched_rt_entity *back;
#ifdef CONFIG_RT_GROUP_SCHED
    struct sched_rt_entity*parent;
    /* rq on which this entity is (to be) queued: */
    struct rt_rq *rt_rq;
    /* rq "owned" by this entity/group: */
    struct rt_rq *my_q;
#endif
};
```

rt_rq is implemented in kernel/sched.c. The first field holds the priority arrays. Almost all other fields are for SMP and group scheduling.

```
/* Real-Time classes' related field in a runqueue: */
struct rt_rq {
    struct rt_prio_array active;
    unsigned long rt_nr_running;
#if defined CONFIG_SMP || defined CONFIG_RT_GROUP_SCHED
    struct {
        int curr; /* highest queued rt task prio */
#ifdef CONFIG_SMP
        int next; /* next highest */
#endif
    } highest_prio;
#endif
#ifdef CONFIG_SMP
    unsigned long rt_nr_migratory;
    unsigned long rt_nr_total;
    int overloaded;
    struct plist_head pushable_tasks;
#endif
    int rt_throttled;
    u64 rt_time;
    u64 rt_runtime;
    /* Nests inside the rq lock: */
    raw_spinlock_t rt_runtime_lock;
#ifdef CONFIG_RT_GROUP_SCHED
    unsigned long rt_nr_boosted;
    struct rq *rq;
    struct list_head leaf_rt_rq_list;
    struct task_group *tg;
#endif
};
```

### Time Accounting

scheduler_tick() calls task_tick_rt() to update the current task's time slice. It is pretty straight forward: In the beginning runtime statistics of the current task and its runqueue are updated in update_curr_rt(). Then the function returns if current is a FIFO task.

If not, the RR task's timeslice is reduced by one. If it reaches 0, it is set back to a default value and put back to the end of its runqueue if other tasks are available in the queue. Additionally the need_resched flag is set.

```
static void task_tick_rt(struct rq *rq, struct task_struct *p, int queued)
{
    update_curr_rt(rq);
    watchdog(rq, p);
    /*
    * RR tasks need a special form of timeslice management.
    * FIFO tasks have no timeslices.
    */
    if (p->policy != SCHED_RR)
        return;
    if (--p->rt.time_slice)
        return;
    p->rt.time_slice = DEF_TIMESLICE;
    /*
    * Requeue to the end of queue if we are not the only element
    * on the queue:
    */
    if (p->rt.run_list.prev != p->rt.run_list.next) {
        requeue_task_rt(rq, p, 0);
        set_tsk_need_resched(p);
    }
}
```

### Picking the Next Task

As said earlier, picking the next task to run can be done with constant time complexity. It starts in pick_next_task_rt() which immediately calls _pick_next_task_rt() to do the actual work.

```
static struct task_struct *pick_next_task_rt(struct rq *rq)
{
    struct task_struct *p = _pick_next_task_rt(rq);
    /* The running task is never eligible for pushing */
    if (p)
        dequeue_pushable_task(rq, p);
#ifdef CONFIG_SMP
    /*
    * We detect this state here so that we can avoid taking the RQ
    * lock again later if there is no need to push
    */
    rq->post_schedule = has_pushable_tasks(rq);
#endif
    return p;
}
```

If no tasks are runnable, NULL is returned and a different scheduling class will be searched for tasks. pick_next_rt_entity() gets the next task with the highest priority from the runqueue.

```
static struct task_struct *_pick_next_task_rt(struct rq *rq)
{
    struct sched_rt_entity *rt_se;
    struct task_struct *p;
    struct rt_rq *rt_rq;
    rt_rq = &rq->rt;
    if (!rt_rq->rt_nr_running)
        return NULL;
    if (rt_rq_throttled(rt_rq))
        return NULL;
    do {
        rt_se = pick_next_rt_entity(rq, rt_rq);
        BUG_ON(!rt_se);
        rt_rq = group_rt_rq(rt_se);
    } while (rt_rq);
    p = rt_task_of(rt_se);
    p->se.exec_start = rq->clock_task;
    return p;
}
```

In pick_next_rt_entity() you can see how a bitmap is used for all priority levels to quickly find the highest priority queue with a runnable task. The priority queue itself is a simple linked list and the next runnable task can be found at its head.

```
static struct sched_rt_entity *pick_next_rt_entity(struct rq *rq,
        struct rt_rq *rt_rq)
{
    struct rt_prio_array *array = &rt_rq->active;
    struct sched_rt_entity *next = NULL;
    struct list_head *queue;
    int idx;
    idx = sched_find_first_bit(array->bitmap);
    BUG_ON(idx >= MAX_RT_PRIO);
    queue = array->queue + idx;
    next = list_entry(queue->next, struct sched_rt_entity, run_list);
    return next;
}
```

Adding and removing a task from the priority queues is also pretty straight forward. There is merely the right queue to be found and a bit to be set or removed from the priority bitmap. Therefore, I will not supply any further details here.


---

# 9. 软实时调度

Linux 调度器支持软实时（RT）任务调度。这意味着内核可以有效的调度有严格时间限制的任务。虽然内核又能力满足对截止时间要求非常严格的实时任务调度，但是并不能保证一定能满足这个截止时间。

对应的调度类是在 `kernel/sched_rt.c` 实现的 `rt_sched_class` 。 RT 任务拥有比 CFS 任务高的优先级。

## 9.1. 调度模式

由 RT 调度器处理的任务可以配置成下面两种不同的模式：

- SCHED_FIFO ： 一个按照 FIFO（先入先出） 模式调度的任务并没有时间片，并且只会在任务终止、放弃处理器时被切换。
- SCHED_RR ： 一个 RR 任务会按照固定的时间片进行调度，然后会按照循环方式被相同优先级的任务抢占。这就是说只要任务的时间片用完了就会被放到任务队列的末尾并且时间片会重新填满。在时间片用完之前，任务同样也可能被高优先级的任务抢占。

## 9.2. 优先级

根据优先级的实现内容，RT 调度类也遵循之前 O(1) （复杂度）调度器同样的规则。内核使用多个运行队列，每个队列对应一个优先级。按照这种方式，添加、删除或者寻找最高优先级任务的操作可以在 O(1) 时间内完成。

## 9.3. 实时调节

你偶尔可以看到 RT 调度器实现中的调节和带宽操作。添加这些操作是为了提高 `SCHED_FIFO` 任务的安全性。在任务自己想要或不想要被打断之前，这些操作会为每个处理器分配一个由 FIFO 任务组成的 RT 任务组。这就在内核开发者当中引起了一些讨论，而讨论的结果还没有定下来。

## 9.4. 实现细节

### 数据结构

和 CFS 类似， RT 调度器有它自己的调度实体和运行队列数据结构，这些已经作为成员添加到了 `task_struct` 和主运行队列。

`sched_rt_entity` 在 `/include/linux/sched.h` 实现。它有记录时间片的字段，一个指向它所属优先级列表的指针，以及和组调度相关的成员。

```
struct sched_rt_entity {
    struct list_head run_list;
    unsigned long timeout;
    unsigned int time_slice;
    int nr_cpus_allowed;
    struct sched_rt_entity *back;
#ifdef CONFIG_RT_GROUP_SCHED
    struct sched_rt_entity*parent;
    /* rq on which this entity is (to be) queued: */
    struct rt_rq *rt_rq;
    /* rq "owned" by this entity/group: */
    struct rt_rq *my_q;
#endif
};
```

`rt_rq` 在 `kernel/sched.c` 实现。第一个字段记录了优先级数组。其他几乎所有的字段都是与 SMP 和组调度相关的。

```
/* Real-Time classes' related field in a runqueue: */
struct rt_rq {
    struct rt_prio_array active;
    unsigned long rt_nr_running;
#if defined CONFIG_SMP || defined CONFIG_RT_GROUP_SCHED
    struct {
        int curr; /* highest queued rt task prio */
#ifdef CONFIG_SMP
        int next; /* next highest */
#endif
    } highest_prio;
#endif
#ifdef CONFIG_SMP
    unsigned long rt_nr_migratory;
    unsigned long rt_nr_total;
    int overloaded;
    struct plist_head pushable_tasks;
#endif
    int rt_throttled;
    u64 rt_time;
    u64 rt_runtime;
    /* Nests inside the rq lock: */
    raw_spinlock_t rt_runtime_lock;
#ifdef CONFIG_RT_GROUP_SCHED
    unsigned long rt_nr_boosted;
    struct rq *rq;
    struct list_head leaf_rt_rq_list;
    struct task_group *tg;
#endif
};
```

## 记录时间

`scheduler_tick()` 调用 `task_tick_rt()` 更新当前任务的时间片。实现相当简单：一开始就使用 `update_curr_rt()` 更新当前任务的运行时数据和对应的运行队列。然后这个函数返回当前任务是否是 FIFO 任务。

如果不是， RR 任务的时间片就会减 1 。如果减到 0 了就设回默认值然后如果还任务队列中有其他任务的话就把当前任务放到队列的末尾。此外还要设置 `need_resched` 标志。

```
static void task_tick_rt(struct rq *rq, struct task_struct *p, int queued)
{
    update_curr_rt(rq);
    watchdog(rq, p);
    /*
    * RR tasks need a special form of timeslice management.
    * FIFO tasks have no timeslices.
    */
    if (p->policy != SCHED_RR)
        return;
    if (--p->rt.time_slice)
        return;
    p->rt.time_slice = DEF_TIMESLICE;
    /*
    * Requeue to the end of queue if we are not the only element
    * on the queue:
    */
    if (p->rt.run_list.prev != p->rt.run_list.next) {
        requeue_task_rt(rq, p, 0);
        set_tsk_need_resched(p);
    }
}
```


### 挑选下一个运行的任务

如之前所说的，挑选下一个运行的任务可以在常数时间复杂度内完成。以函数 `pick_next_task_rt()` 开始，它会直接调用 `_pick_next_task_rt()` 完成实际工作。

```
static struct task_struct *pick_next_task_rt(struct rq *rq)
{
    struct task_struct *p = _pick_next_task_rt(rq);
    /* The running task is never eligible for pushing */
    if (p)
        dequeue_pushable_task(rq, p);
#ifdef CONFIG_SMP
    /*
    * We detect this state here so that we can avoid taking the RQ
    * lock again later if there is no need to push
    */
    rq->post_schedule = has_pushable_tasks(rq);
#endif
    return p;
}
```

如果没有可以运行的任务，就返回 NULL，然后其它调度类会寻找可运行的任务。

`pick_next_rt_entity()` 会从运行队列中选出优先级最高的任务作为下一个运行的任务。


```
static struct task_struct *_pick_next_task_rt(struct rq *rq)
{
    struct sched_rt_entity *rt_se;
    struct task_struct *p;
    struct rt_rq *rt_rq;
    rt_rq = &rq->rt;
    if (!rt_rq->rt_nr_running)
        return NULL;
    if (rt_rq_throttled(rt_rq))
        return NULL;
    do {
        rt_se = pick_next_rt_entity(rq, rt_rq);
        BUG_ON(!rt_se);
        rt_rq = group_rt_rq(rt_se);
    } while (rt_rq);
    p = rt_task_of(rt_se);
    p->se.exec_start = rq->clock_task;
    return p;
}
```

在 `pick_next_rt_entity()` 中你可以看到如何使用位图来在全部优先级中快速的找出拥有可运行任务的优先级最高的队列。优先级队列本事是一个简单的链表，而下一个可运行的任务可以在它的表头找到。


```
static struct sched_rt_entity *pick_next_rt_entity(struct rq *rq,
        struct rt_rq *rt_rq)
{
    struct rt_prio_array *array = &rt_rq->active;
    struct sched_rt_entity *next = NULL;
    struct list_head *queue;
    int idx;
    idx = sched_find_first_bit(array->bitmap);
    BUG_ON(idx >= MAX_RT_PRIO);
    queue = array->queue + idx;
    next = list_entry(queue->next, struct sched_rt_entity, run_list);
    return next;
}
```

在优先级队列中添加、删除任务也是非常的简单。仅仅就是找出正确的队列然后设置或者清掉优先级位图中对应位置。因此我就不在这里提供更进一步的细节了。

