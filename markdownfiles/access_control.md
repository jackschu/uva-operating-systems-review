
# Table of Contents

1.  [Access control](#orgc51ee5a)
    1.  [Protection/Security](#org30a6096)
    2.  [Authorization / Authentication](#org99b5f42)
    3.  [How does login work?](#orgad32dfb)
    4.  [sudo](#org7462eba)
        1.  [setuid bit](#orgcbbbfb6)
    5.  [Different UIDs real/effective/saved](#orgf91e493)
    6.  [Ambient authority v capabilities](#org94b2d23)
    7.  [Sandboxing](#org89f60f2)


<a id="orgc51ee5a"></a>

# Access control


<a id="org30a6096"></a>

## Protection/Security

-   protection: mechanisms for controlling access
    -   page table, preemptive scheduling, encryption etc
-   security: using protection to prevent misuse
    -   misuse is represented by policies
-   we're gunna talk about mechanisms rather than policies
-   Adversaries
    -   do the worst possible thing
    -   and can be clever


<a id="org99b5f42"></a>

## Authorization / Authentication

-   authentication is who is who
-   authorization is who can do what (our focus)
-   access control matrix (big ol table)
    -   domains that users belong to
    -   files and processes have r/w/x for each domain in table
-   could store as access control list (store with file not in table)
    -   assign processes to protection domains
    -   give processes user /group that is ruining em
    -   object has access based on user/group
    -   posix has uids (user ids) theres an effective userid
    -   also gid / effective gid, can get a list of groups
    -   these are numbers (mappings to names are somewhere)
    -   groups can be programmatic 
        -   like add to group if ssh'd to particular machine
        -   eg video group access for monitor
-   or "on the side" eg apparmor via linux
-   posix file permissions has r/w/x for uid, gid, and for others
    -   p inflexible
-   also theres an "access control list" (note the default one is also an acl)
    -   more flexibility, actual list, more fine grain
    -   posix says user rules > group rules
-   auth checked on system call (eg open, kill etc) dont rely on libraries
    -   otherwise could just do syscall to avoid auth
-   UID 0 is root / superuser, can bypass (almost) all permission checks


<a id="orgad32dfb"></a>

## How does login work?

-   check if password correct
-   changes uid
    -   setuid() works in UID 0 (maybe doesnt otherwise maybe does)
    -   system starts login program as superuser
-   runs shell
-   unix password storage *etc/shadow*
-   also could have a network service with encrypted password
-   must be run with UID 0, and run semi automatically


<a id="org7462eba"></a>

## sudo

-   run command with superuser permissions
-   started by non-superuser (how?)
-   there is a metadata bit on executables (set-user-id)
-   so we just make sudo exe owned by root and set this bit


<a id="orgcbbbfb6"></a>

### setuid bit

-   if true then syscall changes effective uid to owners uid
-   this acts as gate to higher privilege
-   allows you to make auth decisions outside the kernel
-   super useful, eg change pass, mount USB stick, bind to port < 1024
-   eg have a printer user with appropriate executables
-   however this stuff is v hard to write, there are v clever workarounds
-   eg cant just pass file to printer to open, might just pass prot password file
    -   broken solution: if og user can read it then read it?
    -   fails b/c could change what the filename is pointing to and then race cond would allow access
    -   this is called TOCTTOU (time to check to time to use) problem
    -   solution: temporarily become original user


<a id="orgf91e493"></a>

## Different UIDs real/effective/saved

-   effective - determines permissions (ie changed with sudo)
-   real - users who started prog (ie not changed with sudo)
-   saved - user id from before last exec
    -   no standard function,
    -   saved when set-uid-prog starts
-   process can swap or set effective UID with saved UID
-   setuid sets all of them
-   seteuid sets effective
-   setreuid sets real (sometimes ish)
-   all these things exist as GID also set-group-id executables


<a id="org94b2d23"></a>

## Ambient authority v capabilities

-   ambient: theres permissions somewhere, go get em
    -   ie we check whether or not x user can access y thing
-   capabilities: tokens to do things
    -   ie we the file itself has a key assoc with it
    -   eg fd, or page numbers kinda as tokens
    -   inherited by spawn programs
    -   can be sent over local sockets or pipes
    -   ex capsicum
        -   no global names
        -   fd have r/w/x/kill permissions


<a id="org89f60f2"></a>

## Sandboxing

-   idea run code that we dont trust in a sandbox
-   doable because we don't need user's full permissions
-   happens in browsers :D first major thing by Google Chrome
    -   sandbox the rendering engine
-   with ambient: create new user with few privilege?
    -   not great because default user can do too much
    -   also creating new user is requires sysadmin perms
    -   dont want to create users for every app
-   with capabilities: just discard most capabilities
-   alternative: filter all system calls
    -   linux allows this through seccomp()
    -   strictmode: only allow read/write/exit/sigreturn
        -   can on read/write what it already has open
-   hope to protect from bugs as well as adversary

