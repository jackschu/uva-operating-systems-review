
# Table of Contents

1.  [Filesystems](#orgbee5682)
    1.  [hard drive interfaces](#org2790187)
    2.  [FAT file system](#org6fcd983)
    3.  [hard drive operation / performance](#org726b891)
    4.  [SSD](#org1a4a8c5)
    5.  [xv6 filesystem](#orgf63decd)
    6.  [linux inode](#orgfad2770)
    7.  [indirect block adv](#orge2e7bb2)
    8.  [sparse files](#orge9e717e)
    9.  [links](#org50182bb)
    10. [fast file system](#org0de3085)
    11. [NOTE ALL PREVIOUS REFERENCES TO LINUX FS ARE ABOUT ext2](#org1e1768d)
    12. [non-FFS solutions](#org65b2d2f)
    13. [FAT IRL](#org08a4966)
    14. [reliability](#org4d8fea1)
        1.  [ordering](#org11202f8)
        2.  [beyond ordering (logging)](#org283da4d)
        3.  [snapshots](#org1aeb1f9)
    15. [multiple file systems?](#orgb011eeb)
    16. [aside: fsync](#org215ea8f)


<a id="orgbee5682"></a>

# Filesystems


<a id="org2790187"></a>

## hard drive interfaces

-   work with sectors (512 ish bytes)
-   typically want to write more than one sector (4k)
-   given file, where is its data, which sectors which parts of sectors
-   would like sector reads to be contiguous
-   given a dir what files are in it
-   what are a thing's metadata
-   where to put new data for file/ growing file/directory


<a id="org6fcd983"></a>

## FAT file system

-   File Allocation Table
-   divided into clusters (1 or more sectors)
    -   recall sector = min amount hw can read
-   array (index = clus number) (data  = next cluster) (-1 indicates end)
-   start locations
    -   want filenames
    -   start locations, size, filename store in directories (also files)
    -   directories have a list of entries (directory entree is fixed size)
    -   directory entree attr field indicates long dir or short (reg) dir
        -   indicates type of file
    -   0x00 as first char of name means end of directory
        -   0xE5 means 'hole' from deletion
-   header has metadata about system
    -   where is root dir
    -   size of sec, sec per clus, reserved sec, number of fats, etc
-   create file
    -   add dir entree
    -   choose free clus nums (they're marked 0 in FAT)
    -   may need to grow num clus for dir
-   Pros
    -   simple
-   cons
    -   wasting space with partial cluster use
    -   finding free space is linear
    -   deleting file is linear
    -   seeking to byte x of file is linear
    -   searching a directory is linear


<a id="org726b891"></a>

## hard drive operation / performance

-   platters and arm move in sync, center of hd is faster access
-   store related things together plz
-   there are cylinders (tall) that make up tracks (circle on one disk)
-   delays
    -   seek time - 5-10ms head to cylinder
    -   rotational latency 2-8 ms platter to sector
    -   transfer time 50-100+ mb/s
-   historically: OS used to know cylinder/head/track/ but now doesnt
-   hd controllers remap bad sectors, detected with err correcting codes
-   r/w ordering strategies (done by disk controller not os):
    -   disk scheduling, read/write queuing
    -   shortest seek time?, take request with least seek time (SSFT)
        -   can trap head on one side of disk (starvation on read)
    -   fairness? SCAN
        -   zig zag back and forth (may favor center)
    -   C-SCAN
        -   out to in, reset to outside repeat
-   controller has a cache too, can hold data waiting to be written


<a id="org1a4a8c5"></a>

## SSD

-   big ol controller
-   no moving parts, read sector-like sizes (pages) eg 4k or 16kb
-   can only erase in blocks
-   can only erase blocks order of 10,000-100,000 times
-   moves things around to create erasure blocks
-   wear leavening - spread writes out
-   Flash translation layer, maps os sector to our sectors
-   write at writehead,
-   garbage collection: copy partially written blocks to writehead to consolidate free space
-   operation: TRIM, drop this sector, good for os to help SSD


<a id="orgf63decd"></a>

## xv6 filesystem

-   similar to modern unix
-   superblock = header
    -   size, nblocks, ninodes, nlog (blocks), inode start,
-   inode - file information thingy
    -   type (dir file device) (type = 0 means free)
    -   major/minor (if device)
    -   nlink
    -   size
    -   addrs (array of addr of data blocks)
    -   names stored in directory file
-   free block map
    -   1 bit per data block 1 if available
    -   contiguous 1s = contiguous blocks
-   directory entry struct dirent
    -   index of inode and a name
    -   each dir referencing an inode is called a hard link
-   adv over fat
    -   reliability via logs
    -   easier to scan for free blocks
    -   easier to find kth block
    -   file type/size stored in inode together
-   cons
    -   inode block map is far from file data so long seek time
    -   choice of file/dir block is bad, so file dir entrees end up scattered
    -   blocks are small compared to metadata size (or large wasting space for small files)
    -   linear search directory entrees to resolve paths
-   the array of addresses of blocks is 13 long
-   the last entry is a pointer to an indirect block (a list of blocks)
    -   which is a block of direct blocks (256 blocks)
-   6144 bytes w/o indir
-   262144 with indir


<a id="orgfad2770"></a>

## linux inode

-   more metadata
-   more indirection, (15 block pointers)
-   [12] is indirect
-   [13] is double indirect
-   [14] is triple indirect


<a id="orge2e7bb2"></a>

## indirect block adv

-   it is a tree so logN time to access
-   little over head for more overhead


<a id="orge9e717e"></a>

## sparse files

-   theres a special none value for these block pointers if block is all 0s


<a id="org50182bb"></a>

## links

-   every file can have multiple links
-   track number of links delete files with 0 links
-   open files are links too
    -   so can do trick: create, open, delete (doesnt disappear until close)
-   ln -> link()
-   rm -> unlink()
-   softlinks / symbolic links
    -   ln -s og new, reference a file by name
    -   so if delete og, then new is gone too


<a id="org0de3085"></a>

## fast file system

-   made by Berkeley, linux is based on this
-   address: inode block map far from data, bad choice of file/dir data blocks
-   block groups
    -   split disk into block groups, each of which is like mini file system
    -   therefore data is close to inodes
    -   inodes from one block group can point to data in another block group
        -   but prefer not to
    -   also keep free map in block group,
    -   makes lower seek time within directory
    -   block groups are designated for specific directories
-   find free block by first free blocks within block group
    -   hope that for large file this ends up being contiguous
-   deliberately under utilize disk
    -   maintain 10% of blocks listed free
    -   that way we can take into account percent full of each block group
    -   use this in some complicated way
-   writes back only once evicted from cache
-   how to deal with small files? (50%+ are less than 1kb)
    -   the last block in a file can be a fragment
    -   eg each block can be split into 4th of frag
    -   extra field in inode to indicate frag
    -   allows one block to store several fragments


<a id="org1e1768d"></a>

## NOTE ALL PREVIOUS REFERENCES TO LINUX FS ARE ABOUT ext2


<a id="org65b2d2f"></a>

## non-FFS solutions

-   NTFS or linux's ext4
-   extents: large file? store in one big chunk, a start block + size
-   how to pick where? how to seek to parts?
-   allocation:
    -   first in block group doesnt work well
    -   so we scan block map for best fit
    -   choose smallest chunk of free blocks that fits
    -   worst fit (ie largest chunk always) also works well
    -   either way allocation is p slow now but ok
-   efficient seeking
    -   store a tree, of where to start seeking from
    -   find the the extent that the input byte number is in
    -   non-binary search tree (as a key value store efficient for disks)
        -   x breakpoint numbers,
        -   x + 1 children ( less than first bp num then follow first child etc)
        -   each node is one block on disk
        -   note wider than binary tree means better for disks since less indirection
        -   removes linear search for find offset x
            -   store index by offset of extent within file
        -   removes linear search for file in dir
            -   index by filename


<a id="org08a4966"></a>

## FAT IRL

-   not that bad cus these days just load the FAT into RAM


<a id="org4d8fea1"></a>

## reliability

-   is the data there?
-   is the filesystem in a consistent state?
-   eg fat stores multiple copies of metadata
-   inode fs often store multiple superblocks
-   (redundancy for metadata)
-   mirror whole disks? with entire backup disk
    -   actually p good
    -   can also read from either disk (faster)
-   RAID 4
    -   disk 1 xor disk 2 = disk 3
    -   writes now require two writes and one read
    -   can also have more disks
    -   4 disks? one write now requires two writes and two reads
    -   n disks? 2 writes and n-2 reads (effect canceled by conseq sector writes)
    -   parity disk is used much more often
    -   RAID 5 can shuffle which disk is parity disk for each sectors
    -   RAID 6 can handle 2 disk loss at a time
-   ZFS implements raid-like redundancy on its own
-   note we havent talked about how RAID handles time b/w updating disk and updating parity disk


<a id="org11202f8"></a>

### ordering

-   Fat Power loss during file creation? ordering?
    1.  put data in file clusters
    2.  create new dir entree
    3.  update fat for file
    4.  update fat for directory
    5.  philosophy: best to not make things valid before they are
        -   even if waste data
-   xv6 FS ordering
    1.  free block map for new file block
    2.  file data block
    3.  new file inode
    4.  free block map for new dir block
    5.  new dir entry for file (in dir block)
    6.  update directory inode
    7.  phil: better to waste space than point to bad data
-   recovery: fsck (chkdsk on windows)
    1.  read all dir entrees
    2.  scan all inodes
    3.  free unused inode (unused = not in dir)
    4.  free unused data blocks (unused = not in inode lists)
    5.  scan dirs for missing update/access times
-   ex: unlink (rorder for removing hard link)
    1.  overwrite directory entree
    2.  decrement link count in inode
-   ex: unlink last link
    1.  overwrite last dir entree for file
    2.  mark inode data as free
    3.  mark inode as free
-   ordering sucks for disk speed though


<a id="org283da4d"></a>

### beyond ordering (logging)

-   recall updating a sector on disk is atomic
-   transaction: bunch of updates that happen at once
    -   redo logging
        -   in log: mark begin (stuff im prob gunna do VV)
        -   write what youre gunna do
        -   mark commit (i am now promising to do this^)
        -   start apply log to disk
        -   when commit reached, clear log
    -   if crash before commit, just dont do it
    -   if crash after commit, just redo it (in-case it wasnt done)
    -   consistency b/c commit message and stuff w/in one sector
-   logged things should be idempotent (ok to do twice)
-   redo logging file systems called journaling filesystems
-   xv6 has one journal
    -   with log header(one sector) num of blocks, locations for blocks
        -   we check if non-zero on boot^ if so redo transaction
    -   data of transactions (blocks) ('what im gunna do' part)
    -   only commit when no active file operation or not enough room in log for more
-   faster to do a large transaction
-   problems
    -   log size - garbage collect log in background
    -   writing everything twice - makes it expensive?
        -   ok b/c saves time over careful ordering and good for ssds
-   degree of consistency
    -   can use log for metadata only
    -   or literally all data (every bit written gets written twice)


<a id="org1aeb1f9"></a>

### snapshots

-   keep old versions of files around
    -   done through copy on write
    -   make a new (current) inode with copy on write of file data
    -   old inode lives on with copies of old data
-   dont want to copy new whole inode array for every version
-   so we split inode array into pieces with a root inode pointing to pieces
-   then the pieces are copy on write with a copy of root inode
-   array of root inodes is the history
-   copy on write also avoids the cost of logging since avoids copying


<a id="orgb011eeb"></a>

## multiple file systems?

-   linux: virtual file system api (VFS)
-   to implement a filesystem have objects
    -   superblock (header)
    -   inode (file)
    -   dentry (directory entree)
    -   file (open file)
-   common code handles dir traversal
-   common code handles file descriptors


<a id="org215ea8f"></a>

## aside: fsync

-   POSIX write the file to disk(ill wait)

