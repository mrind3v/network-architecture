# Web Server Architecture: Apache → Java/FastCGI → nginx → Event Loops

## 1. The central historical problem

The slides are tracing one recurring systems-design question:

> What expensive resource are we allocating, creating, or reserving per request/connection that does not actually need to exist per request/connection?

The progression is roughly:

```text
CGI
  ↓
Stop creating a process/runtime per request.

Servlets / FastCGI
  ↓
Keep application/runtime alive and reuse it.

Apache prefork
  ↓
Create worker processes in advance.

Apache worker MPM
  ↓
Use cheaper threads instead of one process per connection.

nginx / event-driven servers
  ↓
Why dedicate even a thread to an idle connection?

epoll
  ↓
Let the kernel tell us which of thousands of sockets
actually need attention.
```

A second recurring principle is:

> Move expensive setup off the hot request path and pay for it once.

---

# 2. Apache 1.3: prefork

A crucial correction:

Apache prefork does **not** normally fork a brand-new process for every request.

Instead, Apache creates a pool of child worker processes ahead of time.

```text
Apache parent
│
├── child process 1
├── child process 2
├── child process 3
└── ...
```

Each child can handle approximately one connection/request at a time.

So:

```text
Client A → process 1
Client B → process 2
Client C → process 3
```

## Why “prefork”?

Because Apache does the `fork()` before the request arrives.

Instead of:

```text
request
→ fork
→ handle request
```

it does:

```text
server startup
→ fork workers

later...

request
→ give it to existing worker
```

This removes `fork()` from the request latency path.

## Why are processes expensive?

A process has significant operating-system state:

- virtual address space
- stack
- heap
- page tables / memory mappings
- file descriptors
- scheduler state
- CPU register state

Processes can share physical pages via mechanisms such as copy-on-write, so “every process duplicates all memory” is too simplistic, but processes are still substantially heavier than threads.

The bigger scaling problem is that one worker can remain occupied by one connection.

For example:

```text
Client connected
      ↓
Apache process
      ↓
client currently sends nothing
      ↓
process still tied to that connection
```

With thousands of mostly-idle clients, that wastes worker capacity.

---

# 3. Apache 2.0: Worker MPM

MPM means:

> Multi-Processing Module

An MPM determines Apache's concurrency model: how Apache creates and uses processes and threads.

Examples:

```text
prefork MPM
worker MPM
event MPM
```

Worker MPM uses multiple child processes, but each process contains many threads.

```text
Apache parent
│
├── child process
│   ├── thread 1
│   ├── thread 2
│   ├── thread 3
│   └── ...
│
└── another child process
    ├── thread 1
    ├── thread 2
    └── ...
```

A worker thread performs request-processing work such as:

```text
read request bytes
↓
parse HTTP
↓
run request handler
↓
read file / talk to backend
↓
generate response
↓
write response
```

## Why are threads cheaper?

Threads in one process share:

```text
code
heap
global variables
address space
open resources
```

But every thread still needs its own:

```text
stack
register state
instruction pointer
scheduler bookkeeping
thread-local storage
```

So:

> Threads are cheaper than processes, but threads are not free.

If you create tens of thousands of threads, you still pay for stacks, scheduling, context switching, and bookkeeping.

---

# 4. HTTP keep-alive

Without connection reuse:

```text
TCP connection
GET /index.html
close

new TCP connection
GET /style.css
close

new TCP connection
GET /image.png
close
```

HTTP keep-alive allows one TCP connection to remain open for multiple HTTP requests.

```text
TCP connection
    │
GET /index.html
← response
    │
    │ idle
    │
GET /style.css
← response
    │
    │ idle
```

This saves TCP setup overhead.

But it creates a server-side problem:

> What resource remains assigned while the connection is open but idle?

If an entire thread/process remains occupied during the idle period, thousands of keep-alive connections become expensive.

---

# 5. Apache 2.4: Event MPM

Event MPM improves the idle-connection problem.

Instead of:

```text
idle connection → dedicated worker thread waiting
```

the architecture moves toward:

```text
idle connections
       ↓
event mechanism
       ↓
kernel watches them

when data arrives:
       ↓
worker thread handles actual work
```

On Linux, the event mechanism is typically based on `epoll`.

The important idea is:

> An open connection does not automatically deserve a dedicated execution thread.

This is a major conceptual transition from thread-per-connection toward event-driven connection management.

---

# 6. What is `epoll`?

On Unix-like systems, sockets are represented using file descriptors.

Suppose a process has:

```text
fd 10 → client A
fd 11 → client B
fd 12 → client C
...
```

With `epoll`, the process can essentially tell Linux:

> Watch all these sockets and tell me when one becomes ready.

Then the process sleeps:

```text
epoll_wait(...)
```

If socket 872 receives data:

```text
kernel notices
↓
epoll_wait() returns
↓
"socket 872 is readable"
```

You do not need one sleeping thread per socket.

This is what “park idle keep-alives on epoll” means:

```text
idle connection
→ register/watch via epoll
→ no request worker required
→ kernel wakes server when useful activity occurs
```

---

# 7. CGI: the expensive per-request model

Classic CGI often looked roughly like:

```text
HTTP request
↓
fork()
↓
exec()
↓
start interpreter/program
↓
load runtime
↓
load/parse script
↓
run script
↓
produce response
↓
exit process
```

The slide summarized this as:

```text
per request:
fork() + exec() + load interpreter
+ parse script + run + exit
```

## `fork()`

Creates a new process.

## `exec()`

Replaces that process's program image with another program.

For example:

```text
fork()
↓
child process
↓
exec("perl script.pl")
```

or conceptually:

```text
exec("python application.py")
```

The problem is that expensive initialization is repeated for every request.

If useful request work takes little time but setup takes significant time, CGI wastes a large fraction of execution doing initialization.

---

# 8. Java servlets

Servlets invert the CGI model.

Instead of:

```text
request
→ create environment
→ create handler
→ execute
→ destroy
```

the servlet container starts the application once.

Conceptually:

```text
server startup
↓
start JVM
↓
load class
↓
instantiate servlet object
↓
keep it alive
```

Then each request becomes approximately:

```java
servlet.service(req, res);
```

Hence the slide:

```text
at startup:
load class, instantiate once

per request:
service(req, res)
```

The key idea is:

> The object outlives the request.

One servlet instance can serve many requests over its lifetime.

---

# 9. “A servlet is instantiated once and serves every request on a thread from a pool”

There are two kinds of reuse.

First, reuse the application object:

```text
             Servlet object
                   │
      ┌────────────┼────────────┐
      ↓            ↓            ↓
 request 1     request 2     request 3
```

Second, reuse worker threads.

Instead of creating a new thread per request:

```text
Thread pool:

T1
T2
T3
T4
...
```

When a request arrives:

```text
request A → T3
```

After T3 finishes:

```text
T3 returns to pool
```

Later:

```text
request Z → same T3
```

So servlet containers avoid repeated:

```text
process creation
runtime startup
class loading
object construction
thread creation
```

---

# 10. Thread-safety consequence of servlets

The same servlet object may be called concurrently by multiple threads.

```text
Thread A ─┐
          ├→ same servlet instance
Thread B ─┘
```

Therefore mutable instance fields can create races.

For example:

```java
class BadServlet {
    String currentUser;

    void service(Request req, Response res) {
        currentUser = req.user();
    }
}
```

Two simultaneous requests may overwrite shared state.

Efficiency through reuse introduces a concurrency requirement: shared mutable state must be synchronized, avoided, or otherwise made thread-safe.

---

# 11. FastCGI

Classic CGI:

```text
request
→ create process/runtime
→ execute
→ exit
```

FastCGI:

```text
start persistent application workers once
↓
keep runtime loaded
↓
serve many requests through those workers
```

So:

```text
CGI:
setup + work
setup + work
setup + work

FastCGI:
setup once
work
work
work
work
```

This is why the slide says:

> Same move as prefork. Same move as FastCGI.

Not the same implementation—the same optimization principle:

> Perform expensive setup once and reuse the result.

---

# 12. Database connection pools

The exact same principle applies to databases.

Without pooling:

```text
request
↓
open DB connection
↓
TCP/TLS/authentication handshake
↓
query
↓
close connection
```

With pooling:

```text
application startup
↓
establish reusable database connections
↓
request borrows one
↓
query
↓
return connection to pool
```

Again:

> remove expensive setup from the request path.

---

# 13. Igor Sysoev and nginx

Igor Sysoev began writing nginx in 2002.

The key historical problem was not primarily:

> “Apache executes one request too slowly.”

It was:

> “How do I handle enormous numbers of simultaneous connections efficiently?”

Rambler was encountering connection-volume problems.

The key slide distinction:

```text
request volume ≠ connection volume
```

A server can have:

```text
100,000 open connections
```

while only:

```text
100 connections
```

are doing useful work at this instant.

That workload can overwhelm architectures that allocate one worker per connection even though CPU demand is relatively modest.

---

# 14. The C10K problem

C10K means:

> approximately 10,000 concurrent connections.

The historic systems question was:

> How can one server efficiently maintain 10,000 simultaneous network connections?

The difficult case is often:

```text
many connections
mostly idle
few currently active
```

For example:

```text
10,000 open connections
9,990 currently idle
10 currently active
```

A process/thread-per-connection architecture may still allocate resources to all 10,000.

An event-driven architecture tries to make the cost depend mostly on the 10 active connections.

---

# 15. nginx's central answer

Instead of:

```text
connection → process/thread
```

nginx uses:

```text
many connections
      ↓
event loop
      ↓
small number of worker processes
```

A worker can manage thousands of sockets simultaneously.

The worker is not executing 10,000 requests at the same time.

Rather:

```text
5,000 sockets registered
↓
most idle
↓
kernel says 8 are ready
↓
worker processes those 8
↓
returns to waiting
```

This is multiplexing.

---

# 16. Why nginx is a reverse proxy

Sysoev's design did not primarily attempt to turn nginx into a giant application runtime.

nginx handles the network-facing work well:

```text
client connections
TLS
keep-alive
HTTP parsing
slow clients
buffering
static files
connection management
load balancing
```

Dynamic application logic can live elsewhere:

```text
browser
↓
nginx
↓
application server
↓
database
```

For example:

```text
nginx → Java/Tomcat
nginx → FastCGI/PHP
nginx → Python app
```

That makes nginx a reverse proxy.

A forward proxy acts for clients:

```text
clients → proxy → external servers
```

A reverse proxy acts for servers:

```text
internet
↓
nginx
↓
backend servers
```

---

# 17. Why buffering slow clients matters

Suppose the application generates a response instantly, but the client downloads extremely slowly.

Without a reverse proxy:

```text
application worker
↓
waiting on slow client's network
↓
worker remains occupied
```

With nginx:

```text
application
↓
generate response
↓
nginx buffers it
↓
application worker becomes free
↓
nginx slowly transmits to client
```

This protects expensive backend workers from slow network clients.

It is another form of the same principle:

> Do not dedicate an expensive computational resource merely because a network connection exists.

---

# 18. nginx process architecture

The slide showed:

```text
                master
                  │
       ┌──────────┼──────────┐
       ↓          ↓          ↓
    worker 0   worker 1   worker N
    event loop event loop event loop

    cache manager
    cache loader
```

## Master process

The master:

```text
binds listening ports
reads configuration
forks worker processes
handles signals
manages workers
```

It does not normally process ordinary client requests.

Hence:

> “a master that never touches a connection”

More precisely, it establishes/manages listening infrastructure but does not perform normal request handling.

---

# 19. Worker processes

Workers do actual connection/request processing.

Each worker runs an event loop.

Conceptually:

```c
while (true) {
    events = wait_for_events();

    for each ready event:
        advance that connection;
}
```

One worker can own thousands of sockets.

---

# 20. One worker per CPU core

A common nginx configuration is approximately one worker per available CPU core.

For 8 cores:

```text
master

worker 0
worker 1
...
worker 7
```

Why?

One event-loop worker is primarily single-threaded for its normal event-processing work.

One worker provides concurrency among many connections, but multiple workers provide CPU parallelism.

So:

```text
Concurrency:
one worker juggles thousands of sockets

Parallelism:
multiple workers run simultaneously on multiple cores
```

These are distinct concepts.

---

# 21. Fixed process count under load

One important nginx property:

```text
10 clients       → approximately same worker count
1,000 clients    → approximately same worker count
10,000 clients   → approximately same worker count
```

Connections increase.

Worker process count does not increase proportionally.

Contrast with:

```text
fork-per-connection:

1 client     → +1 process
100 clients  → +100 processes
1000 clients → +1000 processes
```

Hence the slide:

> Same `fork()` syscall — the difference is how often you call it.

Using `fork()` is not inherently the issue.

Putting `fork()` on the per-request/per-connection hot path is.

---

# 22. nginx cache manager and cache loader

The cache manager performs cache maintenance, including eviction.

The slide mentions LRU:

> Least Recently Used

When limits are exceeded, older/less-recently-used cached entries can be removed.

The cache loader helps load metadata about cache files already present on disk when nginx starts.

These are separate helper processes rather than ordinary request workers.

---

# 23. The nginx event loop: the six important lines

The slide reduced nginx's worker loop to approximately:

```c
timer = ngx_event_find_timer();

ngx_process_events(cycle, timer, flags);

ngx_event_process_posted(cycle, &ngx_posted_accept_events);

ngx_shmtx_unlock(&ngx_accept_mutex);

ngx_event_expire_timers();

ngx_event_process_posted(cycle, &ngx_posted_events);
```

The conceptual algorithm is:

```text
1. Find when the next timer expires.
2. Sleep waiting for I/O, but no longer than that timeout.
3. Process pending accept-related events.
4. Release accept coordination.
5. Expire timers that are now due.
6. Process other posted events.
7. Repeat.
```

---

# 24. Why the worker spends most of its life asleep

The main waiting call on Linux eventually uses something like:

```text
epoll_wait()
```

This is intentional.

A good event-driven server should consume essentially no CPU while nothing requires attention.

The worker asks:

> Wake me if a registered socket becomes ready, or when my next timer expires.

Then it sleeps.

That is efficient behavior, not inactivity.

---

# 25. Timers are an argument to the sleep

Suppose nginx has timers:

```text
connection A timeout → 10 s
connection B timeout → 3 s
connection C timeout → 20 s
```

The nearest timer is:

```text
3 seconds
```

Then nginx effectively performs:

```text
epoll_wait(..., timeout = 3 seconds)
```

Two possibilities:

```text
socket activity occurs after 500 ms
→ kernel wakes nginx immediately
```

or:

```text
nothing occurs
→ epoll_wait returns after ~3 seconds
→ timeout processing occurs
```

This is what the slide means by:

> nginx never polls for timeouts.

It does not repeatedly ask:

```text
expired yet?
expired yet?
expired yet?
```

The timeout is integrated into the sleeping operation.

---

# 26. Event handlers and callbacks

An important nuance from our discussion:

When `epoll_wait()` reports readiness, the event's handler/callback is already associated with the event structure.

Conceptually:

```text
epoll_wait()
↓
socket 42 ready
↓
nginx obtains/posts corresponding event
↓
event handler runs
↓
handler performs as much nonblocking work as possible
↓
handler returns
↓
event loop continues
```

The handler does **not** normally own the connection until the entire request finishes.

For example:

```text
socket readable
↓
read available 500 bytes
↓
request incomplete
↓
return to event loop
```

Later:

```text
same socket readable again
↓
read remaining bytes
↓
continue request processing
```

This is the critical event-driven pattern:

> A callback advances the connection state; it does not monopolize the connection until completion.

---

# 27. Blocking vs event-driven processing

Traditional blocking model:

```c
read(client);        // wait
call_backend();      // wait
write(client);       // wait
```

One execution worker can spend most of its time blocked.

Event-driven model:

```text
client readable
→ read available data
→ return

backend writable
→ send available data
→ return

backend readable
→ receive available data
→ return

client writable
→ transmit available data
→ return
```

Waiting becomes the kernel's responsibility.

---

# 28. Posted events

The slide emphasized:

> Handlers do not necessarily execute inside the poll; they are posted and drained afterward.

Conceptually:

```text
epoll_wait
↓
A ready
B ready
C ready
↓
queue/post events
↓
return from low-level polling phase
↓
run A handler
run B handler
run C handler
```

This gives nginx controlled ordering and keeps low-level polling separate from higher-level protocol logic.

---

# 29. Load balancing among nginx workers

The slide showed:

```c
ngx_accept_disabled =
    ngx_cycle->connection_n / 8
    - ngx_cycle->free_connection_n;
```

Definitions:

```text
connection_n
= total connection slots available to this worker

free_connection_n
= currently unused connection slots
```

Suppose:

```text
total = 8000
```

Then:

```text
8000 / 8 = 1000
```

If only 500 slots are free:

```text
1000 - 500 = +500
```

The worker is more than 7/8 full.

That is:

```text
> 87.5% utilized
```

It should back off from aggressively accepting new connections.

---

# 30. Why this balances workers

Imagine:

```text
worker A → nearly full
worker B → mostly empty
worker C → mostly empty
```

Worker A locally sees:

```text
I have very few connection slots left
```

and stops competing as aggressively for new connections.

B and C continue accepting.

Thus:

```text
new clients
   ↓
mostly go elsewhere
```

No central scheduler has to maintain a table such as:

```text
worker 0 = 7231
worker 1 = 1843
worker 2 = 2051
```

Each worker can make a local decision.

The underlying pattern is:

> decentralized backpressure.

---

# 31. The thundering-herd problem

Suppose 8 nginx workers all wait for new connections.

One new connection arrives.

A poor kernel/waiting design might wake all 8:

```text
1 connection
↓
worker 0 wakes
worker 1 wakes
worker 2 wakes
...
worker 7 wakes
```

Only one can actually accept the connection.

The rest wake uselessly and contend.

This is the thundering-herd problem.

---

# 32. `accept_mutex`

Historically, nginx used an accept mutex to coordinate workers.

Conceptually:

```text
worker A holds accept mutex
↓
A may accept connections

later:
A releases it

worker B obtains it
↓
B may accept
```

This prevented every worker from fighting over the same incoming connections.

It was essentially a userspace workaround for limitations in kernel wake-up behavior.

---

# 33. Why `accept_mutex` became less important

Modern kernels gained better primitives.

The slide mentioned:

```text
EPOLLEXCLUSIVE
SO_REUSEPORT
```

`EPOLLEXCLUSIVE` helps avoid waking every epoll waiter for the same event.

`SO_REUSEPORT` can allow multiple listening sockets on the same port, letting the kernel distribute incoming connections among them.

Therefore more fan-out/load distribution can occur inside the kernel rather than requiring nginx's userspace lock.

General lesson:

```text
kernel limitation
↓
userspace workaround
↓
kernel improves
↓
workaround becomes less necessary
```

---

# 34. `select()` vs `epoll()`

This is one of the most exam-important comparisons.

The slide's simplified statement:

> `select` is O(watched).  
> `epoll` is O(active).

Meaning:

With 100,000 registered connections but only 20 ready:

```text
select:
cost depends substantially on 100,000

epoll:
event-return cost depends much more closely on 20
```

This is exactly what a C10K-style workload needs.

---

# 35. How `select()` works conceptually

The application has sets of file descriptors:

```text
fd 10
fd 11
fd 12
...
```

Each iteration it essentially constructs/passes the watched set:

```text
prepare fd sets
↓
select(...)
↓
kernel examines them
↓
return modified sets
↓
application scans them
↓
determine which fds are ready
```

Then the next iteration repeats.

The watch set is repeatedly supplied by userspace.

---

# 36. Why `select()` scales poorly

If:

```text
10,000 sockets watched
10 sockets ready
```

the server still repeatedly works with the set of 10,000.

Conceptually there is work in:

```text
building/preparing watched set
copying/communicating it to kernel
kernel examining watched descriptors
application scanning returned set
```

So work is tied to:

```text
number watched
```

rather than mainly:

```text
number ready
```

---

# 37. How `epoll()` differs

With epoll:

```text
epoll_create(...)
```

creates persistent kernel-side event state.

Sockets are registered:

```text
epoll_ctl(ADD fd 10)
epoll_ctl(ADD fd 11)
...
```

They stay registered until changed/removed.

Then:

```text
epoll_wait(...)
```

returns ready events.

So:

```text
register once
↓
kernel retains interest set
↓
wait
↓
receive ready descriptors
```

You do not rebuild the entire watched set every iteration.

---

# 38. “Who owns the watch list?”

This was the deepest statement on the slide.

With `select()`:

```text
application repeatedly supplies/rebuilds watch state
```

With `epoll()`:

```text
kernel retains persistent watch state
```

Because the kernel retains it, it can maintain a ready-event structure and return ready items efficiently.

That single architectural choice produces much of the performance difference.

---

# 39. Example: 100,000 connections, 50 active

`select()` conceptually:

```text
prepare information for ~100,000
↓
kernel considers ~100,000
↓
application examines result set
↓
discover 50 useful fds
```

`epoll()`:

```text
100,000 registered earlier
↓
50 become ready
↓
epoll_wait returns those ready events
```

That is why epoll fits mostly-idle connection workloads.

---

# 40. `FD_SETSIZE` and `select()`

Traditional `select()` APIs use a fixed-size `fd_set`.

A common value is:

```text
FD_SETSIZE = 1024
```

Thus a file descriptor number beyond that cannot simply be placed in the normal fd set.

This is a separate issue from the operating system's maximum number of open files.

Therefore:

```bash
ulimit -n 100000
```

may let your process open far more descriptors, but that does not automatically make traditional `select()` able to represent fd numbers above `FD_SETSIZE`.

---

# 41. `ulimit -n`

`ulimit -n` controls the process's open-file/file-descriptor resource limit.

Since sockets consume file descriptors:

```text
more simultaneous sockets
→ need higher file-descriptor limit
```

With epoll, this resource limit becomes one of the relevant scaling ceilings.

With select, `FD_SETSIZE` can remain an additional independent constraint.

---

# 42. Portability

`select()` is extremely old and widely available.

`epoll` is Linux-specific.

Analogous systems include:

```text
Linux        → epoll
BSD/macOS    → kqueue
Windows      → IOCP-style mechanisms
```

nginx therefore has platform-specific event modules.

---

# 43. Important nuance about “epoll is O(active)”

For the exam based on these slides, remember:

```text
select → O(watched)
epoll  → roughly O(ready events) for event retrieval
```

But do not overgeneralize this into:

> Every operation in epoll is literally O(number of active sockets).

Registration/removal via `epoll_ctl()` has its own costs, and kernel internals are more nuanced.

The slide is summarizing the dominant architectural difference for repeated waits.

---

# 44. The stale-event bug

This is the most subtle slide so far.

`epoll_wait()` returns a batch of ready events.

Example:

```text
event 0
event 1
event 2
...
event 7
```

nginx processes them one after another.

But processing an earlier event can change the world before a later event is handled.

---

# 45. File descriptors are reusable numbers

Suppose:

```text
fd 55 → Client A
```

An event for Client A is already present in the batch.

Then nginx closes it:

```c
close(55);
```

Now `55` becomes available.

A new connection arrives.

The OS may assign:

```text
fd 55 → Client B
```

The integer is the same.

The connection is completely different.

Therefore:

> File descriptor number is not a permanent connection identity.

---

# 46. How the stale event appears

Imagine:

```text
epoll_wait() returned:

event 1 → something
...
event 7 → old Client A, fd 55
```

While processing event 1:

```text
Client A gets closed
↓
fd 55 released
```

Then:

```text
Client B connects
↓
kernel reuses fd 55
```

Later nginx reaches event 7.

If it only checks:

```text
fd == 55
```

it might accidentally apply Client A's old event to Client B.

That is the stale-event bug.

---

# 47. Reused connection structures make it even trickier

nginx also reuses internal `ngx_connection_t` objects.

So potentially both are reused:

```text
same fd number
same connection-object memory address
```

Yet the logical connection is different.

Old:

```text
pointer = 0x1000
fd = 55
Client A
```

New:

```text
pointer = 0x1000
fd = 55
Client B
```

Pointer and fd alone are therefore insufficient as an identity.

---

# 48. Solution: generation / instance identity

Attach a generation marker to distinguish lifetimes.

Conceptually:

```text
Client A:
pointer = P
fd = 55
generation = 0

Client B:
pointer = P
fd = 55
generation = 1
```

Then an old event saying:

```text
P, generation 0
```

can be compared against current state:

```text
P, generation 1
```

Mismatch means:

```text
stale event
→ ignore
```
---

# 55. General principle behind the stale-event bug

This lesson extends far beyond nginx.

If a resource can be destroyed and its identifier reused, then:

```text
handle/address alone
```

may not be enough to establish identity.

You often need:

```text
handle + generation/version
```

This is related to a broader class of reuse/ABA problems.

An event notification should also be viewed as:

> information about something that was true when the notification was produced

not:

> a guarantee that the object is still in the same lifetime/state when you eventually process it.

---


# 57. Three different kinds of “reuse” that the slides repeatedly show

Do not mix these up.

**Execution-resource reuse**

```text
prefork:
reuse worker processes

thread pools:
reuse threads
```

**Application-state reuse**

```text
servlets:
reuse loaded classes
reuse servlet object

FastCGI:
reuse loaded runtime/application
```

**Connection-waiting efficiency**

```text
event MPM / nginx:
do not dedicate a worker to an idle socket
```

They solve different costs.

---

# 58. Request vs connection vs worker

These three terms must be distinct in your head.

A **connection** is the transport-level communication channel, usually TCP.

```text
client ================= server
          TCP
```

A **request** is an HTTP message sent over a connection.

One keep-alive connection may carry multiple requests:

```text
connection
├── request 1
├── request 2
├── request 3
└── ...
```

A **worker** is a process or thread that performs processing.

Older designs tied a worker more closely to a connection.

Event-driven systems decouple those concepts.

---

# 59. Concurrency vs parallelism

Likely exam distinction.

Concurrency:

> making progress on multiple tasks over overlapping periods.

One nginx worker can concurrently manage thousands of sockets through an event loop.

Parallelism:

> literally executing multiple computations simultaneously.

Multiple nginx workers can run on multiple CPU cores in parallel.

Therefore:

```text
one worker:
high concurrency

multiple workers on multiple cores:
parallelism + concurrency
```

---

# 61. Why event-driven architecture is especially good for I/O-bound workloads

If connections spend most of their lives waiting for:

```text
client input
network transmission
upstream server
disk/network readiness
```

then one thread per connection wastes execution resources.

An event loop lets one worker manage many waiting tasks.

However, if an event handler performs long CPU-bound computation:

```text
event callback
↓
CPU work for 2 seconds
```

that worker cannot service its other sockets during those 2 seconds.

Therefore event-loop handlers should generally be short/nonblocking.

This is an important consequence of the architecture.

---

# 62. Likely long-answer exam question: “Explain why nginx scales better for many idle connections”

A strong answer would contain this chain:

```text
Traditional process/thread-per-connection servers associate
a relatively expensive execution resource with each connection.

With many mostly-idle keep-alive connections, this wastes memory,
thread stacks, scheduler state, and context switching.

nginx instead runs a small number of worker processes, typically
around one per CPU core. Each worker uses an event loop.

Sockets are registered with an OS readiness mechanism such as epoll.
The worker normally sleeps in epoll_wait() and is awakened only when
a socket is ready or a timer expires.

Therefore the number of open connections can grow substantially
without requiring a corresponding increase in threads or processes.
```

---

# 63. Likely exam question: “Compare CGI with servlets”

Core points:

```text
CGI:
per request:
fork
exec
start runtime
load/parse program
run
exit

Servlet:
startup:
start JVM
load class
create servlet
create/reuse thread pool

per request:
select worker thread
call service(req,res)
```

Conclusion:

> Servlet architecture amortizes expensive setup across many requests.

Also mention thread-safety because one long-lived object may be called concurrently.

---

# 64. Likely exam question: “Why is epoll better than select for C10K?”

Strong answer:

```text
select requires the application to repeatedly provide a set of watched
file descriptors. The kernel examines that set and the application must
scan the returned set, so cost is tied to the total number watched.

epoll keeps a persistent interest set inside the kernel. Sockets are
added or removed with epoll_ctl(), while epoll_wait() returns ready
events.

For 100,000 mostly-idle sockets with only a few active sockets,
this avoids repeatedly scanning the entire population and makes
event retrieval scale much more closely with the number of ready
events.

epoll also avoids select's traditional FD_SETSIZE limitation,
although normal file-descriptor resource limits still apply.
```

---

# 65. Likely exam question: “Explain nginx's master-worker design”

Mention:

```text
master:
bind/listen
read config
fork workers
handle signals
manage lifecycle

workers:
handle client traffic
run event loops
own many simultaneous connections

roughly one worker per CPU core:
parallelism between workers

worker count does not increase linearly with connection count:
each worker multiplexes many sockets
```

Also distinguish optional helper processes such as cache manager/loader.

---

# 66. Likely exam question: “Explain nginx's event loop”

A compact model answer:

```text
The worker first determines the nearest timer expiration. It then
waits for I/O events using an OS event facility such as epoll, using
the nearest timer as the wait timeout.

When the wait returns, nginx processes accept-related posted events,
performs relevant synchronization, expires timers that are now due,
and drains other posted events.

Individual event handlers perform only the work that can make
progress immediately. If more I/O is required, they arrange to wait
for the next readiness event and return to the event loop.
```

---

# 67. Likely exam question: “Explain the stale epoll event problem”

Use this exact sequence:

```text
1. epoll_wait returns a batch containing an event for connection A.
2. Before that event is processed, another callback closes A.
3. A's fd and nginx connection object can be reused.
4. New client B may receive the same fd/object slot.
5. The old batch still contains A's event.
6. Without generation checking, it could be applied to B.
7. nginx stores an instance/generation bit in the low bit of the
   aligned connection pointer.
8. On event handling, nginx compares the event's instance with the
   current connection instance.
9. Mismatch means stale event, so nginx discards it.
```

That is almost certainly enough for a full-credit conceptual answer.

