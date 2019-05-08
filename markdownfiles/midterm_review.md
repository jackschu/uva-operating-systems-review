
# Table of Contents

1.  [Midterm review for Operating Systems](#org4cb48e9)
    1.  [Parital reads /writes](#orge2f4458)
    2.  [Ker vs user mode](#org5691369)
    3.  [context switching](#org533bc47)
    4.  [differences b/w scheduling algos](#org7083887)
    5.  [process vs threads](#org30cb506)
    6.  [cache coherency](#org34ed22a)
    7.  [addr translation](#org0644ccd)
    8.  [race condition avoiding:](#org612f79f)
    9.  [avoiding busy waits](#org3eb9105)
    10. [implementing spinlock](#orgd39ea4b)


<a id="org4cb48e9"></a>

# Midterm review for Operating Systems


<a id="orge2f4458"></a>

## Parital reads /writes

-   Posix has read write, partial writes can occur because:
-   they return whatever they can (sometimes not everything)
    -   wait until at least one byte
-   write what I can now, return how much I've written (wait at least one)


<a id="org5691369"></a>

## Ker vs user mode

-   Processor feature
-   only allows ops in kernel mode
    -   eg most io, setting up exception handling
    -   changing base table register
    -   accessing virtual pages marked as kernel-mode only
-   convention:
    -   handling system calls, scheduler, io runs in kernel mode
    -   ALL user code runs in user mode
-   switching to kernel mode ONLY happens via exception/trap/interrupt


<a id="org533bc47"></a>

## context switching

-   context: stuff we need to run on the cpu for a thread
    -   minimally: registers + PC
-   save registers + PC of old thread, restore stuff from new thread
-   in xv6 this is done through swtch():
    -   does cool stuff with push/pop
-   context switch b/w processes
    -   save/restore the addr space
    -   ie save old page table base register
    -   in xv6 this is separate from kernel context switch
        -   ie swtch() to change kern
        -   swtchuvm() to change page tables
-   xv6 only swaps b/w programs in kernel mode
-   user registers are saved b/c exception (NOT b/c of a context switch)


<a id="org7083887"></a>

## differences b/w scheduling algos

-   preemptive vs non
    -   preemptive: when ready run it
    -   non- only stop progs when it gives up the CPU
    -   by goals
        -   fair (will not starve)
            -   CFS &#x2013; weights
            -   RR everyone gets turn
            -   lottery, weights with random
        -   throughput
            -   FCFC/FIFO (just run next thing)
        -   wait time/ turnaround 
            -   interactivity optimizing  / get io to happen sooner
            -   SJF/SRTF &#x2013; ie run shortest CPU burst (cant do this)
            -   MLFQ (multilevel feedback queue) (actually dooable)
        -   meetind deadlines &#x2013; earliest deadline first


<a id="org30cb506"></a>

## process vs threads

-   thread &#x2013; what gets run on a cpu
    -   program counter
    -   cpu registers
    -   stack
-   process
    -   one or more threads
    -   file descriptors
        -   shared b/w all threads
    -   addr space
        -   shared b/w all threads
        -   contains mult stacks
    -   process id
    -   current directory


<a id="org34ed22a"></a>

## cache coherency

-   invalid = not cached
-   modifed = cached copy thats newer than memory
-   shared = multiple processors have valid cached copies


<a id="org0644ccd"></a>

## addr translation

-   take virt addr, spit into vpn, and page offser
-   split vpn into parts for each level
-   do an array lookup based on split for vpn at each level
-   use given base addr for first level
-   use pte entree \* page size for base addr of each next level
-   last entree has ppn of the data, concat ppn w/ page offset for phy addr
-   leading zeros are there sometimes


<a id="org612f79f"></a>

## race condition avoiding:

-   if you have shared data, lock before making changes
-   never assume anything about what happens while you dont have it locked


<a id="org3eb9105"></a>

## avoiding busy waits

-   scenario: waiting for event to happen
    -   naive (busywait) continuously check if it happened
-   primitives to help:
    -   condition variables, (protected by lock)
        -   check if happened else wait, when happens , get broadcast/singal
        -   recheck on wake due to spurious wakeups
    -   semaphores:
        -   P/down/wait - wait to become positive then dec
        -   V/up/signal/post - increment by 1, wake a thread if need
        -   initially 0
        -   check if need to wait (prob with lock)
        -   mark that we've started waiting
        -   release lock
        -   wait (down) on semaphore
        -   if someone's stared waiting then post (up) on semaphore


<a id="orgd39ea4b"></a>

## implementing spinlock

-   simplest strat: atomic operation that reads a val then writes LOCKED
-   if we read unlocked, we know that this thread can go and lock it
    -   test and set, did we change it? no? try again?

