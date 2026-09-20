# Network Architecture — Introduction: Consolidated Notes

## 1. TCP server: the basic lifecycle

A low-level TCP server is essentially built using these system calls:

```text
socket → bind → listen → accept → read/write → close
```

### `socket()`
Creates a networking endpoint.

```c
socket(AF_INET, SOCK_STREAM, 0);
```

- `AF_INET` → IPv4
- `SOCK_STREAM` → TCP

The returned integer is a file descriptor representing the socket.

### `bind()`
Associates the socket with a local IP address and port.

Example:

```c
addr.sin_addr.s_addr = INADDR_ANY;
addr.sin_port = htons(2026);
bind(...);
```

### `listen()`
Turns the socket into a listening socket.

It tells the kernel:

> Incoming TCP connections may arrive on this socket.

### `accept()`
Retrieves one established client connection.

Important distinction:

- `server_fd` → listening socket
- `client_fd` → socket representing one specific client connection

The listening socket continues to exist so more clients can connect.

### `read()` / `write()`
Exchange bytes with the connected client.

### `close()`
Closes a socket.

---

# 2. Simple echo server

An echo server sends back whatever the client sends.

Basic flow:

```text
client connects
      ↓
server accept()
      ↓
server read()
      ↓
server write() same bytes
      ↓
connection closes
```

A simple server could contain:

```c
int client_fd = accept(server_fd, NULL, NULL);

char buf[4096];
int n = read(client_fd, buf, sizeof(buf));

write(client_fd, buf, n);
close(client_fd);
```

The first version performs only one `read()` and one `write()`.

---

# 3. Persistent connection

A real conversation usually requires multiple reads/writes over the same TCP connection.

```c
while ((n = read(client_fd, buf, sizeof(buf))) > 0) {
    write(client_fd, buf, n);
}
```

Interpretation of `read()`:

```text
> 0   bytes successfully received
= 0   peer closed its side of the connection
< 0   error
```

There are therefore usually two loops:

```text
outer loop → accept different clients
inner loop → repeatedly communicate with one client
```

Example:

```text
while forever:
    accept client

    while client sends data:
        read
        write

    close client
```

---

# 4. TCP is a byte stream

One of the most important concepts:

> TCP gives applications a stream of bytes, not messages.

Suppose the sender performs:

```text
write("HELLO")
write("WORLD")
```

The receiver is NOT guaranteed to get:

```text
read → "HELLO"
read → "WORLD"
```

It might receive:

```text
"HELLOWORLD"
```

or:

```text
"HEL"
"LOWO"
"RLD"
```

TCP guarantees ordered, reliable bytes, not preservation of application message boundaries.

This is why application protocols need message framing.

---

# 5. Ports

A server is normally associated with a known port.

Examples:

```text
80    HTTP
443   HTTPS
25    SMTP
21    FTP
110   POP3
```

Historically on Unix-like systems, ports below `1024` are privileged ports.

Normally, binding one requires elevated privilege such as root or an appropriate capability.

Ports >= 1024 can usually be bound by ordinary users.

Example development ports:

```text
3000
8080
2026
6379
```

Why avoid running a complex server as root?

If the server contains a vulnerability and runs as root, an attacker may gain highly privileged access.

A common architecture is:

```text
Internet
   ↓
port 80/443
   ↓
reverse proxy
   ↓
application on 8080/3000
```

---

# 6. Client ports and ephemeral ports

The server usually has a known port such as:

```text
server:443
```

The client generally does not manually choose its port.

When the client calls:

```c
connect(...)
```

the OS automatically selects an available temporary port called an ephemeral port.

Example:

```text
client: 192.168.1.10:52341
server: 142.250.1.1:443
```

A TCP connection is identified by the 4-tuple:

```text
source IP
source port
destination IP
destination port
```

The client's source port is included in TCP packets, so the server automatically learns it.

This allows many clients to connect to the same server port:

```text
10.0.0.1:51001 ─┐
10.0.0.2:52013 ─┼──> server:443
10.0.0.3:60122 ─┘
```

---

# 7. What happens if server calls are skipped?

## No listener on the port

If nothing is listening on a port and a TCP SYN arrives, the kernel commonly returns an RST.

Client sees:

```text
Connection refused
```

Conceptually:

```text
SYN →
    ← RST
```

## `listen()` exists but application never calls `accept()`

The kernel can still perform the TCP handshake.

The established connection waits in the kernel's accept queue.

```text
TCP handshake succeeds
       ↓
connection waits in queue
       ↓
application never accept()s it
```

The client may appear connected but receive no application response.

Important lesson:

> The kernel performs much of TCP connection establishment. `accept()` retrieves an already-established connection.

---

# 8. Signals and `SIGPIPE`

Unix processes can receive signals.

Examples:

```text
SIGINT   commonly Ctrl+C
SIGTERM  request process termination
SIGKILL  forced termination; cannot be caught
SIGPIPE  writing to a broken pipe/socket
```

`SIGPIPE` is particularly important for network servers.

Scenario:

```text
client disconnects/reset connection
        ↓
server later calls write()
        ↓
kernel detects broken connection
        ↓
SIGPIPE
```

The default behavior of `SIGPIPE` is process termination.

Therefore a naïve server can die because one client disconnected.

Production servers usually suppress or ignore `SIGPIPE` and handle the write error instead.

---

# 9. FIN vs RST

TCP connections can end in different ways.

## FIN

A normal, graceful shutdown.

Conceptually:

> I am finished sending data.

## RST

Reset: abrupt termination.

Conceptually:

> This connection is no longer valid.

A socket can be configured with:

```c
struct linger l = {1, 0};
setsockopt(fd, SOL_SOCKET, SO_LINGER, &l, sizeof(l));
```

With this setting, `close()` can produce an abortive close using RST rather than a normal graceful FIN sequence.

This was used in the slides to deliberately trigger broken-connection behavior and demonstrate `SIGPIPE`.

---

# 10. Telnet, Netcat, and OpenSSL as clients

A client is simply a program that opens a TCP connection and sends bytes.

## Telnet

Useful for manually interacting with plaintext TCP protocols.

Example:

```bash
telnet google.com 80
```

You can manually type an HTTP request.

Similarly:

```bash
telnet localhost 2026
```

can talk to the echo server.

The important idea:

> If the application protocol is human-readable text, you can sometimes type the protocol manually.

---

## Netcat (`nc`)

Useful as a generic TCP client:

```bash
nc localhost 2026
```

It can also act as a tiny server:

```bash
nc -l 9000
```

---

## OpenSSL client

For TLS-protected connections:

```bash
openssl s_client -connect example.com:443
```

A raw `telnet` or `nc` connection cannot automatically perform the TLS handshake.

`openssl s_client` performs that cryptographic handshake and then lets you interact with the encrypted application connection.

---

# 11. TCP client

A client is simpler than a server.

Server:

```text
socket
bind
listen
accept
```

Client:

```text
socket
connect
```

The client normally does not explicitly call:

```text
bind()
listen()
accept()
```

because it is initiating the connection rather than waiting for one.

The OS automatically assigns the client's local IP/ephemeral port when needed.

---

# 12. Hostname resolution

A program often starts with a hostname:

```text
google.com
```

but TCP/IP ultimately needs an IP address.

Older C code might use:

```c
gethostbyname("localhost");
```

This performs name resolution.

Conceptually:

```text
hostname
  ↓
local configuration/cache
  ↓
DNS resolver
  ↓
IP address
```

The system may first check local sources such as `/etc/hosts`.

If necessary, DNS queries are sent to the configured resolver.

The recursive resolver may ultimately consult:

```text
root DNS
   ↓
TLD DNS
   ↓
authoritative DNS
```

DNS results are cached for a period determined by their TTL.

`gethostbyname()` is old and IPv4-oriented. Modern code generally uses:

```c
getaddrinfo()
```

---

# 13. Byte order and `htons()`

Multi-byte integers can be stored differently by different CPUs.

Two common byte orders:

```text
little endian
big endian
```

Network byte order is big-endian.

Example:

```text
2026 decimal = 0x07EA
```

Network representation:

```text
07 EA
```

On a little-endian CPU it may appear in memory as:

```text
EA 07
```

Therefore:

```c
addr.sin_port = htons(2026);
```

converts the host representation into network byte order.

Functions:

```text
htons()  host → network, 16-bit
htonl()  host → network, 32-bit
ntohs()  network → host, 16-bit
ntohl()  network → host, 32-bit
```

Port numbers are 16 bits, hence `htons()`.

---

# 14. OSI layers encountered so far

The code we wrote touches multiple OSI layers.

```text
Layer 7  Application
         HTTP, echo protocol

Layer 4  Transport
         TCP, ports

Layer 3  Network
         IP addresses

Layer 2  Data Link
         Ethernet, Wi-Fi, MAC addresses

Layer 1  Physical
         copper, fibre, radio
```

Most of our explicit code has dealt with Layers 7, 4 and 3.

---

# 15. HTTP is just application data over TCP

For plain HTTP:

```text
HTTP
 ↓
TCP
 ↓
IP
```

A basic HTTP request can simply be text:

```text
GET / HTTP/1.1
Host: google.com

```

HTTP/1.1 uses CRLF line endings:

```text
\r\n
```

and the blank line terminates the header section.

Conceptually:

```text
connect TCP
   ↓
write HTTP request bytes
   ↓
read HTTP response bytes
```

This is why the slide said:

> An HTTP client is `client.c` with a string in it.

The TCP machinery does not understand HTTP. HTTP is just the meaning assigned to the bytes by the application layer.

---

# 16. Handling many clients

Our original server was sequential:

```text
accept A
serve A
close A
accept B
serve B
```

A slow client therefore blocks others.

Two classic solutions:

```text
fork()
```

or:

```text
select / epoll
```

---

# 17. `fork()`

`fork()` creates a new process.

The unusual thing is that execution continues in both processes after the call.

Hence:

> `fork()` returns twice.

Return values:

```text
0     child process
> 0   parent process; value is child's PID
-1    fork failed
```

Example:

```c
pid_t pid = fork();

if (pid == 0) {
    // child
} else if (pid > 0) {
    // parent
}
```

---

# 18. Fork-per-client server

Typical sequence:

```text
parent accept()s client
        ↓
      fork()
      /    \
 parent    child
   ↓         ↓
close fd    serve client
accept      read/write
next        close
client      exit
```

After `fork()`, parent and child both initially possess copies of the connected socket descriptor.

The child keeps its copy and handles the client.

The parent closes its copy and returns to `accept()`.

Thus:

```text
parent → accepts connections
child 1 → client 1
child 2 → client 2
child 3 → client 3
```

Advantage:

- conceptually simple
- process isolation

Disadvantage:

- process-per-client consumes memory and scheduling resources

---

# 19. `select()` vs `epoll`

Both allow one process to manage many sockets.

The general idea:

> Instead of blocking on one socket, ask the kernel which sockets are ready.

## `select()`

On every iteration you provide the complete set of descriptors.

For 1000 connections:

```text
check all 1000
```

even if only 3 have data.

Conceptually:

```text
cost ≈ O(n)
```

It is portable but has scalability limitations.

---

## `epoll`

Linux-specific.

Sockets are registered once.

Afterward the kernel reports the sockets that are ready.

If 1000 sockets exist but only 3 are active:

```text
return those 3
```

Conceptually:

```text
cost relates more closely to O(ready)
```

Similar mechanisms:

```text
Linux       epoll
BSD/macOS   kqueue
Windows     IOCP
```

Mental model:

```text
select → repeatedly ask about everyone
epoll  → register everyone, get events for active ones
```

---

# 20. Client-side connection limits and pooling

Clients also consume resources for each socket.

Sockets count as file descriptors.

Unix command:

```bash
ulimit -n
```

shows the file-descriptor limit.

Browsers historically limited concurrent connections per host. Extra requests would queue.

Opening connections repeatedly is expensive:

```text
TCP handshake
TLS handshake if HTTPS
resource allocation
```

Therefore clients often use connection pools.

Instead of:

```text
open
request
close

open
request
close
```

use:

```text
open
request
request
request
...
close later
```

This reduces connection setup overhead.

---

# 21. `INADDR_ANY`

When a server does:

```c
addr.sin_addr.s_addr = INADDR_ANY;
```

it means:

> Accept connections arriving on any suitable local IPv4 interface.

A machine might have:

```text
127.0.0.1      loopback
Wi-Fi IP
Ethernet IP
```

Binding to a specific IP restricts the server to that address/interface.

---

# 22. Promiscuous mode

Normally a network interface ignores Ethernet frames not meant for it.

Promiscuous mode lets it pass more observed frames to the OS.

This is useful for:

- packet sniffing
- network debugging
- tools such as Wireshark/tcpdump

This is primarily a Layer-2 concept, not a TCP socket concept.

---

# 23. `SO_REUSEADDR`

Common server socket option:

```c
int yes = 1;

setsockopt(
    server_fd,
    SOL_SOCKET,
    SO_REUSEADDR,
    &yes,
    sizeof(yes)
);
```

It helps servers re-bind to the same address/port after a restart when old TCP connection state still exists.

---

# 24. `TIME_WAIT`

A TCP connection may remain temporarily in kernel state even after the application has closed it.

One such state is:

```text
TIME_WAIT
```

It exists partly so delayed packets from an old connection are not mistaken for packets from a newer connection that reuses the same endpoint information.

Important:

> `TIME_WAIT` does not mean the old application process is still running.

It means the kernel remembers the recently closed TCP connection temporarily.

`SO_REUSEADDR` helps servers restart and bind again where appropriate.

---

# 25. SSL vs TLS

SSL was the older protocol family.

Historical sequence:

```text
SSL 2.0
SSL 3.0
   ↓
TLS 1.0
TLS 1.1
TLS 1.2
TLS 1.3
```

Modern systems use TLS.

However, people still commonly say:

```text
SSL certificate
SSL connection
SSL library
```

when the actual protocol is TLS.

So today:

> “SSL” is often legacy terminology for TLS.

---

# 26. Where TLS sits

For HTTPS:

```text
HTTP
 ↓
TLS
 ↓
TCP
 ↓
IP
```

Sequence:

```text
TCP handshake
      ↓
TLS handshake
      ↓
encrypted HTTP request/response
```

The TLS handshake itself sends bytes over TCP.

Only after the TLS session is established does the application send normal HTTP data through the protected TLS channel.

TLS is not specific to HTTP.

Other protocols can also run over TLS.

---

# 27. TLS handshake: conceptual purpose

TLS needs to accomplish several things:

1. Authenticate the server, usually using certificates.
2. Negotiate cryptographic parameters.
3. Establish shared cryptographic keys.
4. Protect subsequent application traffic.

Public-key cryptography is used during authentication/key establishment.

Afterwards, bulk traffic uses symmetric encryption because it is much faster.

---

# 28. Certificates and Certificate Authorities

A certificate effectively binds:

```text
identity ↔ public key
```

For example:

```text
example.com ↔ public key
```

The certificate is signed by a Certificate Authority (CA).

Browsers/operating systems trust a set of CAs.

Therefore:

```text
trusted CA
   ↓ signs
certificate
   ↓ identifies
server/public key
```

Certificates expire and must be renewed.

---

# 29. Certificate pinning

Normally a client accepts a certificate if it passes the trusted CA validation rules.

Pinning makes the requirement stricter.

The application may effectively say:

> I expect this particular certificate/public key.

This reduces dependence on every trusted CA in the global CA ecosystem.

It has historically been used particularly in controlled clients such as mobile apps, though it introduces operational tradeoffs.

---

# 30. Message framing

Because TCP gives a byte stream, protocols need a way to determine:

> Where does one application message end?

Three major techniques:

## Fixed length

Every message is exactly N bytes.

```text
[N bytes][N bytes][N bytes]
```

Advantages:

- simple parsing
- no ambiguity

Disadvantages:

- wasted padding
- inflexible

---

## Delimiter-based

A special sequence marks the end.

Example:

```text
message\n
```

HTTP headers use a blank-line delimiter.

Problem: if the delimiter can occur inside the data, escaping or other rules may be needed.

---

## Length prefix

First transmit the size.

Example:

```text
219 | <219 bytes>
```

Receiver:

```text
read length
read exactly that many bytes
```

This is robust and works well with arbitrary binary data.

---

# 31. Binary Coded Decimal (BCD)

BCD stores each decimal digit independently in 4 bits.

Example:

```text
98765
```

becomes:

```text
9 → 1001
8 → 1000
7 → 0111
6 → 0110
5 → 0101
```

Thus:

```text
1001 1000 0111 0110 0101
```

Because 4 bits correspond directly to one hexadecimal digit, it is easy to inspect.

In BCD:

```text
decimal digits are encoded individually
```

rather than encoding the entire number as one binary integer.

BCD is used in some financial, telecom and legacy systems.

---

# 32. ASN.1

ASN.1 = Abstract Syntax Notation One.

It is primarily a schema/data-description language.

Instead of manually implementing message parsing, you define the structure of your data.

Conceptually:

```text
Person {
    name: string
    age: integer
}
```

Then tooling can generate encoding/decoding logic.

Used in technologies such as:

```text
X.509
LDAP
SNMP
telecom protocols
```

Core idea:

```text
schema
  ↓
generated encoder/decoder
  ↓
wire data
```

---

# 33. Schema vs encoding

This distinction is important.

A schema describes:

```text
what fields exist
their types
their structure
```

Example:

```text
name = string
age = integer
```

An encoding defines:

```text
how those fields become bytes
```

Therefore:

```text
ASN.1 → mainly schema/data model
TLV   → encoding pattern
```

---

# 34. TLV

TLV stands for:

```text
Type
Length
Value
```

A field might look conceptually like:

```text
Type = username
Length = 5
Value = alice
```

A complete message can contain:

```text
[T L V][T L V][T L V]...
```

Why is the length important?

If an old parser sees an unknown type:

```text
unknown type
   ↓
read length
   ↓
skip that many bytes
   ↓
continue parsing
```

This gives good forward compatibility.

ASN.1 encodings such as BER/DER use TLV-like structures.

---

# 35. Hexadecimal

Hex is commonly used to display binary data for humans.

One byte is represented using two hex characters.

Example:

```text
01001111
```

becomes:

```text
4F
```

Hex dumps are useful for debugging protocols because structures such as tags, lengths and repeated fields can often be spotted visually.

---

# 36. Base64

Base64 converts arbitrary binary data into printable ASCII characters.

Common uses:

```text
email attachments
HTTP Basic Auth
JWT components
data URLs
```

Its purpose is usually safe transport through text-oriented channels.

Cost:

```text
3 bytes → 4 Base64 characters
```

so it adds roughly 33% size overhead.

Difference:

```text
hex     → mainly human inspection/debugging
base64  → mainly binary-through-text transport
```

Neither is encryption.

---

# 37. RPC

RPC = Remote Procedure Call.

It is primarily a programming/distributed-system model rather than one single universal wire protocol.

Goal:

> Make invoking code on another machine resemble calling a local function.

Suppose:

```text
add(2,3)
```

actually runs on another server.

The system has to transmit something equivalent to:

```text
function = add
arguments = 2,3
```

Server:

```text
decode request
run add()
encode result
send result
```

---

# 38. Marshalling and demarshalling

Marshalling:

```text
program objects
      ↓
encode
      ↓
bytes
```

Example:

```text
AddRequest(a=2,b=3)
```

becomes binary data.

Demarshalling:

```text
bytes
  ↓
decode
  ↓
program objects
```

This is essentially serialization/deserialization in an RPC context.

---

# 39. Why RPC is harder than a normal function call

A normal local function call has predictable process-local behavior.

Networks create ambiguity.

Example:

```text
client sends "charge card"
        ↓
server charges card
        ↓
response packet is lost
        ↓
client times out
```

The client does not know whether:

```text
A. operation never occurred
or
B. operation succeeded but response was lost
```

If it blindly retries, it might charge the card twice.

This is why retries, idempotency and failure semantics are important in distributed systems.

---

# 40. gRPC

gRPC is a concrete RPC framework/protocol ecosystem.

A useful approximation is:

```text
gRPC
 =
RPC model
+ Protocol Buffers
+ HTTP/2 transport
```

Conceptual stack:

```text
application RPC
      ↓
protobuf serialization
      ↓
gRPC framing
      ↓
HTTP/2
      ↓
TCP
```

---

# 41. Protocol Buffers

Protocol Buffers, or protobuf, provide:

1. a schema language (`.proto`)
2. a compact binary encoding
3. code-generation tools

Example schema:

```proto
message Person {
    string name = 1;
    int32 age = 2;
}
```

The numbers:

```text
1
2
```

are field identifiers/tags.

---

# 42. `.proto` files and ASN.1

A `.proto` file plays a role conceptually similar to an ASN.1 schema.

Both describe:

```text
message structure
field types
field identifiers
```

Example protobuf:

```proto
message AddRequest {
    int32 a = 1;
    int32 b = 2;
}
```

gRPC `.proto` files can additionally define services:

```proto
service Calculator {
    rpc Add(AddRequest) returns (AddResponse);
}
```

This specifies the remote method and request/response types.

---

# 43. Encoder vs TLV

Do not confuse the program with the format.

An encoder is:

> Code that converts structured program data into bytes.

TLV is:

> One possible pattern used to organize those bytes.

So:

```text
encoder = mechanism/program
TLV     = wire-format idea
```

Protobuf has a tag-based binary wire format with several wire types. It is conceptually related to TLV but is not simply a textbook `Type | Length | Value` for every field.

---

# 44. Code generation in protobuf/gRPC

From:

```proto
message AddRequest {
    int32 a = 1;
    int32 b = 2;
}

service Calculator {
    rpc Add(AddRequest) returns (AddResponse);
}
```

the tooling can generate language-specific code.

Generated pieces include things such as:

```text
AddRequest class/struct
AddResponse class/struct

serialization code
deserialization code

client stub
server stub/interface
```

Instead of manually doing:

```text
construct bytes
write socket
read response
parse response
```

application code can look conceptually like:

```text
response = client.Add(request)
```

The generated machinery handles much of:

```text
serialization
message framing
RPC dispatch
HTTP/2 transport
response handling
```

---

# 45. HTTP/2 role in gRPC

HTTP/2 supports multiple streams over a single connection.

So instead of needing one TCP connection per RPC:

```text
TCP connection
 ├── RPC stream 1
 ├── RPC stream 2
 ├── RPC stream 3
 └── RPC stream 4
```

This is called multiplexing.

It improves efficiency when many simultaneous RPCs are occurring.

---

# 46. Big-picture connections

The whole course section so far fits together like this:

```text
APPLICATION
    |
    |  HTTP / RPC / gRPC / echo protocol
    |
MESSAGE STRUCTURE
    |
    |  schema: ASN.1 / .proto
    |  framing: delimiter / fixed / length prefix
    |  encoding: TLV / protobuf / BCD / etc.
    |
OPTIONAL SECURITY
    |
    |  TLS
    |
TRANSPORT
    |
    |  TCP
    |  ports
    |
NETWORK
    |
    |  IP
    |
DATA LINK
    |
    |  Ethernet / Wi-Fi
```

And at the programming level:

```text
server:
socket → bind → listen → accept → read/write

client:
socket → connect → read/write
```

Then concurrency extends the server:

```text
fork
```

or:

```text
select / epoll
```

And higher-level systems such as gRPC hide most of this behind generated APIs.

---

# Questions you should be able to answer in an exam

1. Why does a TCP server require `socket`, `bind`, `listen`, and `accept`?
2. What is the difference between a listening socket and an accepted socket?
3. Where does a TCP client obtain its source port?
4. Why can multiple clients connect to the same server port?
5. Why does TCP require application-level framing?
6. Compare fixed-length, delimiter-based, and length-prefixed framing.
7. What happens when a server does not call `accept()`?
8. What causes `SIGPIPE`?
9. Compare FIN and RST.
10. Why does `htons()` exist?
11. What is network byte order?
12. Why is `gethostbyname()` more complicated internally than it appears?
13. Compare `fork()`, `select()`, and `epoll`.
14. Why are connection pools useful?
15. Explain `INADDR_ANY`.
16. Explain `SO_REUSEADDR` and `TIME_WAIT`.
17. Where does TLS sit relative to HTTP and TCP?
18. What is the difference between SSL and TLS?
19. What role does a certificate authority play?
20. What is BCD?
21. What is ASN.1?
22. What is TLV, and why does it support forward compatibility?
23. Why are hex and Base64 useful for binary protocols?
24. What is RPC?
25. What are marshalling and demarshalling?
26. Why can retrying an RPC be dangerous?
27. What is gRPC?
28. What is a `.proto` file?
29. What does protobuf code generation generate?
30. How do ASN.1, TLV, protobuf, RPC and gRPC relate to one another?

