
# Table of Contents

1.  [Review for final](#org80152ea)
    1.  [Virtual Memory notes](#org6ffd7e5)
    2.  [DMA/File System  file systems](#org3156e94)
    3.  [More File system April 2-4 file systems](#org09252b2)
    4.  [Sockets / Distributed systems April 9-11 io dist](#orgdef9c95)
    5.  [More Distributed systems / Access Control dist access](#org13eefe6)
    6.  [Virtual Machines  most of the notes](#org51de437)
    7.  [Esp Important topics](#org7aa39fb)
2.  [final review lecture notes](#orgf04e1e8)
    1.  [why use mmap](#org2321551)
    2.  [for direct pointers](#org36bc68f)
    3.  [fs snapshots](#orga2265a6)
    4.  [virtual machine quiz question](#org18de7cf)
    5.  [programmed io vs dma](#org00b88ed)
    6.  [device controller vs driver](#org7d5d0ac)
    7.  [page replace policies](#orgdea066d)
    8.  [pros and cons of FSs](#org7523b26)


<a id="org80152ea"></a>

# Review for final


<a id="org6ffd7e5"></a>

## Virtual Memory [notes](virtual_mem.md)

-   Page Cache comp
-   Fw/Bkwrd Mapping
-   Algorithms for cache replacement
    -   Belady's MIN
    -   LRU
    -   Second Chance
    -   SEQ
    -   CLOCK/CLOCK-Pro
-   Multiple Mappings to a single Physical Page
-   Terms
    -   Thrashing - Memory is fully occupied so no progress due to busy work
    -   Read-ahead -
    -   more&#x2026;


<a id="org3156e94"></a>

## DMA/File System  [file systems](file-systems.md)

-   Programmed I/O
-   Direct Memory Access (DMA)
-   IOMMU
-   FAT file system
    -   Header
    -   Creating files, writing
    -   Disk scheduling algorithms
    -   Block remapping
    -   Xv6 disk-layout
    -   indirection


<a id="org09252b2"></a>

## More File system April 2-4 [file systems](file-systems.md)

-   Performance, Fragmentation
-   Non-binary search trees
-   RAID - how to calculate parity data (4,5,6)
-   Mirroring disks
-   Reliable FS
    -   FAT ordering
    -   inode-based FS
    -   redo logging


<a id="orgdef9c95"></a>

## Sockets / Distributed systems April 9-11 [io](IO.md) [dist](distributed_systems.md)

-   Mailbox Abstraction (somehow important?) not really
-   IPv4/IPv6
-   FTP (corner cases on exam) very important
-   Network API functions
-   concepts
    -   Blocking/Non-blocking I/O
    -   Duplex/Half-duplex
-   Remote Procedure Call RPC
    -   Marshalling
    -   IDL (Interface Description Language)
-   Network File System (NFS)
-   Stateful vs Stateless
-   Consistency issues in NFS/AFS
    -   NFS can deal with inconsistency in block granularity
    -   AFS deals with it in file granularity


<a id="org13eefe6"></a>

## More Distributed systems / Access Control [dist](distributed_systems.md) [access](access_control.md)

-   Two-phase commit
    -   Worker failure
    -   something (server failure)?
-   Access control list


<a id="org51de437"></a>

## Virtual Machines  [most of the notes](virtual_machines.md)

-   Shadow Page Table
-   One approach to implementing: pure emulation


<a id="org7aa39fb"></a>

## Esp Important topics

-   Virtual Memory
-   Context Switch
-   Schedulers
-   Synchronization
-   Threads


<a id="orgf04e1e8"></a>

# final review lecture notes


<a id="org2321551"></a>

## why use mmap

-   convenient to load files into mem
-   load code this way
-   does copy on write for you
-   how it works
    -   proc control block contains list of regions
    -   can share mmapd stuff with cache
-   mmap can be used to shaRe memory b/w programs


<a id="org36bc68f"></a>

## for direct pointers

-   note partial overhead is possible (and common)
-   ie not all pointers have to be non-null


<a id="orga2265a6"></a>

## fs snapshots

-   user point of view
    -   two or more copies of filesystem
    -   which represent different versions
    -   but save space when files in common
-   implementation
    -   copy on write
    -   if two copies have one file thats different
        -   they'll have pointers to different inode arrays (literally all inodes (not 13-17))
            -   inode array is stored in pieces
            -   only parts which are different b/w two copies will have different points
        -   and only one inode will be diff b/w the two
        -   two inodes will have blocks in common where they share data
    -   details:
        -   normal file updates need to make new copies
        -   need to track reference counts


<a id="org18de7cf"></a>

## virtual machine quiz question

-   suppose a guest os is running on vm using trap and emulate
-   a user program running in guest os tries to access unallocated memory
-   this mem will be alloc on demand by guest  os, what will happen
    -   user's mem will trigger page fault in host os
    -   WRONG: the host os pg fault handler will update the guest os system page table then resume
        -   not correct, because hypervisor doesnt touch guest page table itself
        -   guest os is the one that wants to have this allocated on demand, not the host os
    -   WRONG: host pg fault handler will cause guest os fault handler to run in kernel mode
        -   hyper never runs guest os in kernel mode (unless hw virt support)
    -   RIGHT: host pg fault handler will cause guest os fault handler to run in user mode
        -   "reflecting the exception"
-   q2: last-level guest page table has ppn 0x40,
-   guest physical mem starts at 0x1000000 (machine pg num 0x1000, offset 0x0)
    -   4k pages, 12-bit page offset
-   what does the shadow pte corresponding to this look like
-   mapped contiguously
    -   so  ppn 0x0 is mpn 0x1000
    -   ppn 0x1 is mpn 0x1001
        -   physical address 0x1000 is machi addr 0x1001000

-   equivalently ppn 40 is what machine page number


<a id="org00b88ed"></a>

## programmed io vs dma

-   programmed io
    -   buffer on device can do special things
    -   prob better for keyboard
-   dma, buffer is in memory on machine
    -   os tells device where to put memory
    -   doesnt make sense for low data transfer devices
    -   better for usb
    -   even required for some graphics stuff since buffer would be too small otherwise


<a id="org7d5d0ac"></a>

## device controller vs driver

-   driver is on os
    -   response to interrupts, knows how to communicate with device controller
-   controller is part of device hardware


<a id="orgdea066d"></a>

## page replace policies

-   real lru is too expense b/c page fault on every read or write
-   approximations (not recently used)
    -   second chance (one list) mark accessed
    -   seq
        -   keep two ordered list
        -   active assume prob used
        -   inactive guess unused
        -   move pages from active to inactive
        -   only update inactive pages as accessed
    -   clock: general idea of scan + clear "was it accessed information"
        -   and do something with recent history for each page
        -   ie was it accessed when i checked it n times ago
-   secondary family (when lru is wrong) eg files
    -   clock pro: allow things to be evicted if they arent accessed twice (with some interval in between or twice with read() calls)


<a id="org7523b26"></a>

## pros and cons of FSs

-   FAT filesystem
    -   simple to implement (major benefit)
    -   practical to implement with constrained memory? maybe
    -   less overhead, less built around caches (small benefit)
    -   seeking is v slow
    -   reliability is uhh bad (redundant fats but w/e)
    -   doesnt keep file and dir data close to each other
    -   dirs arent automatically close to files
-   FFS-like filesystems
    -   use fragments (small files)
    -   block groups (close to each other)
    -   have all free-block info in one place
-   other FS features
    -   extents:
        -   pro: good for large files
        -   con: more complicated to alloc/seek through
    -   trees for dirs
        -   pro: more efficient to find filename in dir
        -   con: more complicated to do things
    -   journaling/logging
        -   pro: recovery from failures
        -   pro: faster writes
        -   con: more complicated to implement
        -   con: doesnt eliminate data loss entirely

