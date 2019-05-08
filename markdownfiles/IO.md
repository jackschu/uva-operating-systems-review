
# Table of Contents

1.  [Input Output CS4414](#orgf19e287)
    1.  [how to talk to IO devices (still a file)](#org69ea531)
    2.  [device driver flow](#orgaf75648)
    3.  [device files](#org2b8ff0d)
    4.  [how are devices connected to processor](#org482a529)
    5.  [devices and caching?](#org4b38ea6)
    6.  [programmed i/o](#org282dfa0)


<a id="orgf19e287"></a>

# Input Output CS4414


<a id="org69ea531"></a>

## how to talk to IO devices (still a file)

-   implement open read write close for each device (driver)
-   what about options for devices or extra operations eg eject
    -   eg ioctl for weird things
-   Linux has a struct for file<sub>operations</sub>
    -   read write unblocked<sub>ioctl</sub> mmap open releases
    -   special case block devices
-   block devices (devices that work in terms of blocks) eg disk
    -   struct block<sub>device</sub><sub>operations</sub>
        -   open, release, rw<sub>page</sub>, ioctl (sector num = block num)


<a id="orgaf75648"></a>

## device driver flow

-   get io request (top half: read write etc)
-   check if result is in buffer (ie recent keypress)
-   if is goto store and return request
-   else send or queue IO operation and sleep if needed
    -   device hardware does thing and wakes the thread
-   Trap handle bottom half
-   get interrupt then:
    -   update buffers,
    -   wake up thread, if needed
    -   send more to device, if needed


<a id="org2b8ff0d"></a>

## device files

-   xv6 has num for each one (device number)
-   linux/unix has major + minor number
-   ex xv6 top half read:
    -   acquire lock
    -   while(num to read > 0)
    -   while(readloc == writeloc (ie no data)
        -   sleep
    -   copy in to users buffer, dec num to read
-   bottom half:
    -   trap handler if kbd interrupt
    -   handle keyboard then signal done


<a id="org482a529"></a>

## how are devices connected to processor

-   device controller on memory bus
-   controller have control registers with memory addresses
-   reading or writing to a control register can cause something to happen
-   buffers and queues also exist on device controller that have memory addr
-   device controller signals interrupt controller (please call interrupt)
-   device controller is like memory that does extra shit
-   sometimes theres a bus adapter that connect another adapter that has a device controller eg usb


<a id="org4b38ea6"></a>

## devices and caching?

-   please dont cache device buffers
-   therefore os needs to mark mem as uncachable
-   x86 has a bit in PTE that says no caching
-   x86 has an i/o address, which are used for some addresses
    -   because historically there is separate io buss
-   xv6 keyboard interrupt handle:
    -   st = inb(KBSTAT)
    -   if status says data in buffer return -1
    -   else data = inb(KBDATA) &#x2013; reads from data and clears buffer


<a id="org282dfa0"></a>

## programmed i/o

-   direct memory access, why should i move my mem to proc to real memory
-   why not just use me as memory (device controller)
-   where is your buffer os? os chooses location, device writes to real memory
-   OS could chose location to be where it wants it (only one copy) much faster
-   even in parallel cus device controller does it :)
-   also gives device controller huge buffer (saves money)
-   downside: dma requires physical addr as device dont have page table
-   downside: if device messes up, can overwrite arbitrary memory
-   therefore have IOMMU
    -   io memory management unit
    -   helpful for virtual machines
    -   not super common
    -   page tables for devices

