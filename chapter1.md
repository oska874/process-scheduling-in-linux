# 1 Process Scheduling

The current Linux kernel is a multi-tasking kernel. Therefore, more than one process are allowed to

exist at any given time and every process is allowed to run as if it were the only process on the

system. The process scheduler coordinates which process runs when. In that context, it has the

following tasks:

• share CPU equally among all currently running processes

• pick appropriate process to run next if required, considering scheduling class\/policy and

process priorities

• balance processes between multiple cores in SMP systems

