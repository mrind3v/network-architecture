# Nginx Deep Dive — Consolidated Notes

## 1. `sendfile()`: four copies become two

Suppose nginx needs to send a static file to a client.

With ordinary `read()` + `write()`:

```text
disk
 ↓
kernel page cache
 ↓
nginx user-space buffer
 ↓
kernel socket buffer
 ↓
NIC
```

Conceptually nginx does:

```c
read(file_fd, user_buffer);
write(socket_fd, user_buffer);
```

The inefficient part is:

```text
page cache → nginx buffer → socket buffer
```

The bytes enter nginx's address space even though nginx does not need to inspect or modify them.

This means CPU work is spent copying:

```text
kernel → user space
user space → kernel
```

There are also user/kernel mode transitions from the `read()` and `write()` calls.

### `sendfile()`

With `sendfile()`, nginx essentially tells the kernel:

```text
"Send these bytes from this file to this socket."
```

The path becomes approximately:

```text
disk
 ↓
page cache
 ↓
socket/network path
 ↓
NIC
```

The file contents do not need to enter nginx's user-space address space.

Therefore:

```text
read + write:
kernel → nginx → kernel → network

sendfile:
kernel ─────────→ network
```

The slide describes this as reducing four copies to two, with the remaining transfers being handled through kernel/hardware mechanisms such as DMA rather than CPU-driven user-space copying.

### Main advantage

`sendfile()` reduces:

- CPU copying
- syscalls
- mode switches
- memory movement through nginx's own address space

This matches nginx's general design philosophy:

> Do not make the application process perform work that the kernel/hardware can do more efficiently.

### Why `sendfile()` cannot be used for everything

`sendfile()` is ideal when nginx does not need to inspect or transform the bytes.

If nginx has to modify the response, the data may need to enter user space.

Examples:

```text
gzip compression
templating
application-level transformation
some forms of TLS processing
```

This is why features such as gzip and `sendfile()` may conflict in some paths, and why kernel TLS exists as an optimization for encrypted transfers.

---

# 2. `sendfile()` benchmarks: faster architecture does not always mean faster single request

The slide benchmarked a 64 MB transfer over loopback.

Results were roughly:

```text
read()      wall ≈ 37.4 ms
mmap()      wall ≈ 23.3 ms
sendfile()  wall ≈ 33.2 ms
```

So `sendfile()` did not have the best wall-clock time.

That does not contradict its architectural advantage.

## Why?

The test uses loopback:

```text
sender process
 ↓
kernel
 ↓
receiver process
```

There is no real NIC involved.

Therefore the workload becomes heavily dependent on:

```text
memory copies
memory bandwidth
CPU contention
```

The kernel still has to copy data into the receiving application's buffer.

So a loopback benchmark does not model the exact bottleneck `sendfile()` is designed to eliminate in a real network transfer.

## What still improved dramatically?

Syscalls:

```text
read/write approach → 2048 syscalls
sendfile            → 1 syscall
```

And the 64 MB file never entered the sender's user-space address space.

## The important systems lesson

The benefit of `sendfile()` may appear as additional system capacity rather than lower latency for one isolated request.

For example:

```text
without sendfile:
CPU spends time copying bytes

with sendfile:
CPU can spend that time serving another connection
```

So under concurrency, the system gets more CPU headroom.

This distinction is important:

```text
single-request latency
≠
overall throughput / concurrency capacity
```

### Benchmarking lesson

A benchmark is meaningful only if it models the actual bottleneck.

A loopback benchmark primarily measures memory behavior, not a realistic NIC/network path.

---

# 3. Nginx proxy cache: two structures, one on disk

Example configuration:

```nginx
proxy_cache_path temp/cache
    levels=1:2
    keys_zone=demo:10m
    max_size=100m
    inactive=60s
    use_temp_path=off;
```

The cache is conceptually split into:

```text
shared memory → cache metadata/index
disk          → actual cached response data
```

This distinction is critical.

---

## `keys_zone=demo:10m`

Creates a shared-memory zone named `demo` of size 10 MB.

It stores cache metadata, not the actual response body.

The shared memory holds structures such as:

```text
cache lookup index
red-black tree
LRU queue
cache metadata
```

The slide gives a rough estimate of about 8,000 cache keys per MB.

Why shared memory?

Because nginx has multiple worker processes. They need a common cache index.

Conceptually:

```text
worker 1 ─┐
worker 2 ─┼→ shared cache metadata
worker 3 ─┘
```

---

## `levels=1:2`

This controls how nginx distributes cached files into directories on disk.

Without directory levels, you could end up with enormous numbers of files in one directory:

```text
cache/
  file1
  file2
  file3
  ...
  file1000000
```

That can hurt filesystem lookup performance.

Instead nginx creates nested directories, conceptually:

```text
cache/
  a/
    3f/
      cached-file
```

`1:2` means:

```text
first directory level  → 1 character
second directory level → 2 characters
```

---

## `max_size=100m`

This is the disk-space ceiling for the cache.

Important:

```text
keys_zone=10m → shared-memory metadata capacity

max_size=100m → actual disk cache capacity
```

They control different resources.

A cache-manager process removes old objects when the disk cache grows beyond its configured limit.

LRU information helps nginx decide what to evict.

---

## `inactive=60s`

This means a cached object that has not been accessed for 60 seconds may be removed.

This is not the same as TTL/freshness.

---

# 4. `inactive` versus `proxy_cache_valid`

Suppose:

```nginx
proxy_cache_valid 200 10s;
inactive=60s;
```

These control different things.

### `proxy_cache_valid`

Answers:

```text
"How long is this response considered fresh?"
```

After 10 seconds, the response becomes stale.

### `inactive`

Answers:

```text
"How long may this cache object go unused before nginx removes it?"
```

An object may be stale but still remain physically stored.

Example:

```text
t=0   cached
t=10  becomes stale
t=20  accessed
t=40  accessed again
```

It may continue to exist because it is being used.

Conversely, something with a long freshness lifetime can still be removed because it has been inactive.

Therefore:

```text
proxy_cache_valid → freshness
inactive          → retention based on use
```

There is no simple "which one wins?" because they govern different properties.

---

# 5. `proxy_cache_lock`: preventing cache stampedes

Configuration:

```nginx
proxy_cache_lock on;
```

Suppose a popular cached object expires and 500 requests arrive immediately.

Without locking:

```text
500 requests
 ↓
all see cache MISS/expired
 ↓
all 500 contact origin
```

This is a cache stampede.

It can overload the origin server exactly when traffic is high.

With `proxy_cache_lock on`:

```text
500 requests
 ↓
one request goes upstream
 ↓
others wait
 ↓
fresh copy arrives
 ↓
waiting requests use it
```

So `proxy_cache_lock` is an anti-stampede mechanism.

---

# 6. `proxy_cache_use_stale`

Example:

```nginx
proxy_cache_use_stale
    error timeout updating
    http_500 http_502 http_503 http_504;
```

This tells nginx that under certain conditions it may serve an expired cached response rather than fail the request.

Example:

```text
cached page exists
but it is stale

origin now returns 502
```

Without stale serving:

```text
client → 502 Bad Gateway
```

With stale serving:

```text
client → slightly outdated cached response
```

This improves availability when the upstream is unhealthy.

The conditions in the directive specify when stale content is acceptable.

For example:

```text
error
timeout
http_500
http_502
http_503
http_504
```

---

## `updating`

`updating` is particularly useful while one request is refreshing the cached entry.

Without stale-while-updating:

```text
request A → refreshes cache
request B → waits
request C → waits
request D → waits
```

With stale-while-updating:

```text
request A → refreshes cache

request B → immediately gets stale response
request C → immediately gets stale response
request D → immediately gets stale response
```

So users continue receiving responses while the refresh happens in the background.

Together:

```text
proxy_cache_lock
→ prevent many simultaneous refreshes

proxy_cache_use_stale
→ continue serving something useful during failures/updates
```

---

# 7. Seeing whether nginx actually used the cache

A useful debugging configuration is:

```nginx
add_header X-Cache-Status $upstream_cache_status;
```

Then:

```text
first request:
X-Cache-Status: MISS

later request:
X-Cache-Status: HIT
```

Meaning:

```text
MISS
→ nginx did not have a usable cached response

HIT
→ nginx served it from cache
```

This is useful because without observability you may believe caching is working while every request is actually going upstream.

---

# 8. Nginx `location` precedence

The order in which `location` blocks are written does not generally determine which one wins.

Nginx has explicit matching rules.

Examples:

```nginx
location = /exact { ... }

location ^~ /images/ { ... }

location ~ \.php$ { ... }

location / { ... }
```

---

## Exact location: `=`

```nginx
location = /exact
```

Request:

```text
/exact
```

An exact match wins immediately and location searching stops.

---

## Prefix location with `^~`

```nginx
location ^~ /images/
```

Request:

```text
/images/a.php
```

Both of these could appear to match:

```nginx
location ^~ /images/
location ~ \.php$
```

But `^~ /images/` wins.

Meaning of `^~`, conceptually:

```text
"If this is the best prefix match,
do not continue into regex matching."
```

---

## Regex location: `~`

```nginx
location ~ \.php$
```

For:

```text
/other/a.php
```

the regex matches.

Among regex locations:

> The first matching regex in configuration order wins.

Nginx does not choose the longest or "most specific" regex.

---

## Ordinary prefix

```nginx
location /
```

This is a prefix location and acts as a broad fallback.

For example:

```text
/anything
```

may match it if nothing more specific wins.

---

## Mental algorithm

A useful simplified model is:

```text
1. Exact match (=)?
   → use it immediately

2. Find best/longest prefix match

3. If that prefix uses ^~
   → use it and skip regexes

4. Otherwise test regex locations
   → first matching regex wins

5. If no regex matches
   → use the best prefix match
```

Examples:

```text
/exact
→ location = /exact

/images/a.php
→ location ^~ /images/

/other/a.php
→ location ~ \.php$

/anything
→ location /
```

---

# 9. Why nginx has no `.htaccess`

Apache can use `.htaccess` files distributed through directories.

That can require filesystem checks while resolving a request.

Nginx instead loads its configuration centrally.

The slide's architectural point is that avoiding `.htaccess` helps nginx avoid repeatedly walking directories and performing `stat()`-style checks for configuration on every request.

---

# 10. Why many prefix locations are cheap but many regexes are not

Prefix locations and regex locations are stored differently.

## Prefix locations

Examples:

```nginx
location /images/ { ... }
location /api/ { ... }
location /static/ { ... }
```

Nginx organizes prefix locations into a search tree when configuration is parsed.

The source code performs operations conceptually like:

```c
if (rc != 0) {
    node = (rc < 0) ? node->left : node->right;
}
```

Meaning:

```text
compare URI with current tree node
↓
go left or right
```

If part of the URI matches, nginx can descend deeper.

This makes prefix lookup efficient.

Having 500 prefix locations does not generally mean checking 500 entries sequentially.

That is why the slide calls many prefixes effectively "free" relative to regex scanning.

Not literally zero cost — just structurally efficient.

---

## Regex locations

Regex locations are stored essentially in an ordered array/list.

Nginx conceptually does:

```text
try regex 1
↓ no match
try regex 2
↓ no match
try regex 3
...
```

Therefore the cost can increase with the number of regexes evaluated.

This explains the previous matching rule:

```text
first matching regex wins
```

It corresponds naturally to sequential scanning.

---

## Data structure explains the semantics

Prefix:

```text
tree
→ search for best prefix
```

Regex:

```text
ordered list
→ first successful match wins
```

Hence the slide's message:

> The data structure is the documentation.

Knowing how nginx stores these rules makes its location precedence much less arbitrary.

---

# 11. `try_files`: filesystem checks followed by fallback

Example:

```nginx
location / {
    try_files $uri $uri/ /index.html?$args;
}
```

Read left to right:

```text
1. Does $uri identify a real file?
2. If not, does $uri/ identify a real directory?
3. If neither works, use /index.html?$args
```

Example request:

```text
/products/42
```

Nginx may check:

```text
/products/42
/products/42/
```

If neither exists, it internally redirects to:

```text
/index.html
```

while preserving the query arguments through `$args`.

---

# 12. Important clarification: `/index.html` is not automatically an application server

In:

```nginx
try_files $uri $uri/ /index.html;
```

`/index.html` normally refers to a file nginx can serve from the configured filesystem root.

Typical SPA flow:

```text
request /products/42
 ↓
no matching physical file
 ↓
no matching physical directory
 ↓
serve /index.html
 ↓
browser loads JavaScript SPA
 ↓
client-side router handles /products/42
```

So the fallback here is nginx's local static file.

This is common with React, Vue, Angular, etc.

---

# 13. Fallback to an upstream application

Another form is:

```nginx
location /maybe/ {
    try_files $uri @app;
}

location @app {
    proxy_pass http://app;
}
```

Flow:

```text
request
 ↓
try local file
 ↓ not present
@app
 ↓
proxy_pass
 ↓
application server
```

`@app` is a named location.

A named location is an internal nginx destination. It is not normally something the client requests directly through a URL.

It can be reached from mechanisms such as:

```text
try_files
error_page
```

Therefore:

```text
try_files $uri /index.html;
→ fallback is local nginx content

try_files $uri @app;
→ fallback goes to upstream application
```

`try_files` itself does not mean "go to application."

Its meaning is:

> Try these filesystem candidates, and if they fail, use the final fallback.

The nature of the last argument determines what happens.

---

# 14. Nginx source tree

The slide gives a high-level map:

```text
src/core/
src/event/
src/http/
src/os/unix/
src/stream/
src/mail/
```

---

## `src/core/`

Contains nginx's internal foundational infrastructure.

Examples include:

```text
strings
arrays
lists
hash tables
red-black trees
queues
memory pools
buffers
configuration handling
cycle structures
connection structures
```

Nginx implements many of its own utilities because it was designed to remain portable and controlled across operating systems.

---

## `src/event/`

Contains the event-driven I/O machinery.

Different backends include:

```text
Linux      → epoll
BSD/macOS  → kqueue
others     → select, poll, etc.
```

Nginx exposes an internal event abstraction, while the platform-specific event mechanism is selected during configuration/build/runtime setup.

Most higher-level nginx code does not need to care whether the underlying OS is using `epoll` or `kqueue`.

---

## `src/http/`

Contains HTTP-specific logic:

```text
HTTP parser
HTTP state machine
phase engine
location routing
upstream framework
proxying
cache
try_files
HTTP modules
```

Many configuration directives correspond to modules in this area.

---

## `src/os/unix/`

Contains lower-level Unix-specific operations.

Examples:

```text
sendfile
process spawning
shared memory
atomic operations
system calls
OS-specific behavior
```

---

## `src/stream/`

Implements raw TCP/UDP functionality.

This lets nginx act as a Layer-4 proxy/load balancer.

Example:

```text
client
  ↓ TCP
nginx
  ↓ TCP
backend
```

Nginx does not necessarily need to interpret HTTP in this mode.

---

## `src/mail/`

Implements proxying for protocols such as:

```text
IMAP
POP3
SMTP
```

---

## Architectural point

Nginx reuses one core architecture:

```text
            core
             +
         event system
             |
     -----------------
     |       |       |
    HTTP   Stream   Mail
```

There are different protocol layers, but the same basic event-driven foundation underneath.

---

# 15. `ngx_queue.h`: intrusive doubly linked lists

One important nginx primitive is:

```text
src/core/ngx_queue.h
```

It implements an intrusive doubly linked list.

A normal list may have separate list-node objects:

```text
list node
 ├── previous
 ├── next
 └── pointer to object
```

With an intrusive list, the actual object contains its own link field:

```c
struct object {
    ...
    ngx_queue_t queue;
    ...
};
```

So the linkage is embedded directly inside the object.

Benefits:

```text
no extra list-node allocation
no separate list-node free
less indirection
cheap insertion/removal
```

Given a pointer to the embedded `ngx_queue_t`, nginx can recover the enclosing object using pointer arithmetic.

This queue mechanism is used for things such as:

```text
cache LRU
posted event queues
many internal core structures
```

---

# 16. `ngx_palloc.h`: nginx memory pools

Another core abstraction is:

```text
src/core/ngx_palloc.h
```

Nginx often avoids doing:

```text
malloc
free
malloc
free
...
```

for lots of small request objects.

Instead:

```text
request starts
 ↓
create/attach request pool
 ↓
allocate request-related data from pool
 ↓
request finishes
 ↓
destroy entire pool
```

Conceptually:

```text
ordinary allocation:
allocate A
allocate B
allocate C
free A
free B
free C

pool allocation:
create pool
allocate A, B, C
destroy pool
```

Advantages include:

```text
less per-object cleanup
fewer free() calls
simpler ownership
cheap request teardown
```

This is why the slide says there is little `free()` on the hot path.

The tradeoff is that an allocation may remain alive until the whole pool is destroyed, even if nginx no longer needs that particular object.

---

# 17. Why these two files matter

A large amount of nginx code assumes familiarity with:

```text
ngx_queue_*  → intrusive data structures
ngx_palloc_* → pool-based allocation
```

Once you understand those abstractions, nginx source starts to look less like arbitrary C macros and more like code built on a small internal runtime.

---

# 18. Ten important source-code stops

The source tour maps architectural concepts to concrete files/functions.

## 1. Process model

```text
file:
os/unix/ngx_process_cycle.c

function:
ngx_start_worker_processes
```

This corresponds to:

```text
master
├── worker
├── worker
└── worker
```

The master creates the worker processes.

---

## 2. Main event loop

```text
file:
event/ngx_event.c

function:
ngx_process_events_and_timers
```

Corresponds to:

```text
find next timer
↓
wait for events
↓
process events
↓
expire timers
↓
repeat
```

---

## 3. Accepting connections

```text
file:
event/ngx_event_accept.c

function:
ngx_event_accept
```

Related to:

```text
accepting new connections
accept coordination
thundering herd
```

---

## 4. `epoll` and stale events

```text
file:
event/modules/ngx_epoll_module.c

function:
ngx_epoll_process_events
```

This contains Linux `epoll` event processing and connects to the stale-event/generation-bit mechanism you studied earlier.

---

## 5. HTTP parser

```text
file:
http/ngx_http_parse.c

function:
ngx_http_parse_request_line
```

Parses request lines such as:

```text
GET /images/a.png HTTP/1.1
```

---

## 6. HTTP phases

```text
file:
http/ngx_http_core_module.c

function:
ngx_http_core_run_phases
```

Controls nginx's HTTP processing pipeline.

---

## 7. Routing

```text
file:
http/ngx_http_core_module.c

function:
ngx_http_core_find_static_location
```

Implements prefix location lookup and related routing behavior.

---

## 8. `sendfile`

```text
file:
os/unix/ngx_linux_sendfile_chain.c

function:
ngx_linux_sendfile
```

Implements Linux static-file transfer optimization.

---

## 9. Cache LRU

```text
file:
http/ngx_http_file_cache.c

function:
ngx_http_file_cache_expire
```

Handles cache expiration and eviction.

---

## 10. FastCGI records

```text
file:
http/modules/ngx_http_fastcgi_module.c

function:
ngx_http_fastcgi_process_record
```

Handles FastCGI wire-protocol records.

---

# 19. HTTP parser: resumable state machine

An HTTP request line might be:

```text
GET /images/a.png HTTP/1.1
```

A simple blocking parser could assume the complete string has arrived.

Nginx cannot make that assumption.

TCP may deliver:

```text
segment 1: GE

segment 2: T /images/a.

segment 3: png HTTP/1.1
```

Nginx uses non-blocking I/O, so when only the first piece is available, it must not block waiting for the rest.

Instead it uses a resumable state machine.

---

## Parser states

The source defines many states, for example:

```text
sw_start
sw_method
sw_spaces_before_uri
sw_schema
sw_host
sw_uri
sw_http_H
...
```

The request line alone has many parsing states.

The parser reads approximately one byte at a time and transitions between states.

Example:

```text
state = METHOD

read 'G'
read 'E'
read 'T'
read ' '

→ transition to URI state
```

---

## Saving state

Important line:

```c
state = r->state;
```

`r->state` persists between parser invocations.

Suppose nginx receives:

```text
GE
```

The parser knows it is currently reading a method.

Then it runs out of available bytes.

It saves:

```text
state = sw_method
```

and returns to the event loop.

Later:

```text
socket becomes readable
↓
nginx parser called again
↓
state restored from r->state
↓
parsing continues with T...
```

So:

```text
receive some bytes
↓
parse as far as possible
↓
save state
↓
return
↓
receive more bytes later
↓
resume
```

---

## Why not use `strtok()` or regex parsing?

Because the complete request string may not exist yet.

The parser therefore does not depend on:

```text
regex
strtok
substring allocation
complete request buffering
```

It incrementally processes available bytes.

---

## Architectural lesson

This is the source-code cost of non-blocking I/O:

> Any computation that can be interrupted by unavailable I/O must be resumable.

This matches nginx's general callback model:

```text
do available work
↓
would need to wait
↓
save progress
↓
return to event loop
↓
resume later
```

---

# 20. Nginx HTTP request phases

After parsing, nginx processes a request through a phase pipeline.

The slide lists:

```text
POST_READ
→ SERVER_REWRITE
→ FIND_CONFIG
→ REWRITE
→ POST_REWRITE
→ PREACCESS
→ ACCESS
→ POST_ACCESS
→ PRECONTENT
→ CONTENT
→ LOG
```

Different modules attach handlers to different phases.

Important examples:

```text
limit_req
→ PREACCESS

auth_basic
→ ACCESS

try_files
→ PRECONTENT

proxy_pass
→ CONTENT

fastcgi_pass
→ CONTENT

static serving
→ CONTENT
```

Conceptually:

```text
parse request
↓
rewrite
↓
rate limiting
↓
authentication/access
↓
try_files
↓
produce/proxy content
↓
logging
```

This means nginx directives do not simply execute according to the textual order in which they appear in configuration.

Their modules execute in the phase where they registered themselves.

---

# 21. Phase processing can suspend and resume

Core logic is conceptually:

```c
while (ph[r->phase_handler].checker) {
    rc = ph[r->phase_handler].checker(
        r,
        &ph[r->phase_handler]
    );

    if (rc == NGX_OK) {
        return;
    }
}
```

`r->phase_handler` remembers where the request currently is in the phase pipeline.

Suppose the CONTENT phase performs:

```text
proxy_pass
```

Nginx contacts the upstream but the upstream response is not ready yet.

Nginx must not block.

It therefore:

```text
starts upstream operation
↓
sets callback/event state
↓
returns
↓
worker handles other connections
```

Later:

```text
upstream becomes readable
↓
epoll wakes nginx
↓
callback runs
↓
request processing continues
```

So in this context:

```text
return
```

does not necessarily mean:

```text
"the request is completely finished"
```

It can mean:

```text
"processing is suspended until another event occurs"
```

---

# 22. Nginx as a hand-rolled coroutine system

A coroutine can:

```text
run
↓
suspend
↓
later resume
```

Modern code might express that using mechanisms such as:

```text
async / await
```

Nginx implements the same broad idea manually in C using:

```text
state fields
callbacks
event loop
phase handlers
```

The parser and HTTP phase engine are both examples.

Parser:

```text
parse
→ suspend
→ resume
```

Phase engine:

```text
run request phases
→ suspend for I/O
→ resume
```

This is one of the most important architectural ideas in the source code.

---

# 23. FastCGI framing

FastCGI communication consists of records.

Each record starts with an 8-byte header.

Fields shown on the slide:

```text
version
type
requestId
contentLength
paddingLength
reserved
```

After the header:

```text
contentData[contentLength]
```

followed by optional padding.

So:

```text
8-byte header
↓
contentLength bytes
↓
padding
```

---

# 24. Length-based framing

The `contentLength` field tells the receiver exactly how many bytes belong to the record.

If:

```text
contentLength = 100
```

the receiver knows:

```text
"the next 100 bytes are this record's payload."
```

This answers:

> Where does this individual record end?

---

# 25. FastCGI also uses empty records as delimiters

Some logical FastCGI streams consist of multiple records.

Examples include:

```text
PARAMS
STDIN
```

Suppose nginx sends several `PARAMS` records containing request/environment information.

How does the application know there will be no more `PARAMS` records?

Nginx sends an empty `PARAMS` record:

```text
type = PARAMS
contentLength = 0
```

That means:

```text
end of PARAMS stream
```

Similarly:

```text
empty STDIN record
→ end of request body
```

So FastCGI uses two framing mechanisms:

```text
contentLength
→ tells you where one record ends

empty record
→ tells you where the logical stream ends
```

---

# 26. Why forgetting the empty record is serious

If nginx sends:

```text
PARAMS
PARAMS
PARAMS
```

but never sends:

```text
empty PARAMS
```

the FastCGI application may keep waiting because it has never received the signal:

```text
"no more parameters are coming"
```

Likewise for request-body `STDIN`.

---

# 27. Example FastCGI request flow

Conceptually:

```text
BEGIN_REQUEST

PARAMS
PARAMS
PARAMS
PARAMS(empty)
    ↑
    end of parameter stream

STDIN(body chunk)
STDIN(body chunk)
STDIN(empty)
    ↑
    end of request-body stream
```

This is why the slide says FastCGI framing uses both length and delimiter.

---

# 28. The architecture ladder

The final recap slide summarizes the evolution of server architecture.

## Stage 1: fork per client

```text
01-fork
fork() per client
```

Model:

```text
client 1 → process 1
client 2 → process 2
client 3 → process 3
```

Easy to understand but expensive.

Processes have significant memory and scheduling overhead.

---

## Stage 2: thread per client

```text
02-thread
pthread per client
```

Model:

```text
client 1 → thread 1
client 2 → thread 2
```

Threads share memory and are cheaper than processes.

But every thread still requires resources such as:

```text
stack
scheduler state
register state
thread-local storage
```

At very large concurrency, this remains expensive.

---

## Stage 3: one thread + `select()`

```text
03-select
one thread, select()
```

Now:

```text
many clients
    ↓
one event loop/thread
    ↓
select()
```

This is the major conceptual transition:

> Stop assigning one execution context to every connection.

But `select()` has scaling problems.

You studied:

```text
select → cost tied to watched set
```

and the traditional:

```text
FD_SETSIZE ≈ 1024
```

limit.

Increasing the OS file-descriptor limit does not fix the fundamental bitmap limitation of `select()`.

---

## Stage 4: one thread + `epoll()`

```text
04-epoll
one thread, epoll()
```

Now the kernel persistently watches many sockets.

Nginx asks:

```text
"Which sockets are ready?"
```

instead of scanning the entire set.

Conceptually:

```text
thousands of connections
↓
epoll_wait()
↓
small set of ready sockets
```

This is ideal when:

```text
many connections are open
but only a small number are active
```

That is the C10K-style workload.

---

## Stage 5: `sendfile()`

Once connection concurrency is cheap, another cost remains:

```text
moving file bytes
```

`sendfile()` reduces unnecessary copying and syscalls.

So the optimization ladder progresses from:

```text
make waiting cheap
```

to:

```text
make transferring bytes cheap
```

---

## Stage 6: FastCGI

Classic CGI:

```text
request
↓
fork
↓
exec
↓
start runtime
↓
run request
↓
exit
```

FastCGI:

```text
start backend once
↓
request
request
request
request
```

Hence the slide's phrase:

```text
CGI without the fork
```

It moves expensive process/runtime startup off the per-request path.

---

## Stage 7: nginx

Real nginx combines the ideas:

```text
fixed workers
event-driven sockets
epoll/kqueue
routing
sendfile
proxying
caching
load balancing
FastCGI
fallbacks
```

The point is that nginx is not based on one single trick.

It is the combination of several resource-efficiency ideas.

---

# 29. Why nginx source is much larger than toy implementations

The slide contrasts roughly:

```text
course demos → ~1,200 lines
real nginx   → ~200,000 lines
```

The key idea is that a simplified implementation can ignore many production failure paths.

Real software must deal with:

```text
partial reads
partial writes
timeouts
client disconnects
upstream disconnects
malformed input
failed syscalls
memory allocation failures
OS differences
retries
cleanup paths
logging
race conditions
security cases
resource exhaustion
```

Therefore:

> The basic architecture can be conceptually compact even though the production implementation is large.

Production complexity often comes from correctness, portability and failure handling rather than from the central architecture itself.

---

# 30. The major principles connecting all these slides

These are the ideas that repeatedly appear across the material.

### Principle 1: Do not reserve expensive resources for idle things

Old architecture:

```text
1 connection → 1 process/thread
```

Nginx:

```text
many connections → small worker set + event loop
```

The kernel waits for activity.

---

### Principle 2: Make computation resumable rather than blocking

Parser:

```text
parse what arrived
→ save state
→ return
→ resume later
```

Phase engine:

```text
process request
→ need I/O
→ suspend
→ resume later
```

Connection callback:

```text
advance connection state
→ return to loop
```

This is the heart of event-driven programming.

---

### Principle 3: Keep expensive work off the hot path

Examples:

```text
FastCGI
→ start application once instead of per request

memory pools
→ destroy one pool instead of individually freeing objects

location tree
→ precompute/search efficiently

cache metadata
→ shared index rather than repeatedly rebuilding it
```

---

### Principle 4: Let the right layer do the work

Examples:

```text
epoll
→ kernel watches sockets

sendfile
→ kernel moves file data

DMA
→ hardware moves bytes

nginx static serving
→ avoid waking application server unnecessarily
```

---

### Principle 5: Optimize active work, not total open state

This is a central nginx idea.

A server might have:

```text
100,000 open connections
```

but only:

```text
100 active right now
```

The goal is for cost to scale closer to:

```text
active work
```

rather than:

```text
total open connections
```

---

### Principle 6: Data structures often explain behavior

Examples:

```text
prefix locations
→ tree
→ longest/best prefix semantics

regex locations
→ ordered sequence
→ first matching regex semantics

cache
→ rbtree + LRU queue

posted events
→ intrusive queues
```

Understanding the data structure often explains why nginx behaves the way it does.

---

# 31. High-yield comparisons for the exam

## `read/write` vs `sendfile`

```text
read/write:
kernel → user → kernel

sendfile:
kernel → network path

Result:
less copying, fewer syscalls, more CPU headroom
```

---

## `proxy_cache_valid` vs `inactive`

```text
proxy_cache_valid
→ freshness lifetime

inactive
→ remove if unused for this long
```

---

## `proxy_cache_lock` vs `proxy_cache_use_stale`

```text
proxy_cache_lock
→ prevent cache stampede

proxy_cache_use_stale
→ serve expired object during configured failure/update conditions
```

---

## Prefix location vs regex location

```text
prefix
→ tree lookup
→ best/longest prefix

regex
→ sequential search
→ first match
```

---

## `/index.html` fallback vs `@app` fallback

```text
try_files $uri /index.html
→ usually local nginx file

try_files $uri @app
→ named location
→ may proxy to application
```

---

## Blocking parser vs nginx parser

```text
blocking:
wait until complete input exists

nginx:
consume available input
save state
resume later
```

---

## Record length vs empty FastCGI record

```text
contentLength
→ end of current record

empty PARAMS / STDIN
→ end of logical stream
```

---

## `select()` vs `epoll()`

```text
select
→ repeatedly deals with watched set
→ scales poorly with huge mostly-idle sets

epoll
→ persistent kernel interest set
→ returns ready events
```

---

# 32. A request through nginx — combined mental model

This is a useful end-to-end picture for long-answer questions.

```text
client connects
        ↓
kernel detects connection
        ↓
nginx worker accepts it
        ↓
socket registered/watched by event mechanism
        ↓
request bytes arrive
        ↓
epoll wakes worker
        ↓
HTTP parser consumes available bytes
        ↓
if incomplete:
save parser state
return to event loop
        ↓
later resume
        ↓
request completely parsed
        ↓
HTTP phase engine starts
        ↓
rewrite / access / precontent / content...
        ↓
location/routing determines handling
        ↓
possible outcomes:

static file
    → maybe sendfile()

cached proxy response
    → serve cache

cache miss
    → contact upstream

FastCGI
    → send FastCGI records

reverse proxy
    → contact application server

try_files failure
    → fallback to index.html or @app
        ↓
if any operation would block:
save state
return to event loop
        ↓
resume when event fires
        ↓
send response
        ↓
LOG phase
```

This diagram ties most of the studied material together.

---

# 33. One-paragraph exam summary

Nginx is built around a small number of long-lived worker processes, each using event-driven non-blocking I/O to manage many connections concurrently. Instead of assigning a process or thread to every connection, nginx relies on kernel mechanisms such as `epoll` to notify workers only when sockets require attention. Because operations can stop whenever data is unavailable, nginx's parsers, connection handlers and HTTP phase engine are explicitly resumable state machines. Its design repeatedly avoids unnecessary work: `sendfile()` avoids copying static files through user space; FastCGI keeps application processes alive across requests; proxy caching reduces origin load; cache locks prevent stampedes; stale responses preserve availability; prefix locations use efficient tree lookup; intrusive queues avoid extra allocation; and request memory pools enable bulk cleanup. The overall design aims to make resource cost scale with active work rather than with the total number of open connections.

# 34. Shortest last-minute revision sheet

If you only had a few minutes before the exam, remember:

```text
nginx philosophy:
don't dedicate expensive resources to idle connections.

epoll:
kernel watches many sockets and returns ready ones.

non-blocking:
do available work → save state → return → resume later.

HTTP parser:
resumable state machine.

HTTP phase engine:
request passes through phases; modules register in particular phases.

sendfile:
file data avoids nginx user-space buffer; fewer copies/syscalls.

cache:
shared memory = metadata/index
disk = actual cached content.

proxy_cache_valid:
freshness.

inactive:
unused-retention timeout.

proxy_cache_lock:
one request refreshes; prevents stampede.

proxy_cache_use_stale:
serve old content during configured failures/updates.

locations:
exact = wins immediately
^~ = best prefix and suppress regex
regex = first matching regex
ordinary prefix = fallback if regex doesn't win.

prefix locations:
tree.

regex locations:
sequential array/list.

try_files:
test filesystem candidates; final item is fallback.

/index.html fallback:
usually local static file.

@app fallback:
internal named location, often proxies upstream.

ngx_queue:
intrusive doubly-linked list.

ngx_palloc:
request-lifetime memory pool.

FastCGI:
long-lived backend instead of process-per-request.

FastCGI framing:
contentLength ends a record;
empty PARAMS/STDIN ends a stream.

overall progression:
process/client
→ thread/client
→ select
→ epoll
→ sendfile
→ FastCGI
→ nginx combines everything.
```

The deepest theme across all of these slides is:

> nginx repeatedly converts “wait while holding a resource” into “store a small amount of state, return to the event loop, and resume only when useful work becomes possible.”