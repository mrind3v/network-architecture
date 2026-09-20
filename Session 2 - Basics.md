## 1. The core idea: layering and encapsulation

Modern networks are built as layers. Each layer solves a particular class of problem and tries not to depend on the internal details of the other layers.

The most important idea is encapsulation:

`Application data`
→ `[TCP header | application data]`
→ `[IP header | TCP header | application data]`
→ `[Ethernet header | IP header | TCP header | application data]`

Each lower layer treats the complete output of the layer above as its payload.

This is why the same pattern appears in Ethernet, TCP/IP, SS7, email, FTP, etc.

A useful phrase:

> One layer’s packet is another layer’s payload.

---

# 2. OSI model

The OSI model divides networking into seven conceptual layers.

| Layer | Name | Main job | Examples |
|---|---|---|---|
| L7 | Application | Application-level protocols | HTTP, FTP, DNS, SMTP |
| L6 | Presentation | Encoding, compression, encryption | TLS-like functions, serialization |
| L5 | Session | Session management | Dialogues, authentication/session state |
| L4 | Transport | Process-to-process transport | TCP, UDP |
| L3 | Network | Routing packets between networks | IP, ICMP |
| L2 | Data Link | Local-link delivery | Ethernet, MAC, ARP, switches |
| L1 | Physical | Move raw bits/signals | Copper, fiber, radio |

The practical Internet stack does not map perfectly to OSI, especially L5/L6, but OSI is still useful conceptually.

The point of layering is abstraction. HTTP does not need to know whether the underlying transmission is over Ethernet, Wi-Fi, fiber, etc.

---

# 3. Ethernet vs Wi-Fi: collision detection vs collision avoidance

Old shared Ethernet used CSMA/CD:

Carrier Sense Multiple Access with Collision Detection.

A wired Ethernet device could transmit and detect a collision while transmitting.

Wi-Fi uses CSMA/CA:

Carrier Sense Multiple Access with Collision Avoidance.

A Wi-Fi radio generally cannot reliably listen for another transmission while it is itself transmitting because its own signal dominates.

Typical Wi-Fi logic:

1. Listen to the channel.
2. If busy, wait.
3. If free, wait a random backoff interval.
4. Transmit.
5. Wait for an ACK.
6. If ACK does not arrive, assume possible loss/collision and retransmit.

Important consequence:

`Ethernet → detect collisions`
`Wi-Fi → avoid collisions`

Wi-Fi retransmissions happen at L2, below TCP. TCP may not even know that Wi-Fi locally retransmitted a frame several times.

---

# 4. SS7: the big architectural idea

SS7 = Signalling System No. 7.

Its major historical idea was to separate signalling from the voice path.

Before SS7, telephone signalling was often in-band. The same channel carrying voice also carried control tones.

Examples included:
- 2600 Hz signalling tones
- dialled digits encoded as tones

This was dangerous because control information was effectively just sound.

SS7 moved signalling out-of-band.

So:

`Voice bearer → actual conversation`

`SS7 signalling network → call setup, routing, teardown, subscriber queries`

This is a classic control-plane/data-plane separation.

---

# 5. SS7 stack vs TCP/IP stack

SS7 is not one protocol comparable to TCP.

SS7 is a complete protocol stack.

Rough correspondence:

| Approx. layer | SS7 | TCP/IP world |
|---|---|---|
| L7 | MAP, INAP, CAP, ISUP | HTTP, DNS, SMTP, SIP |
| L5/L6 | TCAP | TLS / application framing |
| L4-ish | SCCP | TCP / UDP |
| L3 | MTP3 | IP |
| L2 | MTP2 | Ethernet / PPP / HDLC |
| L1 | MTP1 | copper / fiber / radio |

The fair comparison is:

`SS7 stack ↔ TCP/IP stack`

not:

`SS7 ↔ TCP`

---

# 6. SS7 encapsulation

SS7 also uses nested encapsulation.

For an SMS:

`SMS text`
→ SMS TPDU
→ MAP
→ TCAP
→ SCCP
→ MTP3
→ MTP2

Each layer adds metadata.

The actual user payload may be very small, while the full signalling message is much larger.

For example, `"Hello World!"` may occupy only around 11 bytes in the relevant encoding, but many layers surround it.

General principle:

`small application payload + many headers = much larger transmitted unit`

---

# 7. Why classic SMS is 140 bytes

Classic SMS payload = 140 octets.

The reason comes from stacked protocol size limits.

Roughly:

- MTP2 signalling information field has a hard size limit around 272 octets.
- MAP has its own user-data limits.
- After all the protocol overhead, about 140 bytes are left for the SMS payload.

The 140-byte choice was made so one SMS could fit into one signalling packet without fragmentation.

Important:

`140 bytes ≠ 140 characters`

Using GSM 7-bit encoding:

`140 bytes = 1120 bits`

`1120 / 7 = 160 characters`

That is where the famous 160-character SMS limit comes from.

Key idea:

`272-ish signalling limit → protocol overhead → 140-byte SMS payload → up to 160 GSM-7 chars`

---

# 8. How SMS delivery works

SMS is basically:

`store-and-forward + database lookup`

Important entities:

- Phone A: sender
- VMSC/VLR: sender-side mobile switch/database
- SMSC: Short Message Service Center
- HLR: Home Location Register
- MSC B: serving switch for receiver
- Phone B: receiver

Simplified flow:

`Phone A`
→ serving MSC
→ SMSC stores the message
→ SMSC asks HLR where Phone B currently is
→ HLR returns serving MSC info
→ SMSC sends SMS toward MSC B
→ MSC B delivers to Phone B

If Phone B is unavailable, the SMSC can retain the SMS and retry later.

The HLR does not carry the SMS payload. It tells the network where the subscriber currently is.

Important distinction:

`HLR = location/subscriber lookup`

`SMSC = message storage and forwarding`

---

# 9. Does SS7 carry actual user data?

Yes, sometimes.

This is important because saying “SS7 only carries control” is too broad.

For voice calls:
- SS7/ISUP carries signalling
- actual voice travels on a separate bearer circuit

For SMS:
- the SMS payload itself is embedded in signalling messages, typically through MAP

So:

| Service | What SS7 carries |
|---|---|
| Voice call | control/signalling only |
| SMS | control information + SMS payload |

That is why SMS size is constrained by SS7 signalling message limits.

---

# 10. SS7 ISUP phone-call setup

ISUP is used for call setup and teardown between telephone switches.

Typical sequence:

`IAM → ACM → ANM → conversation → REL → RLC`

Meaning:

`IAM` = Initial Address Message  
“Set up a call to this number.”

`ACM` = Address Complete Message  
“Destination has been reached / ringing.”

`ANM` = Answer Message  
“Called party answered.”

Then the voice bearer carries the actual conversation.

When the call ends:

`REL` = Release  
“End the call.”

`RLC` = Release Complete  
“Resources/circuit released.”

Important:

`ISUP controls the call`

`Bearer circuit carries the voice`

During the conversation, SS7 signalling may be mostly idle.

---

# 11. SIP and modern IP calling

Modern IP-based telephony uses SIP for signalling.

SIP = Session Initiation Protocol.

Typical call flow:

`INVITE`
→ `100 Trying`
→ `180 Ringing`
→ `200 OK`
→ `ACK`

Then media flows.

When hanging up:

`BYE`
→ `200 OK`

SIP is text-based and resembles HTTP in style.

It uses status codes such as:

`100`
`180`
`200`

Actual audio/video normally uses RTP/SRTP, commonly over UDP.

So:

`SIP = signalling/control`

`RTP/SRTP = media`

This is conceptually similar to:

`SS7/ISUP = signalling`

`voice bearer = media`

The architecture changed, but control and media are still logically separated.

---

# 12. SS7 reliability vs TCP reliability

This is a major design difference.

Imagine:

`A → R1 → R2 → B`

SS7 traditionally provides reliability hop-by-hop.

Each adjacent signalling link is made reliable:

`A ↔ R1`
`R1 ↔ R2`
`R2 ↔ B`

If a frame is lost on one link, that local link handles recovery.

TCP provides end-to-end reliability:

`A ---------------- B`

Routers in the middle do not implement TCP reliability.

If a TCP segment is lost, endpoint A retransmits it because endpoint B did not acknowledge it.

Therefore:

`SS7 reliability scope = per hop`

`TCP reliability scope = end-to-end`

This reflects two different architecture philosophies.

---

# 13. TCP three-way handshake

TCP connection opening:

`SYN →`
`← SYN-ACK`
`ACK →`

Why three messages?

Because TCP is bidirectional, and each side has its own sequence-number space.

Suppose:

Client ISN = 1000  
Server ISN = 5000

Client sends:

`SYN seq=1000`

Server replies:

`SYN-ACK seq=5000 ack=1001`

Client replies:

`ACK ack=5001`

Why `1001` and `5001`?

Because SYN consumes one sequence number even though it carries no application payload.

The purpose of the three-way handshake is to make sure both sides:

- announce their Initial Sequence Number
- acknowledge the other side's Initial Sequence Number

After that, both sides are synchronized.

---

# 14. TCP options in the handshake

TCP may negotiate options during SYN exchange.

Examples:

`MSS`
Maximum Segment Size.

`SACK permitted`
Selective acknowledgements are supported.

`Window scale`
Allows larger receive windows.

These are normally exchanged during connection setup.

---

# 15. Why initial sequence numbers must be unpredictable

Older TCP stacks sometimes used predictable sequence numbers.

If an attacker could guess the next sequence number, they might forge TCP segments that appeared to belong to a legitimate connection.

Modern implementations generate ISNs in ways intended to be difficult to predict.

---

# 16. SYN flood attack

Normal server behavior:

Client sends `SYN`.

Server:
- allocates a half-open connection entry
- stores it in the SYN backlog
- sends `SYN-ACK`
- waits for final `ACK`

Attack:

The attacker sends huge numbers of SYNs but never completes the handshake.

Result:

`SYN #1 → backlog entry`
`SYN #2 → backlog entry`
`SYN #3 → backlog entry`
`...`

Eventually, the finite SYN backlog fills.

Legitimate clients may then be unable to create new connections.

That is SYN flooding.

---

# 17. SYN cookies

SYN cookies defend against SYN floods by avoiding state allocation before the client proves it received the server’s SYN-ACK.

Normal TCP:

`receive SYN`
→ store connection state
→ send SYN-ACK
→ wait for ACK

With SYN cookies:

`receive SYN`
→ compute cookie
→ encode cookie into server’s sequence number
→ send SYN-ACK
→ store nothing

Conceptually:

`cookie = secret_hash(client IP, client port, server IP, server port, time, ...)`

The legitimate client sends the final ACK.

Since TCP ACKs acknowledge:

`server sequence number + 1`

the cookie effectively comes back.

The server recomputes the expected cookie.

If valid:

- then it creates the real TCP connection state.

So the key idea is:

Without SYN cookies:
`server remembers client`

With SYN cookies:
`server makes client carry the information that the server would otherwise remember`

This is a stateless-defense technique.

It prevents fake SYNs from consuming one backlog entry each.

---

# 18. TCP connection close

TCP teardown usually uses four segments:

`FIN →`
`← ACK`
`← FIN`
`ACK →`

Why four instead of three?

Because TCP is full-duplex.

There are two independent directions:

`Client → Server`

`Server → Client`

A FIN closes only one sending direction.

Suppose the client sends FIN.

That means:

“I will not send any more data.”

The server may still continue sending data.

So after the client FIN is acknowledged:

`client→server = closed`

`server→client = still open`

Later, the server sends its own FIN.

That closes the other direction.

---

# 19. TCP half-close and CLOSE_WAIT

A half-close is a real TCP state, not an error.

After one side has sent FIN and the other side has ACKed it, one direction is closed while the other may remain open.

`CLOSE_WAIT` means:

“The peer has closed its sending side, but my local application has not closed yet.”

If many sockets are stuck in CLOSE_WAIT, the problem is often application code that forgot to close sockets.

---

# 20. TIME_WAIT

After the side that performs the active close sends the final ACK, it often enters `TIME_WAIT`.

Purpose:

- allow old delayed TCP segments from the previous connection to expire
- allow retransmission of the final ACK if needed
- prevent an old segment from being mistaken as part of a new connection using the same endpoints

TIME_WAIT lasts roughly `2 × MSL`.

MSL = Maximum Segment Lifetime.

This is why restarting a server can sometimes produce:

`address already in use`

This also motivates options such as `SO_REUSEADDR`.

---

# 21. Round-trip time and protocol startup cost

RTT = Round-Trip Time.

It is the time for a message to travel from client to server and for a response to come back.

Plain TCP typically requires about:

`1 RTT`

for connection setup before normal application data is exchanged.

TCP + TLS 1.2 historically could require roughly:

`1 RTT TCP + ~2 RTT TLS`

TCP + TLS 1.3 reduces cryptographic setup to roughly:

`1 RTT TCP + 1 RTT TLS`

QUIC combines transport and cryptographic setup more tightly.

The main lesson:

`Every required round trip before application data = startup latency`

Bandwidth can be increased by faster links, but long-distance RTT is constrained by propagation delay and physics.

---

# 22. TCP Fast Open

TCP Fast Open allows a returning client, under suitable conditions, to include application data in the SYN itself.

Instead of:

`handshake → then data`

it can partially do:

`SYN + data`

This can reduce one startup delay.

---

# 23. UDP

UDP provides a very small transport-layer abstraction.

UDP header:

- Source Port: 2 bytes
- Destination Port: 2 bytes
- Length: 2 bytes
- Checksum: 2 bytes

Total:

`8 bytes`

TCP has a minimum 20-byte header and many more fields.

UDP does not provide:

- connection handshake
- ordering
- acknowledgements
- retransmissions
- TCP-style flow control
- TCP-style congestion control

Therefore UDP is fast and simple, but applications must implement missing features themselves if needed.

---

# 24. Why use UDP?

UDP is useful when waiting for retransmissions would be worse than losing data.

Example: real-time audio.

If audio packet #100 is lost and packet #110 is already being played, retransmitting #100 later may be pointless.

Better to drop it and continue.

Typical UDP users:

- DNS
- real-time audio/video
- gaming
- WebRTC
- streaming
- QUIC underneath HTTP/3

---

# 25. QUIC

QUIC runs over UDP but reconstructs many transport features in userspace.

Conceptually:

`HTTP/3`
over
`QUIC`
over
`UDP`
over
`IP`

QUIC implements:

- reliability
- acknowledgements
- retransmissions
- congestion control
- stream management
- connection management
- TLS 1.3 integration

That is why people loosely say:

`QUIC rebuilds TCP on top of UDP`

But QUIC is not merely TCP copied into UDP. It improves several aspects.

---

# 26. HTTP/2 head-of-line blocking

HTTP/2 multiplexes many streams over one TCP connection.

But TCP exposes one ordered byte stream.

If one TCP segment is lost, TCP may need to recover that segment before delivering later bytes.

Therefore one packet loss can stall multiple HTTP/2 streams.

This is transport-level head-of-line blocking.

---

# 27. QUIC independent streams

QUIC gives different streams more independent ordering behavior.

If data for stream A is lost:

`stream A` may wait

while:

`stream B`
and
`stream C`

may continue.

This is one major advantage of HTTP/3 over QUIC.

---

# 28. Why QUIC being in userspace matters

TCP is usually implemented in the OS kernel.

Changing TCP significantly may require:

- operating-system updates
- kernel changes
- compatibility with middleboxes

QUIC is typically implemented in application libraries or browsers.

Therefore protocol changes can be deployed faster.

A browser vendor can ship a QUIC update without waiting for every operating system to replace its kernel TCP implementation.

---

# 29. SMTP

SMTP = Simple Mail Transfer Protocol.

It is used to send/relay email.

Important ports:

`25` → server-to-server SMTP relay

`587` → email client submission, commonly with STARTTLS

`465` → SMTP over implicit TLS

SMTP is human-readable text.

A mail transfer consists of text commands and responses.

SMTP was designed for simplicity, not security.

---

# 30. Why SMTP remained text-based

Mail servers may need to:

- inspect headers
- add headers
- rewrite information
- relay messages through many systems

Text protocols are easy to extend.

New commands and headers can often be added while maintaining compatibility.

This helped email evolve over decades.

---

# 31. SMTP security limitations

Base SMTP historically did not prove that the sender truly owned the address appearing in commands such as:

`MAIL FROM:<alice@example.com>`

Later security mechanisms were added.

STARTTLS:
encrypts SMTP transport.

SPF:
states which servers are authorized to send email for a domain.

DKIM:
cryptographically signs email using a domain’s key.

DMARC:
defines policy for interpreting SPF/DKIM and handling failures.

Important:

SMTP itself was not originally designed with strong sender authentication.

---

# 32. MIME

MIME = Multipurpose Internet Mail Extensions.

MIME allows email to carry:

- plain text
- HTML
- images
- PDFs
- attachments
- multiple message parts

MIME labels each section using headers such as:

`Content-Type:`

and:

`Content-Transfer-Encoding:`

---

# 33. Why images become text in email

Classic mail systems were designed around text and 7-bit-safe transport.

Arbitrary binary bytes could not always safely travel through old mail infrastructure.

Therefore binary attachments are often encoded with Base64.

---

# 34. Base64

Base64 transforms binary data into safe printable text.

Three input bytes:

`3 bytes = 24 bits`

Split into:

`4 groups × 6 bits`

Each 6-bit value has:

`2^6 = 64`

possible values.

These map to 64 printable characters:

`A-Z`
`a-z`
`0-9`
`+`
`/`

Padding may use:

`=`

Therefore:

`3 input bytes → 4 output characters`

Size expansion:

`4 / 3 ≈ 1.33`

So Base64 adds roughly 33% overhead.

Example:

`3 MB binary`
→ roughly
`4 MB Base64`

before other message overhead.

---

# 35. MIME multipart boundaries

MIME can separate multiple parts using a boundary.

Conceptually:

`boundary`
→ text part

`boundary`
→ image part

`boundary`
→ PDF part

`boundary end`

A MIME section may include:

`Content-Type: image/png`

`Content-Transfer-Encoding: base64`

Then the Base64 text follows.

---

# 36. POP3

POP3 = Post Office Protocol version 3.

Purpose:

retrieve email from a mail server.

Main contrast:

`SMTP = send/transfer mail`

`POP3 = retrieve mail`

In normal use, the receiver’s email application runs POP3 commands automatically.

The human does not normally type them.

---

# 37. POP3 client and server

Suppose Bob owns:

`bob@example.com`

Mail arrives at Bob’s provider.

The provider stores Bob’s mailbox on a mail server.

Later:

`Bob’s email app`
→ POP3
→ Bob’s mail server

The client retrieves messages.

So yes, the receiver is the one connecting to the POP3 server, typically via their email application.

---

# 38. POP3 commands

Typical POP3 session:

`USER bob`

`PASS secret`

Then:

`STAT`
asks:
“How many messages and how many total bytes?”

`LIST`
lists message numbers and sizes.

`RETR 1`
retrieves message 1 in full.

`DELE 1`
marks message 1 for deletion.

`QUIT`
ends the session and commonly causes marked deletions to be committed.

---

# 39. POP3 security

Basic POP3 can send username/password in plaintext.

Encrypted POP3 commonly uses POP3S, traditionally on port:

`995`

Without encryption, anyone who can inspect traffic might see credentials.

---

# 40. POP3 mental model

Traditional POP3 model:

`mail lives primarily on your device`

The server acts more like a spool to be emptied.

This causes problems with multiple devices.

If one device downloads/deletes a message, another device may not see it.

---

# 41. IMAP

IMAP = Internet Message Access Protocol.

The key difference from POP3 is where mailbox state lives.

POP3:

`master state mainly on client`

IMAP:

`master mailbox state on server`

Clients keep synchronized copies/caches.

This design is much better for multiple devices.

---

# 42. POP3 vs IMAP

POP3:
- mail traditionally downloaded to one device
- read/unread state often local
- simple inbox model
- poor multi-device synchronization
- often retrieves whole messages
- searches may require local download

IMAP:
- mail remains on server
- read/unread state stored server-side
- supports folders/mailboxes
- multiple devices see same state
- server-side search
- partial fetching
- synchronized flags such as `\Seen`, `\Answered`, `\Flagged`

Mental model:

`POP3 = download`

`IMAP = synchronize`

---

# 43. Framing

Framing answers:

“How does the receiver know where one application message ends and the next begins?”

This is especially important over TCP because TCP is a byte stream, not a message protocol.

TCP may give the application bytes such as:

`abcdef123456xyz`

The application protocol must determine message boundaries.

---

# 44. Delimiter-based framing

Choose a special byte sequence to mark the end of a message.

Examples:

SMTP:
special terminating line.

HTTP headers:
`CRLF CRLF`

MIME:
boundary markers.

Conceptually:

`read until delimiter`

Advantage:
the sender does not need to know the total size in advance.

Problem:
what if the delimiter appears inside the content?

Then escaping is needed.

SMTP uses dot-stuffing for this type of problem.

---

# 45. Length-prefixed framing

The sender first says how much data is coming.

Example:

`Content-Length: 4096`

Then receiver reads exactly:

`4096 bytes`

Conceptually:

`read length → read exactly N bytes`

Advantage:
payload bytes cannot accidentally be mistaken for protocol delimiters.

Disadvantage:
the sender usually needs to know the size first.

---

# 46. Using both delimiter and length

Many protocols combine both approaches.

HTTP/1.1 often uses:

`CRLF-delimited headers`

plus:

`Content-Length: N`

for the body.

So:

`headers → delimiter framed`

`body → length framed`

This is common and practical.

---

# 47. HTTP chunked transfer idea

Sometimes the sender does not know the final body size in advance.

Instead of one Content-Length for the whole body, the data can be sent as multiple chunks.

Each chunk has its own length.

This allows streaming without first knowing the total size.

---

# 48. FTP

FTP = File Transfer Protocol.

Its defining architecture is that it separates commands and file data into different TCP connections.

There is:

`control connection`

and:

`data connection`

The control connection is usually long-lived.

The data connection is created when files or directory listings need to be transferred.

---

# 49. FTP control connection

Typically:

`Client → Server port 21`

Commands travel here.

Examples include:

`USER`

`PASS`

`LIST`

`RETR`

and similar FTP commands.

Actual file bytes do not normally travel over this control connection.

---

# 50. FTP active mode

Historical FTP active mode:

Client opens control connection:

`Client → Server`

But for data transfer, the server opens a new connection back toward the client.

So:

`Client → Server: control`

`Server → Client: data`

Historically the server often used port 20 for data.

This is awkward with NAT and firewalls because they dislike unexpected inbound connections toward clients.

---

# 51. FTP passive mode

Passive mode reverses the data-connection initiation.

The client sends:

`PASV`

The server replies with an IP/port where it is listening.

Then:

`Client → Server: control`

and:

`Client → Server: data`

Both connections originate from the client side.

That works much better through NAT/firewalls.

Therefore passive FTP is much more common in modern environments.

---

# 52. FTP passive port range

A real FTP server often restricts passive-mode data connections to a configured range, for example:

`30000–30009`

Why?

Because firewall rules need to know which inbound ports to allow.

If passive FTP could use any random port from 1–65535, firewall configuration would be much harder.

So servers often define a small finite passive range.

---

# 53. FTP demo setup

The FTP demo created:

`ftp-demo/hello.txt`

with sample content.

A disposable FTP server was started inside a container.

Important configuration ideas:

`2121`
used as demo control port instead of 21.

`30000–30009`
used as passive-mode data ports.

A local directory was mounted into the container as:

`/srv`

The FTP server therefore sees files in that mounted directory.

Username/password were configured.

Write access was enabled.

A NAT-advertised address was configured because software inside a container may not know the address that clients can actually reach.

---

# 54. The recurring architectural themes

Several ideas repeat across all of these protocols.

Layering:
each protocol solves one level of the problem.

Encapsulation:
each lower layer wraps the upper layer.

Control plane vs data plane:
control messages may travel separately from user data.

Examples:

`SS7 ISUP vs voice bearer`

`SIP vs RTP`

`FTP control vs FTP data connection`

End-to-end vs hop-by-hop reliability:

`TCP → end-to-end`

`SS7 → historically hop-by-hop`

Framing:

`delimiter`
vs
`length`

Statelessness:

SYN cookies avoid holding state before trust/proof is established.

Latency:

Reducing round trips is often more valuable than merely increasing bandwidth.

---

# 55. High-value contrasts to memorize

These are especially useful for exams:

| Topic | A | B |
|---|---|---|
| Ethernet vs Wi-Fi | collision detection | collision avoidance |
| TCP vs UDP | reliable ordered stream | minimal datagram service |
| TCP vs SS7 reliability | end-to-end | hop-by-hop |
| SS7 ISUP vs bearer | signalling | voice |
| SIP vs RTP | signalling | media |
| POP3 vs IMAP | download-oriented | sync/server-oriented |
| FTP active vs passive | server opens data connection | client opens data connection |
| Delimiter vs length | read until marker | read exact number of bytes |
| TCP vs QUIC | one ordered stream | multiple independent streams |
| TCP handshake vs teardown | 3 segments | usually 4 segments |
| SMTP vs POP3 | send/relay | retrieve |
| POP3 vs SMTP | receiver side | sender/relay side |

---

# 56. Exam-level “why” questions you should be able to answer

You should be able to explain, not just memorize:

Why does Wi-Fi use collision avoidance instead of collision detection?

Because a radio cannot reliably listen for weak incoming signals while transmitting its own much stronger signal.

Why is SMS limited to 140 bytes?

Because protocol overhead and SS7 signalling-unit limits constrain the payload so one SMS fits into one signalling packet.

Why does an SMS contain actual user data inside SS7 while voice does not?

SMS was designed to embed the message inside signalling operations. Voice uses a separate bearer circuit.

Why does TCP use a three-way handshake?

Because each side must announce and confirm its own initial sequence number.

Why does TCP teardown normally require four messages?

Because each direction of the full-duplex connection closes independently.

Why do SYN cookies work?

Because the server delays storing connection state until the client proves it received the SYN-ACK.

Why does UDP have lower overhead?

Because it removes connection setup, ordering, acknowledgements, retransmissions, flow control, and most transport state.

Why does QUIC use UDP?

UDP provides a minimal transport substrate that QUIC can build on in userspace without being constrained by kernel TCP behavior and ossified middleboxes.

Why can HTTP/2 still suffer head-of-line blocking?

Because all streams still pass through one ordered TCP byte stream.

Why does HTTP/3/QUIC improve this?

Because QUIC provides multiple independently ordered streams.

Why is Base64 about 33% larger?

Because 3 bytes become 4 text characters.

Why does IMAP work better across multiple devices?

Because mailbox state lives on the server and devices synchronize against it.

Why does passive FTP work better through NAT?

Because the client initiates both the control and data connections.

Why does application framing exist?

Because TCP gives a byte stream and does not preserve application message boundaries.

---

# 57. Final mental model

The entire set of slides can be reduced to a few recurring principles:

`Networks are layered.`

`Layers encapsulate other layers.`

`Control and data are often separated.`

`Reliability can be hop-by-hop or end-to-end.`

`Every guarantee costs state, bytes, computation, or latency.`

`UDP removes guarantees; QUIC selectively rebuilds them.`

`Text protocols need framing and encoding.`

`Old protocol constraints often survive for decades.`

`Protocol design is largely about deciding where state, reliability, framing, security, and control should live.`

If you understand those principles, the individual protocols become much easier to reason about.