
# Table of Contents

1.  [Distributed Systems:  multiple machines on one task](#org2a13078)
    1.  [Models](#org2f4247c)
        1.  [client-server model](#orgaadee17)
        2.  [peer-to-peer model](#org6b0ae42)
    2.  [why dsitributed](#org4b3eae9)
    3.  [mailbox model (abstraction)](#org8fc26f9)
        1.  [connections](#orgeff2c3b)
    4.  [DNS/addresses](#orgf3d1e60)
    5.  [Protocol](#org0822b91)
    6.  [Sockets](#org97365dc)
        1.  [Incomplete Writes](#orgf86aa9b)
        2.  [Read and write at same time](#org5cb9559)
    7.  [Remote Procedure Calls (RPCs)](#org7c0ccd5)
        1.  [Interface Description Language IDL](#orga697d82)
        2.  [gRPC](#orga983212)
        3.  [Failure to be transparent](#org1f155da)
        4.  [RPC locally](#org04b84f9)
    8.  [Network Filesystems](#orgfed7d86)
        1.  [NFS version 2 (RFC 1094) 1989](#org92bd4d9)
        2.  [Performance improvement NFSv3](#org7efe718)
        3.  [Andrew File System v2](#orgd5a8c0d)
    9.  [Dealing with network failures](#org2c4c764)
        1.  [distributed transaction](#orgbfa7d8b)


<a id="org2a13078"></a>

# Distributed Systems:  multiple machines on one task


<a id="org2f4247c"></a>

## Models


<a id="orgaadee17"></a>

### client-server model

-   centeral point of failure, but simple
-   client: sometimes on, server: always on
-   server never initiates
-   layers of servers happen often
    -   webserver talks to cache talks to app server talks to login server


<a id="org6b0ae42"></a>

### peer-to-peer model

-   no always-on server
    -   hopefully no scaling issues
-   any machiine talk to any machine


<a id="org4b3eae9"></a>

## why dsitributed

-   multiple owners collaborlating
-   delegation of responsability
-   combine cheap machines
-   easier to add incrementally
-   allows redundancy


<a id="org8fc26f9"></a>

## mailbox model (abstraction)

-   its an abstraction (send/receive message)
-   machine A to newtwork to machine B
-   network has a queue of messages not yet recieved by recieving program
-   how to reply to client as server? where did msg come from?
-   how to receive multiple messages (multi part messages)
-   can build with mailbox idea
    -   send return addr
    -   send message as parts
-   abstraction that does all this is a connection


<a id="orgeff2c3b"></a>

### connections

-   two-way channel for messages
-   extra operations: connect(request to a machine), accept(a connection)
-   this is how the real internet works
-   connections ~= two-directional pips
    -   In fact same API in POSIX
-   how to specify machine?
    -   name: logical id eg www.va.edu or notes.txt
    -   address: location eg 128.143.22.36 or inode # 31474
    -   conversion done through DNS (domain name system)


<a id="orgf3d1e60"></a>

## DNS/addresses

-   dns is a distributed db
-   my machine -> isp dns server -> root dns idk try edu -> idk try va.edu -> idk try cs.va.edu
-   we cache this addr (let cache expire after week or so)
-   IPv4 is 4 32 bit number
    -   authority gives ip addr ranges to different people (eg uva, google)
-   router sends data for x addr to y network
-   special addr
    -   127\* is localhost (loopback)
    -   192, 10, 172 are private ips (good for local speed) network addr translation
    -   169 'never forwarded' by routers link-local
-   network addr translation
    -   convert many private addr to one public ip
-   IPv6
    -   ::1 is localhost
    -   :: is arbitrarily many 0s
    -   6 128-bit numbers,
    -   b/c so many: maybe no need for network addr translation?
-   port number (16 bits)
    -   PO boxes for each addr
    -   80 http 443 https 22 ssh
    -   49152-onward os uses whenever


<a id="org0822b91"></a>

## Protocol

-   agreement on how to communicate
-   ex IP UDP TCP HTTP TLS SSH SCP SFTP HTTPS FTP
-   stateful
    -   what previously happens matters
    -   ex FTP
-   status codes are typical


<a id="org97365dc"></a>

## Sockets

-   kinda of file descriptor POSIX
-   2 directional pipe
-   read/write can be used on socket like file
-   client socket
    -   connect()
-   server socket
    -   listen(for new connections)
    -   accept()
    -   something with a fd that we use to get more fds
-   connections are stored as a 5 tuple
-   there is addr and port for client and server (set through bind())
-   client server flow
    -   server: creates server socket, binds to host:port, start listen
        -   accept a connection fork:, in loop(read request, write response)
        -   close connection
    -   client: connect to server hostname:port (gets assigned local host:port)
        -   in loop(write request, read response)
        -   close socket
-   socket addr is a struct
    -   getaddrinfo() creates this conveniently (can take string of domain name)
        -   returns a linked list of possibilities
        -   older way to do it is gethostbyname()
        -   the hint: AI<sub>PASSIVE</sub> tells we're making a server socket
            -   useful for binding to all hostnames I can
    -   has: family, port, addr
    -   port and addr are in network byte order
    -   network byte order means big endian
    -   other utils
        -   getsockname getpeername getnameinfo


<a id="orgf86aa9b"></a>

### Incomplete Writes

-   writes might write less than requested
    -   eg buffer full or wifi goes down
-   read might less than requested
    -   not enough data got there in time
-   therefore write fully or fill buffer or readline funcs are needed


<a id="org5cb9559"></a>

### Read and write at same time

-   one solution: threads, one to read one to write
-   other solution event loop with nonblocking i/o functions eg select, poll


<a id="org7c0ccd5"></a>

## Remote Procedure Calls (RPCs)

-   I write a bunch of functions, I want to call them from another machine
-   hope that this is more transparent, 
    -   b/c no difference b/w remote-local function calls
-   typically implemented with stubs
    -   a stub is a wrapped function that stands in for other machine
    -   to do remote procedure, just call a stub
    -   to implement remote procedure, another stub function calls you
    -   client stub
        -   generated by compiler like tool (THRIFT)
        -   contains wrapper function,
        -   converts args to bytes and bytes to return value
    -   server stub
        -   also generated by compiler
        -   converts bytes to args, and return value to bytes
    -   stubs conncected via network and rpc library on each side of network
-   RPC's have a field called context
    -   specify where the function actually is
-   we can hide a surprising amount of this detail with OO design
-   marshalling or serialization
    -   cant just copy the bytes cus like what is char\*?
    -   what about 32 vs 64 bit and endianness
    -   note marshalling is useful even if not rpc (ex protocol buffers)


<a id="orga697d82"></a>

### Interface Description Language IDL

-   tool/library needs to know what remote procedures exist and what types they do
-   this is specified by RPC server author in IDL
-   compiled into stubs and marshalling / unmarshalling code
-   why not just give a header file?
    -   what does my pointer point to?(size)
    -   portability, size on different compilers
        -   we want machine-neutrality and language-neutrality
    -   size of array?
    -   what about if we modify an argument by reference
    -   different versions of RPC across client/server
-   eg: protocol dir protocol { 
    -   1: int32 mkdir(string);
    -   2: int32 rmdir(string); }
-   uses id number so we can change name and thats fine, also save space
-   fields also numbered


<a id="orga983212"></a>

### gRPC

-   every single function takes one message returns one message
    -   make a struct/class thats a message
-   server implementation
    -   gets context, arg message, return message as pointers
    -   modify return message instead of returning
    -   actually return the status
-   versioning
    -   messages have field numbers
    -   rules allow adding optional fields
        -   if extra field its ok
        -   if missing optional its ok
    -   if need to change required fields, make new method
    -   or just make api v2 a whole new protocol


<a id="org1f155da"></a>

### Failure to be transparent

-   setup is not transparent (ie what server/port) wish we didnt have to spec
-   errors might happen
    -   yes errors happen on local, but there are unique remote errors
        -   like what if the server no longer has that function
-   server and client can fall out of sync (version wise)
-   performance is v diff local vs remote (nano vs micro/milli)
-   leaking resources? 
    -   what if client opens a file and then crashes
    -   does the server keep it open?


<a id="org04b84f9"></a>

### RPC locally

-   can use locally on one machine
-   convenient alternative to pipes
-   allowed shared memory implementation to get speed back


<a id="orgfed7d86"></a>

## Network Filesystems

-   department machines always have your stuff how? theres a fileserver
-   RPC calls for open read list etc
-   just turn syscalls into RPC calls?
    -   doesnt quite work (but good high level idea)
    -   what state to store on the server?
        -   open fds on each client? pretty expensive
        -   soln: client stores more stuff
    -   what if client crashes?
        -   server cant really know
    -   what if server crashes?
        -   what do clients do in the meantime?
        -   client tracks more info to
    -   performance
        -   milliseconds instead of microseconds


<a id="org92bd4d9"></a>

### NFS version 2 (RFC 1094) 1989

-   bunch of RPC lookup getattr read write create remove
-   common datatype is file ID 
    -   = device + inode number + "generation number" (concat all)
-   client specifies file ID
-   client gives file ID, if client crashes, server doesnt care
    -   called fate sharing, the mapping is 'deleted'
-   server converts fileids to files on disk
    -   server doesnt get notified unless client is using the file
-   generation number 
    -   incremented every time inode is reused
    -   so this prefents opening wrong file if inode num still valid
    -   stored in inode
-   this is a stateless server protocol, no open/close/etc
-   read dir takes optional cookie and returns next cookie
    -   pattern: client stores tokens it cant read but sends to server
    -   very cool!
-   cons: 
    -   performance
        -   every read/write goes to server,
        -   want: cache things to read in client
        -   if only one user of file want: cache writes and write later
    -   offline operation not doable


<a id="org7efe718"></a>

### Performance improvement NFSv3

-   caching copies causes lots of consistency problems
-   gotta tell server about every update write away? means no caching
-   solution: allow inconsistency (some kinds only) :(
-   we probably expect this inconcsitency anyways
-   we want open to close consistency
    -   opening a file checks for updated version
    -   closing a file writes updates from the cache
-   really bad inconsistency can still happen though 
    -   eg open file, change it a day later, save it
    -   fix: check server/write to server periodically


<a id="orgd5a8c0d"></a>

### Andrew File System v2

-   a stateful server
-   works file at a time, not parts of file
-   still uses consistency compromise (for small speedup)
    -   still wont support simultaneous read/write from diff machines well
-   always read/write entire file
-   checkout/cache entire file ( can be performance hit but can be fine )
-   last writer wins
    -   adv over last writer wins per block on NFS
-   when file is opened, registers callback to server
    -   when file changed, callbacks are called to inform client
-   on server failure, just recontact when it comes back up and re register callbacks
-   on client failure, fine, just maybe inconsistency issues, client can ping server every so often
-   could support offline operation
    -   just have to deal with merging on reconnect
-   Coda FS: based on this idea offline allows file resolution
    -   specify what to cache with version ids
    -   has issues (what to do when conflict)


<a id="org2c4c764"></a>

## Dealing with network failures

-   no real great way to do this, maybe the 'dont need to retry' msg doesnt make it
-   try2 (idemptotent requests):
    -   req: do thing
    -   resp: ok did it (failed msg)
    -   req: do thing if you havent
    -   resp: ok did it
    -   but how many times to retry
-   connection in posix knows if got data (not if did right thing with data)
-   types of failure
    -   fail-stop - stop responding
    -   Byzantine failure - do the worst possible thing (eg "i did it" but did not)
-   we assume fail-stop
-   recover when machine comes back up
    -   doesnt work for byzantine
-   require a quorum of machines to working
    -   require 1 extra machine to handle fail-stop
    -   requires 3f + 1 to handle f byzantine failures


<a id="orgbfa7d8b"></a>

### distributed transaction

-   goal: we all do it or we all dont
-   centraliized solution?
    -   machine d maintains redo log for all machines
    -   machine d treats all others as data storage
    -   but uh not a distribute system
-   goals: only coordinate when transaction involves more machines
-   goals: only coordinate with involved machines
-   goals: scales to many many machines
-   idea: persistent log - same mechanism of redo log
    -   machine remembers what happens on failure
    -   what if fail during log?
-   two phase commit:
    -   every machines votes to commit - we do it else dont
    -   phase 1: voting
        -   agree to commit: promise I will accept
        -   agree to abort: promise I will NOT accept
        -   (MUST KEEP PROMISES) even if viewpoint changes later
        -   record in mach log
    -   phase 2: finishing
        -   learn all machines agree to commit: commit
        -   learn one+ machine said abort: abort
        -   do the thing
        -   record decision in log
        -   if unsure of result? ask again
    -   this is pretty blocking, if a machine never comes back, we never run
-   two phase commit roles
    -   coordinator or worker
    -   c -> w prepare
    -   w -> c vote commit or vote abort
    -   c -> w global commit or global abort
-   four coordinated states INIT, WAITING, ABORTED, COMMITED
    -   resend prepare messages periodically
    -   workers also resent vote periodically
-   coordinator failure recovery
    -   log indicating last state
    -   log before sending any messages
    -   on boot back up, resend appropriate transition message
    -   note crash while in wait -> send abort to all
-   worker states: INIT, ABORTED, AGREE TO COMMIT, COMMITTED
    -   also log last state, and resend
-   quorums: i need x number to say yes to do it
    -   quorums must overlap
    -   therefore someone in majority must know about every operation
    -   therefore no quorum will agree with something inconsistent with past
    -   vary quorum based on operation type?
    -   eg 5 mach, 4 for update, 2 for read
    -   details v tricky
-   Raft(quorum implementation) (v complicated, this is the high level view)
    -   leader election
        -   elect new leader on failure
        -   cant be leader if not up to date
        -   quorum must elect each leader (lmao)
        -   nodes on believe in latest (highest numbered leader)
    -   leader uses other maches as remote logs
    -   leader ensures quorum logs operations
-   what about byzantine failures?
    -   supermajority needed 3f+1
    -   nodes can give inconsistent votes
    -   just overlap is not enough
