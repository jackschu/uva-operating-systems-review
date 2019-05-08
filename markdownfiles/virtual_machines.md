
# Table of Contents

1.  [Virtual machines notes](#org0ec13b3)
    1.  [recall virtual machine interface](#org37740a6)
    2.  [System virtual machine](#orge26038f)
    3.  [How to do it?](#orgc384826)
    4.  [Address terms (for these notes)](#org428fe29)
    5.  [Proc control block for guest OS](#org2c4f1b1)
    6.  [basic hypervisor flow](#org8ff556d)
        1.  [what about memory mapped I/O?](#orgb3cd1a0)
        2.  [This doesnt work for everything though](#orgcbbd138)
    7.  [What about guest OS virtual memory](#org60dc628)
    8.  [Shadow page table synthesis](#org8c52cf3)
    9.  [Hardware hypervisor support](#orgb34799f)
    10. [Non-virtualizable Instructions](#orgc68cfd7)


<a id="org0ec13b3"></a>

# Virtual machines notes


<a id="org37740a6"></a>

## recall virtual machine interface

-   process virtual machine = operating system
-   it isolates an application


<a id="orge26038f"></a>

## System virtual machine

-   not a process virtual machine
-   terms
    -   hypervisor or virtual machine monitor : runs sys vms
    -   guest OS: os that runs as an app on hypervisor
    -   host OS: operating system that runs hypervisor (sometimes hypervisor is OS)
-   levels of virtualization:
    -   full virtualization: guest OS has no clue, as if on real hardware
    -   paravirtualization: small modifications to guest OS to support vm
        -   can make vms simpler and faster
    -   this can be fuzzy: what if we need device driver for vm? which one is it? unclear


<a id="orgc384826"></a>

## How to do it?

-   compile guest OS machine code to new machine code
    -   sounds slow, not actually that slow
    -   what qemu does


<a id="org428fe29"></a>

## Address terms (for these notes)

-   virtual addr: virt addr for guest OS
-   phys addr: phys addr for guest OS
-   machine addr: phy addr for hypervisor/host OS


<a id="org2c4f1b1"></a>

## Proc control block for guest OS

-   hypervisor runs guest OS like a process but needs to track extra stuff
    -   if guest OS thinks interrupts are disable
    -   what guest OS thinks it's interrupt handler is
    -   what guest OS thinks it's base pg table register is
    -   if guest OS thinks its running in kernel mode


<a id="org8ff556d"></a>

## basic hypervisor flow

-   run guest OS in user mode
-   other OS stuff causes exceptions that hypervisor now handles
    -   eg disable interrupts, use hardware etc
-   making io / kern mode instructions work
    -   trap and emulate
    -   force instruction to cause a fault
    -   fault hander does what instruction would do
    -   eg i/o
        -   guest os: try to access keyboard
        -   causes trap
        -   if guest os thinks its in pretend kernel mode
        -   hypervisor reads from keyboard on behalf of guest os
        -   hypervisor updates guest os state and switches back
-   making exceptions/interrupts work
    -   reflect exceptions/interrupts into guest os
    -   eg system call (exception in hw)
    -   hw calls exception handler in hypervisor
    -   lookup exception handler in guest os
    -   update page table to be in hypervisor kernel mode
    -   return to guest os's syscall handler with kernel mode enabled
    -   after return from system call handler
    -   calls real syscall handler for syscall return in hw
    -   hw goes to exception handler in hypervisor
    -   hypervisor moves guest os back to user mode (page table update)
    -   returns back to guest os
-   making page table work
    -   hard, is its own topic
-   Guest os is in usermode
-   Hypervisor is on kernel mode
-   hypervisor lets guest os pretend its in kernel mode


<a id="orgb3cd1a0"></a>

### what about memory mapped I/O?

-   magic memory
-   make a page fault happen when guest os tries to access magic device addr
-   therefore hypervisor has two types of page faults (+)
    -   gotta detect if pg fault is for device
    -   or a regular page fault


<a id="orgcbbd138"></a>

### This doesnt work for everything though

-   eg what if we have instructions that operate differently in ker vs user
-   but do not cause a fault (there are such instructions)
-   there are work arounds but they're annoying


<a id="org60dc628"></a>

## What about guest OS virtual memory

-   three page tables
    -   guest page table
        -   page table location set with priv instruction to let hypervisor knows
    -   hypervisor page table (?) maybe some other data structure
    -   need a third page table to translate virtual add to machine addr
        -   dont want to go through hypervisor if not setting up (most of time)
        -   this is would take too long
        -   we call it the shadow page table (shadow of guest page table)
        -   made by hypervisor combining guest page table and hyper pt


<a id="org8c52cf3"></a>

## Shadow page table synthesis

-   creating new page table
-   when to create new shadow page table?
    -   its p expensive, dont want to do it often
    -   trick: use the Translation Lookaside Buffer (cache for pte)
-   TLB has the same problem (contents synthesized from normal page table)
-   processor also needs to decide when to update it
    -   recall tlb gets asked for addr
    -   then fetches on demand from page table if cache miss
    -   when os sets pte, tlb is out of sync
    -   so the procs are supposed to tell os when the update page table
    -   note this implies we can use that telling to update shadow pt
-   in some sense, the shadow page table is imitating tlb a virtual tlb
    -   it caches page tables entrees just like tlb, filled in by hypervisor
    -   update on demand
-   shadow page table is used by hypervisor
-   memory mapped i/o: we cant map this in pt, just leave invalid
-   many os's invalidate entire tlb on context switch
-   this is not fast enough, would like to cache shadow page table b/w switches
-   but problem: os won't tell you when the cached shadow page table has changed
    -   most of the time (very recent x86 will)
    -   this is a reason for paravirt
-   solution: same idea as magic memory
    -   mark every page table entree of non-current page table as invalid
    -   then handle updating cached shadow page table on exception in hypervisor
    -   complicated, slow, but worth it
    -   problem: what if we dont need the cached page table anymore? idk guess
-   page tables in kernel mode?
    -   have one shadow kernel pt
    -   have one shadow user pt
    -   (or we could set bits but we dont)
-   Ex how many page table switches
    -   guest makes read() syscall [1 syscall user to kernel]
    -   guest os switches to other [2 original to other] [3 kernel-> user (if next prog is in usermode)]
    -   guest os gets interr from keyboard [4 kernel to user]
    -   guest switches back to og, returns from syscall [5 other -> orig 6 return syscall ker-> user]
    -   how many guest pt switches (2 orig-> other, other -> og)
    -   how many real/shadow pt [6 (maybe 5)]


<a id="orgb34799f"></a>

## Hardware hypervisor support

-   Intel's VT-x:
-   HW tracks whether a vm is running
-   new VMENTER instruction
    -   switches page tables, sets prog counter etc
    -   this makes all x86 bad boi instructions cause exceptions (configurable)
-   similar VMEXIT instruction
    -   exits vm is running mode, switch to hypervisor
-   this way can run guest OS directly on the hw for most things
-   some things cause VMEXIT (hypervisor enter) some things run directly on hw
-   allows us to have one shadow pg table for both user kernel per proc
-   tagged tlbs:
    -   address space in TLB entries
    -   helps normal OS (faster context switching)
    -   hypervisor and or os sets address space ID when switching page tables
    -   makes extra work for OS/hypervisor (needs to flush TLB entries even when changing non-active pt)
-   nested page tables:
    -   allows hypervisor to specify two page table base registers
    -   then the hardware walks both page tables whenever it needs to do lookup
    -   much faster cus hw does it not conseq hypervisor job, not the easiest thing to do
    -   for 2 nested 2 level page tables, how many lookups needed?
        -   8 lookups (every lookup in guest requires 2 lookups in hypervisor (conv physical to machine)
    -   this is even more for 4 level page tables (more realistic)


<a id="orgc68cfd7"></a>

## Non-virtualizable Instructions

-   popf pushf is hard for the trap-exception thing
    -   pops flags from stack
    -   including I/O privilege level, and interrupt enable flag
    -   does different things for user / kernel mode
    -   in user mode does nothing (not even exception), in kernel mode does something
        -   design mistake
    -   pushf pushes flags to stack (including privileged flags)
    -   does the same thing in user/kernel (no exception)
        -   but hypervisor needs to know about this
-   options:
    -   patch the os (not too bad, small changes)
        -   this is paravirtualization
        -   eg make popf, pushf cause exception
    -   binary translation
        -   compile machine code into new machine code
        -   practical cus only do it for for kernel code that uses hard instructions
    -   change instruction set

