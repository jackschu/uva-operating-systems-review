
# Table of Contents

1.  [Virtual memory notes](#orgcdd040e)
    1.  [Last Time: deadlock](#orgf320311)
    2.  [Toy example](#org9588812)
    3.  [Two-level page tables](#org3300f61)
    4.  [The XV-6 way](#org9136ded)
    5.  [space on demand](#org2373419)
    6.  [copy on write](#orgee4ad7b)
    7.  [mmap](#org75ac49c)
    8.  [memory is a cache for disk](#org7f89ec9)
    9.  [Linux: forward mapping (file access)](#org0dd1551)
    10. [minor vs major page fault](#org9cfc8f4)
    11. [on miss](#org4004ee4)
    12. [physical pages are manged](#org99fca06)
    13. [evicting page](#org49c80ee)
        1.  [working set model ](#org1d15285)
        2.  [aside zipf model ](#org4e229e3)
        3.  [lru model ](#org89bf254)
    14. [Lazy replacement or not](#orga9f5999)
    15. [caching for files?](#orgcaba1ce)
    16. [proactive loading](#org47a7d8a)
    17. [thrasing](#org1353610)
    18. [fair paging replacement?](#org6fb2326)


<a id="orgcdd040e"></a>

# Virtual memory notes


<a id="orgf320311"></a>

## Last Time: deadlock

-   Prevent deadlock
    -   Stop sharing
    -   Stop waiting
    -   stop holding
    -   Useful one: consistent order of acquiring
-   deadlock detection
    -   dependency graph to detect
    -   single resource: look for cycles
    -   mutli-resource: imagine threads finishing
    -   thing called Baker's Algorithm: if it will cause deadlock: dont start


<a id="org9588812"></a>

## Toy example

-   10 bit addr: stack -> empty/heap idk -> data/heap -> code
-   virtual pages are fixed size (in this class)
-   each virtual page is 2<sup>8</sup> bytes in this case one for stack data/heap code etc
-   therefore last 8 bits are the page offset
-   ppn can be different number of bits than vpn its max-8 in this case
-   trigger exception if valid bit not set
-   Page table is stored in memory with physical memory addr


<a id="org3300f61"></a>

## Two-level page tables

-   What we actually use
-   ex 2<sup>20</sup> pages total, 2<sup>10</sup> entrees per table
-   invalid bit on in first level means large gap (good)
-   ppn of first level page table is where to find next level table
    -   has to be multiplied by page size
-   user bit - true if can access in user mode


<a id="org9136ded"></a>

## The XV-6 way

-   Page Directory Index PDX is first level page table index
-   Page Table Index PTX is the second level table index
-   Page frame is a physical page (intell)
-   kernbase = 0x8000000 , above it is kernel only memory
-   every physical addr in xv6 has two virtual addrs, one for ker, one for user
-   P2V and V2P map kernel virtual to physical (!not user virtual)
-   kalloc / kfree alloc free whole page from linked list
-   sbrk() sets break between heap and invalid memory


<a id="org2373419"></a>

## space on demand

-   useful to telling program they have huge RAM
-   trick: push decreases rsp first so uhh invald page num might be lower
-   xv6 checks bounds using sz


<a id="orgee4ad7b"></a>

## copy on write

-   use case: fork, note some parts are not writable so dont even need copy
-   also a lot will not be written for a while hence COW
-   tell page table that shared part is read-only (both parent and child)


<a id="org75ac49c"></a>

## mmap

-   map a file to spot in memory
-   eg:
    -   int file = open("somefile.dat", O<sub>RDWR</sub>);
    -   char \*data = mmap(&#x2026; , file, 0);
    -   data[100] = 'x';
-   options: 
    -   prot: PROT<sub>READ</sub>, PROT<sub>WRITE</sub>, PROT<sub>EXEC</sub>, what you're allowed to do
    -   flags: MAP<sub>SHARED</sub> - changes are linked, MAP<sub>PRIVATE</sub> - changes cause cow
    -   addr: suggest (not strict) where to put mapping
-   linux uses this to load executable (cool)
-   mapped pages may be initially preloaded or invalid (assume invalid)
-   mapped (read only) on invalid access: find cached (or load cached) pg and retry
-   mapped (r/w shared) on write: invalidate data on disk, when cache is freed update disk
-   mapped (cow) on write: protection handler is called and copy created (pg table update)
-   mapped pages can exist with no backing file: alloc on demand, swap out causes temp storage in disk
-   copied data created by COW can also be swapped out to disk temporarily like above
-   swapped data (swap files) are becoming outdated b/c of size of ram
-   swapping is approx mmap with default files (using disk as backup)


<a id="org7f89ec9"></a>

## memory is a cache for disk

-   cache block = physical page
-   fully assoc
-   replacement is managed by os
-   normal hit is not managed by OS
-   virtual addr on cache hit: page table gets accessed (no os needed)


<a id="org0dd1551"></a>

## Linux: forward mapping (file access)

-   process control block (task<sub>struct</sub>)
    -   has a page table
    -   has mmap region info
        -   points to open file info
    -   has open file info (struct file)
        -   which has inodes (which has physical pages for files)
        -   ie inodes have addr space which has a tree of pages
    
    -   for file access waits for page fault then updates page table


<a id="org9cfc8f4"></a>

## minor vs major page fault

-   minor page fault: page already in pg cache
    -   just fill in pte
-   major page fault: not cached, need to allocate


<a id="org4004ee4"></a>

## on miss

-   file: OS data structure based on filesystem
    -   filesystem is black box which gives pys addr
-   virtual address (if part of file): use linux mmap regions
-   swapped out pages get location stored in unused region of invalid pte
    -   note that this is elegant because pte only gets overwrote if
    -   virtual addr is rewrote, meaning that we dont care about old thing
    -   recall you can have a full cache without having a 'full' page table
        -   full page table doesnt really happen


<a id="org99fca06"></a>

## physical pages are manged

-   how to find free physical page to use
-   track usage (linux uses a list of LRU pages) (a lil more complex than this)
-   physical page might need to be evicted to disk, how do we do that
    -   we also need reverse mappings to avoid: (on physical page eviction)
        -   invalid pte mapping to nonexistent physical page
        -   invalid file pointing to physical page
        -   for file pages linux does:
            -   every physical page has a pointer to address space
            -   has pointer to mmap region info
            -   has pointer to process control block
            -   has pointer to page table
            -   so that given ppn, we can remove/ invalidate references to that page
        -   for non-file pages linux:
            -   every page ihas linked list of mmap regions
                -   list is important b/c makes fork easy
            -   which points to mmap region info
            -   points to proc control block
            -   points to page table


<a id="org49c80ee"></a>

## evicting page

-   find victim page to evict (do so with linux mechanism above)
-   if dirty/changed save to disk
-   how to choose? goals:
    -   hit rate (min num of misses)
        -   best possible replace furthest accessed in future
            -   best assuming no reading in advance
            -   called belady's min also impossible
        -   cant to belady so look for patterns
        -   working set model [more here](#org5fb96e2)
            -   motivation: program usually uses a subset of its memory
            -   phase change: discard current working set, get another
            -   phase change results in spike of cache misses
            -   how to use it? one idea: split memory into working set/not
            -   if not enough space for all of working set, stop program do later
        -   LRU model [1.13.3](#orgc345b8a)
-   note hit rate not equal throughput
    -   since algo for choosing could be slow
    -   cost of miss could be dramatically variable (ie NFS vs HDD fetch)
    -   create zero page could happen on miss (very fast)
    -   grouping reads not considered in hit rate, but considered in throuput


<a id="org1d15285"></a>

### working set model <a id="org5fb96e2"></a>

-   motivation: program usually uses a subset of its memory
-   phase change: discard current working set, get another
-   phase change results in spike of cache misses
-   how to use it? one idea: split memory into working set/not
-   mayb if not enough space for all of working set, stop program do later
    -   reasonable because might be faster to do later
-   mayb LRU is likely not part of working set, so use that
    -   doesnt always work, there are bad cases (ie working set doesnt fit)


<a id="org4e229e3"></a>

### aside zipf model <a id="org26b5e8d"></a>

-   straight line on log-log rank v count access graph
-   long tail
-   lru is ok for this
-   what about changes in popularity?


<a id="org89bf254"></a>

### lru model <a id="orgc345b8a"></a>

-   no one does this exactly since requires page fault on every access
-   so we approximate LRU = "was this accessed recently?"
-   is it currently still being used?
    -   can do sw approach: mark it invald check if page fault happens
        -   update last known access if accessed/referenced
    -   can do hw approach: hardware sets a bit to 1 during access
        -   os periodically reads + record + clears access bit
        -   annoying if multiple processes has have 2 vpn to same ppn
        -   os need to clear + check all references
-   for files we can check if its been changed/ if swapped still valid
    -   sw: mark read only for a sec
    -   hw: dirty bit
-   second chance model:
    -   new pages on top of list, pull from bottom list
    -   if pulled page has accessed bit set, put back on top and reset
    -   else evict this page (made from top to bottom w/o access)
    -   cant actually thrash due to shuffling (good)
    -   note loading is referenced
-   SEQ model (active/inactive list) (linux move for non-file pages)
    -   new pages start in active list
    -   oldest active page is really an inactive page
    -   evict bottom of inactive page list
    -   if ANY inactive page is reference, moved to top of active
    -   this is basically second chance with more than one page in hot seat
-   CLOCK model - more general algo
    -   track pages over time window, periodically check for accesses


<a id="orga9f5999"></a>

## Lazy replacement or not

-   note all previous models are 'lazy'
-   os's dont want to wait unit memory is full to evict page
    -   be more proactive
-   why? (primary for avoiding data loss) what happens when power lost, good for performance
-   non-lazy eviction
    -   evict earlier in the background keep a free linked list
    -   to alloc, remove from free linked list (v fast)


<a id="orgcaba1ce"></a>

## caching for files?

-   LRU is trash for reading a file once
-   LRU is trash for reading a huge file x times
-   CLOCK-Pro: special casing for one-use pages
    -   files cant be protected in cache (by lru) until second access
    -   new files to top of inactive list
    -   if on inactive list and referenced once, idk ingore but track
    -   if on inactive list and referenced twice, move to top of active
    -   if on bottom of active list and referenced, move to top of active
    -   else if on bottom of active, move to top of inactive
    -   results in evicting if refrenced once or referenced mult but not recently


<a id="org47a7d8a"></a>

## proactive loading

-   do we have to load into cache on demand? do it ahead of time?
-   readahead: try and predict sequential reads
-   naive irony, if naively load ahead, we dont page fault anymore so
    -   we cant know if we should keep reading ahead (can force pg fault tho)
-   how much to readahead?
    -   read faster than prog needs but not by much
    -   read ahead x amount, set trap at x/2 ish, then read more if triggered


<a id="org1353610"></a>

## thrasing

-   always reading from disk, causes performance collapse


<a id="org6fb2326"></a>

## fair paging replacement?

-   share between users?
-   linux has cgroups
    -   each has high limit- cant use more than this many pages (replace thiers first)
    -   low limit - dont steal from this group when using less than this many
    -   max - actively deallocate if youre using

