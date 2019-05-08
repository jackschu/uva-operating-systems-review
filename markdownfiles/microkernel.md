
# Table of Contents

1.  [Monolithic vs microkernel](#org40d969f)
    1.  [Monolithic](#org19ac054)
    2.  [microkernel](#orgc5a75f0)
    3.  [example microkernel seL4](#org576ab10)
    4.  [other os designs](#org4fbcdec)


<a id="org40d969f"></a>

# Monolithic vs microkernel


<a id="org19ac054"></a>

## Monolithic

-   linux
-   have everything in kernel thats convenient


<a id="orgc5a75f0"></a>

## microkernel

-   have everything as process (as much as possible)
-   file system, network driver, device driver
-   more modular, killable processes (less bugs)
-   kernel now only provides fast communication to other procs
    -   kernel must be v fast since we use it for communication
    -   also need to do a lot of auth now
-   kernel is responsible for
    -   interprocess communication
    -   raw access to devices
    -   cpu scheduling
    -   virtual memory (do a lot less of this please)


<a id="org576ab10"></a>

## example microkernel seL4

-   good literature on it, formally verified as machine check proof
-   full list of default syscalls
    -   send, nbsend, reply, recv, nbrecv, call, replyrecv, yield
-   how to alloc memory?
    -   send message to kernel object asking for memory
    -   everything is a message
-   everything is capabilites
    -   where to send/recv from
    -   can send capabilities via messages
-   objects
    -   kernel objects (named via capabilities)
        -   cnode - capability store
        -   tcp - thread control block
        -   endpoint, notification - ipc thingy
        -   pagedirectory, pagetable - virtual memory
        -   frame, untyped - available memory
        -   irqcontrol, irqhandler - interrupts
        -   a couple more..
-   choices they made:
    -   exposes stuff p directly (keeps kernel simple)
    -   no kernel memory allocation built in to kernel
    -   even memory for kernel objects is not allocated by kernel (choice for proof)
-   memory:
    -   memeory starts as untyped objects (variable size, can be split)
    -   convert it to other things
    -   to alloc memory convert it to a frame
    -   can only be converted to one thing at a time
-   capabilites
    -   takes slots in cnode
    -   can copy capabilites (and drop some permissions)
    -   can copy derived capabilities to other processes
-   object deletion:
    -   kernel tracks reference count of every object
    -   if ref count is 0, original is deleted -> untyped
    -   kernel knows where capabilities are for each obj, deletes them recursively
    -   tracks parent child relationship of capabilities
        -   revoking parent capability kills children (derived capabilities)
-   messages:
    -   tag: message type and size
    -   badge: identifying source (says which sender was used)
    -   message words ( a few): messages are small (stored in cpu registers fast)
    -   one or more capabilites: (send these instead if your message is gunna be long)
    -   messages are synchronous
        -   adv message not copied into kernel buffer
        -   handle message by context switching to target
        -   fast path; messages are entirely in registers
        -   scheduler can just switch from sender to receiver w/ minimal context switch
    -   send cases
        -   send to kernel obj: invoke kern handler reply
        -   send to ready to recieve prog (schedulable): just switch now
        -   send to program not ready to recieve: add thread to queue (switch to something else)
        -   send to invalid dest: reply with error
    -   sendrecv optimization:
        -   combined system call, marked ready to recieve immediately after send
        -   avoids too much waiting
-   notifications: async IPC
    -   works on binary semaphores
-   virtual memory:
    -   threads are assoc with page dir and page table (gotta send capabilites for requested frame)
    -   send messages to objects to map pages
    -   cow is implemented on your own
        -   specify your own exception endpoint
        -   exceptions are message sends, do page faults through this


<a id="org4fbcdec"></a>

## other os designs

-   exokernel: (an extreme)
    -   kernel only provides hw interface,
    -   only filters hw usage for safety
    -   kernel just askes programs which page to free
    -   kernel only moves packets (no socket abstractions)
    -   kernel does not keep thread control blocks: library os tell me to run
    -   tell about io, dont do anything else in response to io
-   singlularity: (failed microsoft research)
    -   os runs CIL bytecode
    -   no page tables at all (system calls literally just function calls)
    -   rely on bytecode to keep processes memory seperate
    -   specture / meltdown leaks through caches makes this not work

