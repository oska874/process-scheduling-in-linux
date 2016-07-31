# 7. Completely Fair Scheduler \(CFS\)

Lets now describe the scheduling algorithm used for the NORMAL category tasks: The current scheduler is the completely fair scheduler by Ingo Molnar based on the Rotating Staircase Deadline Scheduler \(RSDL\) by Con Kolivas. CFS' implementation hides behind the fair\_sched\_class in \/kernel\/sched\_fair.c.

The CFS algorithm is based on the idea of an ideal multi-tasking processor. Such a processor would run all currently active tasks literally at the same time while each task gets a portion of its processing power. For example, if two tasks would be runnable, they would run at the same time with 50% processing power each. That means, assuming the two tasks started at the same time, the execution time or runtime of each task is at any moment in time exactly the same, therefore completely fair. You could also say, each tasks runs for an infinitesimal small amount of time but with full processing power.

Since it is physically not possible to drive current processors in that way and it is highly inefficient to run for very short times due to switching cost, CFS tries to approximate this behaviour as closely as possible. It keeps track of the runtime of each runnable task, called virtual runtime, and tries to maintain an overall balance between all runnable tasks at all times.

The concept is based on giving all runnable tasks a fair share of the processor. The total share grows and shrinks with the total number of running processes and the relative share is inflated or deflated based on the tasks' priorities.

## 7.1. Time Slices vs. Virtual Runtime

In CFS, the traditional model of an epoch divided into fixed time slice for each task does not exist any more. Instead, a virtual runtime \(vruntime\) was introduced for each task. In every call of scheduler\_tick\(\) for a scheduled CFS task, its vruntime is updated with the elapsed time since it was scheduled. As soon as the vruntime of another task in the runqueue is smaller than the current tasks', a rescheduling is executed and the task with the smallest vruntime is selected to run next.

To avoid an immense overhead of scheduling runs due to frequent task switches, a scheduling latency and a minimum task runtime granularity were introduced.

The target scheduling latency \(TSL\) is a constant which is used to calculate a minimum task runtime. Lets assume TSL is 20ms and we have 2 running tasks of the same priority. Each task would then run for a total of 10ms before pre-empting in favour of the other.

If the number of tasks grows, this portion of runtime would shrink towards zero which again would lead to an inefficient switching rate. To counter this, a constant minimum granularity is used to stretch the TSL depending on the amount of running tasks.

## 7.2. Priorities

A task is prioritised above another by weighting how fast its vruntime grows while being scheduled. The time delta since the task was scheduled is weighted by the task's priority. That means, a high priority tasks' vruntime grows slower than the vruntime of a low priority one.

## 7.3. Handling I\/O and CPU bound tasks

The previous scheduling algorithm, the O\(1\) scheduler, tried to use heuristic of sleep and runtimes to determine if a task is I\/O or CPU bound. It would then benefit one or the other. As it turned out later, this concept did not work quite as satisfying as expected due to the complex and error prone heuristics.

CFS' concept of giving tasks a fair share of the processor works quite well and is easy to apply. I would like to explain how this works using the example given in Robert Love's Linux Kernel Development:

Consider a system with two runnable tasks: a text editor and a video encoder. The text editor is I\/Obound because it spends nearly all its time waiting for user key presses. However, when the text editor does receive a key press, the user expects the editor to respond immediately. Conversely, the video encoder is processor-bound.

Aside from reading the raw data stream from the disk and later writing the resulting video, the encoder spends all its time applying the video codec to the raw data, easily consuming 100% of the processor. The video encoder does not have any strong time constraints on when it runs—if it started running now or in half a second, the user could not tell and would not care. Of course, the sooner it finishes the better, but latency is not a primary concern. In this scenario, ideally the scheduler gives the text editor a larger proportion of the available processor than the video encoder, because the text editor is interactive. We have two goals for the text editor. First, we want it to have a large amount of processor time available to it; not because it needs a lot of processor \(it does not\) but because we want it to always have processor time available the moment it needs it. Second, we want the text editor to pre-empt the video encoder the moment it wakes up \(say, when the user presses a key\). This can ensure the text editor has good interactive performance and is responsive to user input.

CFS solves this issue in the following way: Instead of assigning the text editor a specific priority and timeslice, it guarantees the text editor a specific proportion of the processor. If the video encoder and text editor are the only running processes and both are at the same nice level, this proportion would be 50% — each would be guaranteed half of the processor’s time. Because the text editor spends most of its time blocked, waiting for user key presses, it does not use anywhere near 50% of the processor. Conversely, the video encoder is free to use more than its allotted 50%, enabling it to finish the encoding quickly.

The crucial concept is what happens when the text editor wakes up. Our primary goal is to ensure it runs immediately upon user input. In this case, when the editor wakes up, CFS notes that it is allotted 50% of the processor but has used considerably less. Specifically, CFS determines that the text editor has run for less time than the video encoder. Attempting to give all processes a fair share of the processor, it then pre-empts the video encoder and enables the text editor to run. The text editor runs, quickly processes the user’s key press, and again sleeps, waiting for more input. As the text editor has not consumed its allotted 50%, we continue in this manner, with CFS always enabling the text editor to run when it wants and the video encoder to run the rest of the time.

## 7.4 Spawning a new Task

CFS maintains a vruntime\_min value, which is a monotonic increasing value tracking the smallest vruntime among all tasks in the runqueue. This value is used for new tasks that are added to the runqueue to give them a chance of being scheduled quickly.

## 7.5 The Runqueue – Red-Black-Tree

As a runqueue, CFS uses a Red-Black-Tree data structure. This data structure is a self balancing binary search tree. Following a certain set of rules, no path in the tree will ever be more than twice as long as any other. Furthermore, operations on the tree occur in O\(log n\) time, which allows insertion and deletion of nodes to be quick and efficient.

Each node represents a task and they are ordered by the task's vruntime. That means the left most node in the tree always points to the task with the smallest vruntime, say the task that has the gravest need for the processor. Caching of the left most node makes it quick and easy to access.

## 7.6 Fair Group Scheduling

Group scheduling is another way to bring fairness to scheduling, particularly in the face of tasks that spawn many other tasks. Consider a server that spawns many tasks to parallelise incoming connections. Instead of all tasks being treated fairly uniformly, CFS introduces groups to account for this behaviour. The server process that spawns tasks share their vruntimes across the group \(in a hierarchy\), while the single task maintains its own independent vruntime. In this way, the single task receives roughly the same scheduling time as the group.

Let's say that next to the server tasks another user has only one task running on the machine. Without group scheduling, the second user would be treated unfairly in opposition to the server. With group scheduling, CFS would first try to be fair to the group and then fair to the processes in the group. Therefore, both users would get 50% share of the CPU.

Groups can be configured via a \/proc interface.


---







