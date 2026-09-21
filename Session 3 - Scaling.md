## 1. The core scalability problem

A server may have thousands or millions of clients connected at once.

The basic question is:

> How should the server manage many simultaneous connections without wasting CPU and memory?

The naive solutions are:

```text
one process per client
one thread per client
```

Both are simple, but both become expensive at large scale.

The scalable idea is:

```text
small number of processes/threads
        +
event loops watching many sockets
```

This is called I/O multiplexing.

---

# 2. Process per client: `fork()`

A classic Unix server can do:

```c
while (1) {
    client_fd = accept(server_fd, ...);

    if (fork() == 0) {
        // child handles this client
    }

    // parent continues accepting clients
}
```

`fork()` creates a child process.

Its return value tells you which process you are in:

```c
pid_t pid = fork();

if (pid == 0) {
    // child
}
else if (pid > 0) {
    // parent
}
else {
    // fork failed
}
```

After `fork()`, both parent and child continue from the same point, but they receive different return values.

Typical structure:

```text
Parent
├── accepts client A → child A
├── accepts client B → child B
└── accepts client C → child C
```

Advantages:

- simple programming model
- good process isolation
- one client's failure does not necessarily corrupt another

Disadvantages:

- process creation is expensive
- page tables, file-descriptor tables, scheduler state, memory mappings, etc.
- large numbers of idle clients still consume resources

So:

```text
1 connection = 1 process
```

does not scale well.

---

# 3. Thread per client

A cheaper version is:

```text
1 client = 1 thread
```

Threads share the same process address space, so they are cheaper than processes.

But every thread still costs:

- stack memory
- scheduler bookkeeping
- context-switching overhead
- synchronization complexity

Therefore:

```text
10,000 clients
≈ 10,000 threads
```

can still become expensive, especially if most of those clients are idle.

The exact number 10,000 is not a strict threshold. The idea is simply that per-client thread cost grows linearly with connection count.

---

# 4. I/O multiplexing

The scalable alternative is:

```text
many sockets
    ↓
one/few event loops
```

Instead of dedicating a thread to each connection, one thread asks the kernel:

> Tell me which sockets currently have useful work available.

Then it only services those sockets.

Example:

```text
socket A  idle
socket B  readable
socket C  idle
socket D  writable
```

The event loop handles B and D instead of wasting resources on A and C.

This is the fundamental idea behind:

- `select`
- `poll`
- `epoll`
- `kqueue`
- IOCP
- modern async runtimes

---

# 5. `select()`

`select()` allows one thread to watch multiple file descriptors.

Typical use:

```c
fd_set readfds;

FD_ZERO(&readfds);
FD_SET(server_fd, &readfds);

for (...) {
    FD_SET(client_fd, &readfds);
}

select(maxfd + 1, &readfds, NULL, NULL, NULL);
```

The arguments conceptually are:

```c
select(
    maxfd + 1,
    readfds,
    writefds,
    exceptfds,
    timeout
);
```

`readfds`:
watch for readable descriptors.

A listening socket becomes readable when a new connection is waiting.

A connected TCP socket becomes readable when:

- data arrived
- EOF/connection close can be observed

`writefds`:
watch for sockets that can currently accept more outgoing data without blocking.

`exceptfds`:
exceptional conditions. It is not generally the normal mechanism for detecting TCP disconnects.

The last argument is the timeout.

```c
select(..., NULL);
```

means:

```text
wait indefinitely until some descriptor becomes ready
```

---

# 6. Why `select()` is better than one-thread-per-client

Suppose:

```text
10,000 clients
9,999 idle
1 sends data
```

With one thread per client, you may have 10,000 thread stacks and scheduler entries.

With `select()`, one thread can wait on all of them.

So multiplexing dramatically reduces thread/process overhead.

But `select()` itself has scaling problems.

---

# 7. Problem 1 with `select()`: `FD_SETSIZE`

On many glibc/Linux systems:

```text
FD_SETSIZE = 1024
```

`fd_set` is a fixed-width bitmap.

It can normally represent descriptor numbers:

```text
0 ... 1023
```

but not descriptor `1024`.

This is a representation limit of `select()`.

Important:

```text
FD_SETSIZE = 1024
```

is not the same as:

```text
ulimit -n = 1024
```

They may have the same default value but represent different limits.

---

# 8. Problem 2 with `select()`: O(n) scanning

With `select()`, on every iteration:

1. user code rebuilds the descriptor set
2. kernel checks the monitored descriptor range
3. program checks which descriptors were marked ready

Therefore the cost grows with the number of watched descriptors.

Conceptually:

```text
select → O(n)
```

If:

```text
10,000 sockets
only 1 ready
```

the system still has to deal with the whole monitored set.

Idle connections therefore still impose scanning overhead.

---

# 9. `epoll`

Linux `epoll` improves this model.

Instead of repeatedly saying:

> Here are all my sockets. Check them.

you first register sockets:

```c
epoll_ctl(...)
```

The kernel remembers them.

Then you call:

```c
epoll_wait(...)
```

and the kernel reports which registered descriptors are ready.

Mental model:

```text
select:
"Check all of these again."

epoll:
"You already know them.
Tell me which became ready."
```

Simplified scaling intuition:

```text
select → cost related to number monitored
epoll  → cost closer to number of ready events
```

The exact internal complexity is more nuanced, but this is the correct architectural model.

Equivalent ideas on other systems include:

```text
Linux       epoll
BSD/macOS   kqueue
Windows     IOCP
```

`io_uring` is a broader modern Linux asynchronous I/O mechanism.

---

# 10. Production servers combine models

Real servers usually do not choose purely:

```text
processes
OR
threads
OR
event loops
```

They combine them.

Typical architecture:

```text
master process
   ↓
several worker processes
   ↓
each worker has an event loop
   ↓
each loop handles many sockets
```

Often:

```text
one worker per CPU core
```

This gives:

```text
processes → CPU parallelism + crash isolation
event loop → scalable connection handling
```

Example:

```text
Worker 1 → 5,000 sockets
Worker 2 → 5,000 sockets
Worker 3 → 5,000 sockets
Worker 4 → 5,000 sockets
```

`SO_REUSEPORT` can allow multiple workers to listen on the same port, letting the kernel distribute new connections among them.

---

# 11. Timeouts

Network software should not wait forever.

A connection may:

- stop sending data midway
- disappear without a clean close
- stall while receiving
- stall while sending

Therefore operations should generally have deadlines.

For `select()`:

```c
select(..., timeout);
```

A `NULL` timeout means indefinite waiting.

Important nuance:

An idle client does not by itself freeze an event loop.

Suppose:

```text
A idle
B sends data
C idle
```

`select()` still wakes because B became ready.

The danger is instead:

- blocking operations after the event loop
- requests that never complete
- no timeout on slow/stalled peers

---

# 12. Retry and backoff

Clients also need deadlines.

Typical flow:

```text
request
↓
wait until deadline
↓
timeout
↓
possibly retry
↓
wait longer before retrying again
```

Example exponential backoff:

```text
1 second
2 seconds
4 seconds
8 seconds
```

This prevents retry storms and reduces pressure on an already overloaded service.

---

# 13. CGI: Common Gateway Interface

CGI is an old mechanism for letting a web server run application programs.

Typical architecture:

```text
client
  ↓ HTTP
Apache
  ↓ starts CGI process
CGI application
```

Traditional CGI generally runs on the same machine as Apache.

For each request:

```text
request
↓
fork/exec new CGI process
↓
run application
↓
produce response
↓
process exits
```

This is effectively process-per-request.

---

# 14. CGI is an interface, not a language/runtime

CGI itself is not:

- a compiler
- an interpreter
- a language

It is simply a contract describing how a web server communicates with an external application process.

If the CGI program is Python:

```text
Apache
↓
start Python interpreter
↓
run script
```

If it is a compiled C executable:

```text
Apache
↓
execute binary
```

Apache still handles:

- TCP connections
- HTTP parsing
- request routing
- invoking CGI
- building environment variables
- passing request body
- collecting output
- returning HTTP response

The CGI program handles application logic.

---

# 15. CGI communication channels

Apache connects three standard streams of the CGI child:

```text
stdin
stdout
stderr
```

Conceptually:

```text
Apache → CGI stdin
Apache ← CGI stdout
Apache ← CGI stderr
```

Sequence:

```text
fork
↓
wire stdin/stdout/stderr to pipes
↓
exec CGI program
```

The request body goes to CGI `stdin`.

The CGI response comes from `stdout`.

Errors/debugging go to `stderr`, generally into Apache logs.

---

# 16. CGI environment variables

HTTP request metadata is converted into environment variables.

Example HTTP request:

```http
GET /search?q=books HTTP/1.1
User-Agent: curl
```

could produce:

```text
REQUEST_METHOD=GET
QUERY_STRING=q=books
HTTP_USER_AGENT=curl
SERVER_PROTOCOL=HTTP/1.1
```

General header conversion:

```text
User-Agent
↓
uppercase
↓
USER-AGENT
↓
replace - with _
↓
USER_AGENT
↓
prefix HTTP_
↓
HTTP_USER_AGENT
```

Examples:

```text
Accept-Language → HTTP_ACCEPT_LANGUAGE
X-Test          → HTTP_X_TEST
```

Some CGI variables are special and do not use `HTTP_`:

```text
REQUEST_METHOD
QUERY_STRING
CONTENT_TYPE
CONTENT_LENGTH
SERVER_PROTOCOL
```

---

# 17. GET and POST in CGI

For GET:

```text
QUERY_STRING
```

typically contains URL query parameters.

Example:

```text
/search?q=books
```

gives:

```text
QUERY_STRING=q=books
```

For POST, the request body is read from:

```text
stdin
```

---

# 18. CGI response format

The CGI program writes something like:

```text
Content-Type: text/html

<html>...</html>
```

The blank line separates:

```text
headers

body
```

Apache reads the headers, then streams the response body to the client.

Therefore a CGI program can be very simple:

```text
read environment
read stdin if needed
run application logic
print headers
print blank line
print body
```

Any language capable of this can implement CGI.

---

# 19. The main problem with CGI

CGI pays startup cost on every request.

For every request:

```text
fork
exec
load runtime
load code
initialize
run once
destroy process
```

This is wasteful under high request rates.

The core optimization principle is:

> Move expensive setup out of the request path and pay for it once at startup.

---

# 20. Embedded modules: `mod_php`, `mod_perl`

One historical fix was to put the language runtime directly into Apache.

Example:

```text
Apache process
├── HTTP server
└── PHP runtime
```

Benefits:

- no new process per request
- runtime already loaded

Problem:

- application runtime shares Apache's address space
- weaker isolation
- a bad crash or memory corruption may affect the web server

This motivates persistent but separate application processes.

---

# 21. FastCGI

FastCGI keeps application processes alive and communicates with them over sockets.

Architecture:

```text
client
   ↓ HTTP
nginx/Apache
   ↓ FastCGI
persistent application process
```

Example with PHP:

```text
client
↓
nginx
↓
php-fpm
```

`php-fpm` maintains a pool of long-lived PHP workers.

Instead of:

```text
request → start PHP → execute → kill
```

you get:

```text
start PHP worker once
↓
request 1
request 2
request 3
...
```

---

# 22. Why FastCGI uses sockets instead of CGI pipes

CGI:

```text
Apache
↓ creates child
pipes
↓
new CGI process
```

Apache creates the child, so it can directly wire up stdin/stdout/stderr.

FastCGI:

```text
nginx
↕ socket
already-running FastCGI process
```

The application process already exists independently.

Therefore a general IPC mechanism is required.

FastCGI commonly uses:

```text
Unix-domain socket
or
TCP socket
```

Unix socket:

```text
same machine only
```

TCP socket:

```text
same machine or remote machine
```

---

# 23. nginx as front-end, FastCGI as application layer

nginx handles:

- client TCP connections
- TLS
- HTTP parsing
- slow clients
- buffering
- timeouts
- thousands of simultaneous connections

FastCGI application workers handle:

- application code
- database queries
- business logic
- response generation

This separation protects expensive application workers from slow clients.

---

# 24. Slow client buffering

Suppose a client uploads very slowly:

```text
client → very slowly → nginx
```

nginx can absorb/buffer that request before engaging an application worker.

Similarly, if the application generates a large response quickly but the client downloads slowly:

```text
FastCGI worker
↓ fast
nginx buffer
↓ slow
client
```

The application worker becomes free quickly while nginx handles the slow network transfer.

That matters because nginx can efficiently maintain huge numbers of waiting sockets using an event loop.

---

# 25. FastCGI protocol: typed records

FastCGI does not simply send raw HTTP text.

A request is a sequence of typed records.

Typical flow:

```text
FCGI_BEGIN_REQUEST

FCGI_PARAMS
  REQUEST_METHOD=POST
  QUERY_STRING=x=10
  SCRIPT_FILENAME=/var/www/test.php
  CONTENT_LENGTH=17
  ...

FCGI_PARAMS
  <empty>

FCGI_STDIN
  request body

FCGI_STDIN
  <empty>
```

`FCGI_BEGIN_REQUEST`:
starts a request.

`FCGI_PARAMS`:
contains CGI-style metadata.

Empty `FCGI_PARAMS`:
means parameter stream finished.

`FCGI_STDIN`:
contains request body bytes.

Empty `FCGI_STDIN`:
means request body stream finished.

---

# 26. FastCGI preserves CGI concepts

FastCGI still uses familiar CGI-style names:

```text
REQUEST_METHOD
QUERY_STRING
CONTENT_LENGTH
HTTP_HOST
HTTP_USER_AGENT
```

The major difference is transport.

CGI:

```text
environment variables + pipes + new process
```

FastCGI:

```text
typed records + socket + persistent process
```

---

# 27. FastCGI response

Response records include:

```text
FCGI_STDOUT
FCGI_STDERR
FCGI_END_REQUEST
```

`FCGI_STDOUT` carries response headers and body.

Example:

```text
Content-Type: text/html

<html>...</html>
```

An empty `FCGI_STDOUT` means the stdout stream is finished.

`FCGI_STDERR` carries debugging/error output.

It goes to server logs rather than the client.

`FCGI_END_REQUEST` means the request has completed.

Crucially:

```text
request ends
but worker process stays alive
```

---

# 28. FastCGI framing

FastCGI uses both:

```text
length fields
and
delimiters
```

Each record contains a content length.

Example:

```text
contentLength = 500
```

means read exactly 500 bytes.

This allows arbitrary binary data inside the payload.

Then an empty record acts as an end marker.

Example:

```text
FCGI_STDIN length 1000
FCGI_STDIN length 1000
FCGI_STDIN length 500
FCGI_STDIN length 0
```

Interpretation:

```text
record lengths → where each record ends
zero-length record → where the stream ends
```

`CONTENT_LENGTH` is another length-based framing mechanism at a higher protocol level.

---

# 29. Servlets and the JVM model

Java Servlets solve the same startup-cost problem differently.

With Tomcat/Jetty:

```text
Tomcat JVM
├── HTTP server
├── servlet container
└── application code
```

The JVM stays running.

Requests are dispatched to already-existing threads.

Typical flow:

```text
request
↓
idle worker thread
↓
execute servlet
↓
thread returns to pool
```

This is pre-threading.

---

# 30. Tomcat vs Apache + embedded runtime

Conceptually they are similar:

`mod_php`:

```text
Apache process
├── HTTP server
└── PHP runtime
```

Tomcat:

```text
JVM process
├── HTTP server
├── servlet container
└── Java runtime/application
```

Tomcat is specifically designed around the integrated runtime model.

---

# 31. Servlet isolation

Multiple applications may run inside the same JVM.

Tomcat can isolate applications through:

- class loaders
- thread management
- application contexts

But this is weaker than process isolation.

If one application:

- consumes all heap
- spins CPU endlessly
- creates excessive threads
- causes JVM-wide failure

other applications may suffer.

Strong isolation therefore often requires:

```text
separate JVMs
containers
VMs
```

---

# 32. What is a web server?

A web server is a server that speaks HTTP.

Examples:

```text
nginx
Apache
router admin UI
many Node.js applications
CouchDB HTTP API
```

Not every server speaks HTTP.

Examples:

```text
Redis → RESP
MySQL → MySQL wire protocol
Kafka → Kafka protocol
MQTT broker → MQTT
```

A REST proxy can make a non-HTTP backend accessible through HTTP:

```text
HTTP client
↓
REST proxy
↓ native protocol
Kafka
```

Scaling techniques often depend on protocol properties.

---

# 33. HTTP statelessness

HTTP is stateless at the protocol request level.

Each request is designed to carry the information needed for that request.

Therefore:

```text
request 1 → server A
request 2 → server C
request 3 → server B
```

can work naturally.

This enables easy load balancing.

A web server can often be added to a pool:

```text
load balancer
├── server A
├── server B
├── server C
└── new server D
```

That is why horizontally scaling stateless web tiers is comparatively easy.

---

# 34. Persistent database connections

Database protocols commonly keep persistent connections because reconnecting for every query is expensive.

A fresh connection might require:

```text
TCP handshake
↓
protocol handshake
↓
authentication
↓
session setup
↓
query
```

Database connections may also contain connection-specific state:

```text
transactions
prepared statements
session variables
authentication context
temporary resources
```

Therefore the scaling question becomes not only:

```text
requests per second?
```

but also:

```text
how many persistent connections can this server maintain?
```

---

# 35. The four major scalability walls

The course identified four recurring limits:

```text
1. file descriptors
2. memory
3. syscall/context-switch overhead
4. queues
```

Almost every scaling technique attacks one or more of these.

---

# 36. File-descriptor limits

Every socket uses a file descriptor.

Roughly:

```text
1 TCP connection ≈ 1 fd
```

The process may have a soft limit:

```bash
ulimit -n
```

which commonly may be something like:

```text
1024
```

That means the process cannot keep arbitrarily many descriptors open.

Some descriptors are already used by:

```text
stdin
stdout
stderr
log files
listening sockets
database sockets
files
```

So the number of client connections is lower than the raw limit.

---

# 37. C10K

C10K means:

> Can one server maintain 10,000 concurrent connections?

Around 1999 this was considered a major scalability challenge.

Later targets became much larger:

```text
C10M = 10 million connections
```

Modern systems have demonstrated millions of concurrent connections under controlled conditions.

The problem was not solved by one trick.

It required combinations of:

- higher fd limits
- event-driven I/O
- fewer threads
- efficient kernels
- careful memory management
- socket tuning
- timeouts

---

# 38. Raising descriptor limits

High-scale servers typically raise the process/system fd limits.

But this only solves one bottleneck.

Even if:

```text
ulimit -n = 500000
```

you still need:

- enough RAM
- efficient socket handling
- enough kernel resources
- appropriate TCP settings
- event-driven I/O such as `epoll`

Also, raising `ulimit` does not fix `select()`'s separate `FD_SETSIZE` limitation.

---

# 39. Memory as a scaling limit

Connections consume memory even while idle.

Possible per-connection cost:

```text
socket buffers
application buffers
request objects
TLS state
connection metadata
```

Threads also require stacks.

Therefore:

```text
10,000 idle connections
```

still consume real memory.

This is one reason event-driven architectures outperform thread-per-client designs.

---

# 40. System calls and user/kernel boundary

Application code runs in user space.

Networking and files are managed by the kernel.

Calls such as:

```c
read()
write()
accept()
send()
```

are system calls.

Conceptually:

```text
user-space code
↓ syscall
kernel
↓
user-space resumes
```

A syscall has overhead:

- state transition
- validation
- kernel bookkeeping
- security checks
- return transition

At low volume this is tiny.

At millions of operations per second it becomes significant.

---

# 41. Syscalls vs context switches

These are related but different.

A syscall:

```text
user mode → kernel mode → user mode
```

A context switch:

```text
thread/process A → thread/process B
```

A syscall does not necessarily imply a context switch.

Too many threads increase scheduler/context-switch overhead.

Too many tiny I/O calls increase syscall overhead.

---

# 42. Kernel bypass

Some extremely high-performance systems bypass much of the normal kernel networking stack.

Examples:

```text
DPDK
netmap
user-space TCP stacks
```

Conceptually:

```text
normal:
NIC → kernel network stack → application

kernel bypass:
NIC → user-space networking system
```

This can greatly increase performance but sacrifices convenience and tooling.

It is specialized, not the default approach.

---

# 43. `sendfile()`

Normal static-file serving:

```c
read(file_fd, buf, n);
write(sock_fd, buf, n);
```

Data path:

```text
file
→ kernel
→ user-space buffer
→ kernel
→ socket
```

With:

```c
sendfile(sock_fd, file_fd, ...);
```

the kernel can move data toward the socket without first copying it into the application's user-space buffer.

Conceptually:

```text
file → kernel → socket
```

Benefits:

- fewer syscalls
- fewer copies
- less CPU overhead
- very useful for static-file servers like nginx

---

# 44. `io_uring`

`io_uring` reduces syscall overhead and supports asynchronous I/O through shared ring buffers.

Instead of:

```text
syscall
syscall
syscall
syscall
```

an application can submit many operations and receive completions through shared queues.

It is especially useful for high-throughput async I/O.

---

# 45. Queues

When requests arrive faster than the system can finish them:

```text
arrival rate > processing rate
```

the excess work waits.

Example:

```text
1000 requests/s arrive
800 requests/s complete
```

Then:

```text
200 requests/s accumulate
```

The queue grows.

Consequences:

- higher latency
- more memory usage
- timeouts
- eventually drops/failures

This is why a slow server consumes more than time—it also consumes space.

---

# 46. Little's Law: concurrency = throughput × latency

A crucial relationship:

```text
concurrency
≈ throughput × latency
```

or:

```text
connections in flight
=
requests per second
×
seconds per request
```

Example:

```text
10,000 requests/s
×
0.1 s/request
=
1,000 concurrent requests
```

If latency becomes 1 second:

```text
10,000 × 1
=
10,000 concurrent requests
```

Same traffic. Ten times as much concurrency.

---

# 47. Why latency causes memory pressure

Every in-flight request holds resources:

```text
socket
buffer
request object
possibly thread/task
possibly DB state
```

If latency increases, requests stay alive longer.

Therefore more requests coexist.

So:

```text
higher latency
→ higher concurrency
→ more memory/resource consumption
```

This is what the slide meant by:

> slowness is a memory bug

Not literally a memory bug, but slow work causes more simultaneous state to accumulate.

---

# 48. Asynchronous/background work

Long-running work is often moved out of the synchronous HTTP request path.

Instead of:

```text
HTTP request
↓
perform 30-second task
↓
response
```

use:

```text
HTTP request
↓
enqueue job
↓
return quickly

background worker
↓
does slow task
```

Benefits:

- shorter request latency
- fewer open requests
- smaller queues at the web tier
- better resource utilization

---

# 49. HAProxy

HAProxy is a load balancer/proxy.

Architecture:

```text
clients
↓
HAProxy
├── backend A
├── backend B
└── backend C
```

Historically:

- HAProxy appeared before Linux `epoll`
- early versions used `select()`/similar mechanisms
- later adopted `epoll`
- it was largely event-driven before becoming multi-threaded

Its design proved that a single event-driven process could achieve very high network throughput.

---

# 50. HAProxy as a load-balancing tier

HAProxy popularized the idea of a dedicated layer:

```text
clients
↓
load balancer
↓
application servers
```

This layer can:

- distribute traffic
- health-check backends
- retry/fail over
- route different traffic to different services

---

# 51. HAProxy and connection churn

Client connections often open and close frequently.

HAProxy can absorb this churn.

Client side:

```text
connect
request
disconnect
```

Backend side:

```text
HAProxy ↔ backend persistent connection
request 1
request 2
request 3
...
```

This reduces repeated TCP handshakes and setup cost on upstream servers.

---

# 52. Varnish

Varnish is a caching reverse proxy.

A traditional forward proxy such as Squid sits near users:

```text
client → proxy/cache → internet
```

Varnish reverses the direction:

```text
client → Varnish → origin server
```

It caches responses close to the origin.

---

# 53. Varnish cache behavior

Cache hit:

```text
client → Varnish
          ↓
      cached response
```

No backend request needed.

Cache miss:

```text
client
↓
Varnish
↓
origin server
↓
response
↓
Varnish caches it
↓
client
```

This reduces backend load and response latency.

---

# 54. Varnish concurrency model

Varnish combines threads with event loops.

Active requests:

```text
worker thread
```

Idle keep-alive connections:

```text
epoll/kqueue waiter
```

So:

```text
threads for work
event loop for waiting
```

This is the same hybrid pattern we saw elsewhere.

---

# 55. nginx

nginx is a major example of event-driven server architecture.

Typical structure:

```text
master process
├── worker 1
├── worker 2
├── worker 3
└── worker N
```

Often approximately one worker per CPU core.

Each worker runs an event loop, commonly using `epoll` on Linux.

Each worker can manage thousands of sockets.

---

# 56. nginx + `sendfile()`

nginx uses event-driven networking and efficient file transfer.

Instead of:

```text
file → nginx user buffer → socket
```

it can use:

```text
file → kernel → socket
```

via `sendfile()`.

This is particularly efficient for static content.

---

# 57. nginx as reverse proxy

nginx also became a general reverse proxy.

Architecture:

```text
client
↓
nginx
↓
application server
```

It can provide:

- TLS termination
- caching
- load balancing
- rate limiting
- static serving
- buffering
- compression
- routing

It effectively became a common edge layer.

---

# 58. nginx and blocking disk I/O

A single event-loop worker has one important weakness:

if it performs a blocking operation, all connections handled by that worker may pause.

Example:

```text
worker
↓
slow disk/page fault
↓
worker blocks
↓
its other sockets wait
```

Therefore nginx introduced thread pools for operations that may block.

Architecture:

```text
event-loop worker
↓
offload blocking task
↓
thread pool
```

The worker remains available for network events.

---

# 59. Apache evolution

Apache's concurrency model evolved through several forms:

```text
prefork
↓
worker
↓
event
```

Later/event models combine:

- processes for isolation
- threads for workers
- event-driven socket waiting

Apache also historically popularized:

- modules
- CGI application execution
- extensible server functionality

---

# 60. Convergence of server architectures

Apache, HAProxy, Varnish, and nginx started from different goals:

```text
Apache  → general web server/application execution
HAProxy → load balancing
Varnish → caching
nginx   → scalable web serving/reverse proxy
```

Yet they converged toward similar patterns:

```text
small worker pool
+
event-driven socket handling
```

This is not accidental.

The architecture follows from the economics of large-scale connections:

```text
waiting is cheap if multiplexed
dedicated threads/processes are expensive
```

Therefore:

> Do not dedicate expensive execution resources to idle connections.

---

# 61. The central design pattern of this entire section

The course repeatedly arrives at one principle:

```text
use expensive resources for work
use cheap multiplexing for waiting
```

Examples:

```text
Varnish:
worker thread for active work
epoll waiter for idle sockets

nginx:
event loop for sockets
thread pool for blocking disk work

production servers:
worker processes for parallelism
event loop inside each worker

FastCGI:
persistent worker processes
web server handles slow network connections
```

---

# 62. One more unifying principle: pay setup cost once

CGI:

```text
setup on every request
```

Bad at scale.

FastCGI:

```text
start workers once
reuse them
```

Servlets:

```text
start JVM once
reuse threads
```

Pre-forking:

```text
start processes once
reuse them
```

Connection pooling:

```text
connect once
reuse connection
```

General rule:

> Expensive initialization should be moved out of the hot request path whenever possible.

---

# 63. Important distinctions to memorize

Do not confuse these pairs.

`FD_SETSIZE` vs `ulimit -n`

```text
FD_SETSIZE:
select() data-structure limit

ulimit -n:
process open-file-descriptor limit
```

Syscall vs context switch:

```text
syscall:
user mode ↔ kernel mode

context switch:
thread/process A ↔ thread/process B
```

CGI vs FastCGI:

```text
CGI:
new process per request
pipes + environment variables

FastCGI:
persistent process
socket + typed records
```

Forward proxy vs reverse proxy:

```text
forward proxy:
acts for clients

reverse proxy:
acts in front of servers
```

Stateless HTTP vs persistent DB sessions:

```text
HTTP request:
independently routable

DB connection:
may contain connection/session state
```

---

# 64. Short exam-ready definitions

I/O multiplexing:
A technique where one thread monitors many sockets and only handles those that are ready.

Event loop:
A loop that waits for I/O events and dispatches work when descriptors become ready.

`select()`:
A classic I/O multiplexing system call that scans sets of descriptors and has fixed-size fd-set limitations.

`epoll`:
Linux event notification mechanism designed for efficiently monitoring large numbers of file descriptors.

`fork()`:
Creates a child process. Returns `0` in the child and the child's PID in the parent.

CGI:
An interface where a web server launches an external process and exchanges request/response information through environment variables and standard streams.

FastCGI:
A persistent-process version of CGI that sends structured request/response records over sockets.

Servlet:
Java server-side component executed inside a long-running servlet container/JVM.

Reverse proxy:
A server that receives client requests and forwards them to backend/origin servers.

Load balancer:
A component that distributes requests or connections across multiple backend servers.

Caching reverse proxy:
A reverse proxy that can serve cached responses without contacting the origin.

`sendfile()`:
A system call that transfers file contents toward a socket while avoiding an unnecessary user-space copy.

Little's Law:
In steady state:

```text
number in system = arrival rate × time in system
```

For servers:

```text
concurrency ≈ throughput × latency
```

---
