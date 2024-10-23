**Paper:** [The QUIC Transport Protocol: Design and Internet-Scale Deployment]([The QUIC Transport Protocol:Design and Internet-Scale Deployment](https://dl.acm.org/doi/pdf/10.1145/3098822.3098842))

We have heard a lot about QUIC, and while some aspects of the protocol can be a cause of concern when used on the WAN, certain principles of its design are quite useful, and we also see them repeating through this course.

When the paper was written (i.e. 2017), Google estimated that around 7 percent of internet traffic was QUIC, the primary sources being through Google Chrome and Youtube.

>[!NOTE]
>QUIC isn't just about doing away with TCP and using UDP. It also replaces the majority of TLS, and all things needed for HTTPS, and as such the contribution go a lot deeper.
>This where QUIC becomes a bit more specific than just a Congestion Control Protocol (CCP), it also handles encryption and security.
>We will not focus on that part too much though.

# Introduction

If we want to just start with a single figure, let us start with one of these stack figures:

![[Pasted image 20241022195025.png|500]]

So the main thing that stand out from this is that QUIC essentially does away with TCP (hence reliability and congestion control become a task for it now) and the entirety of TLS.

The main features of QUIC that stand out are:
- **Just enough congestion control to be useable**. That by no means suggests that QUIC plays nice with other protocols (and even with itself!).
- **Reliability and security**, QUIC authenticates endpoints and encrypts packets over UDP transport. It also employs some tricks to prevent this layer from slowing down connections (one of the reasons that Google decided to rip TLS from the protocol)
- **Userspace native**, which means that you can deploy it as long as your kernel knows how UDP works!
- **Efficiency, especially in response to packet loss.** QUIC uses a *stream* abstraction, which multiplexes a single flow into several connections. This is very beneficial, since it prevents head-of-line blocking in case of packet loss, since only the stream that was affected needs to retransmit.

>[!FAQ] QUIC Seems To Have Changed, No?
>The paper was written before the IETF draft of QUIC was done. As such, the QUIC that most of us today is quite different compared to the one deployed at Google around that time.
>In particular:
>- New TLS standard has been proposed since then, and standard QUIC actually adopts that now instead of the cryptographic layer that was developed for it at its inception.
>- The QUIC that Google used, was a monolithic protocol. In fact, Congestion Control wasn't probably an explicit part of it that you could take out and look at. The IETF standard version actually modularizes it so you can do away with things that you want to change.

## Why Was QUIC Needed?

We have heard this a hundred times in this course, but let us do it with another one for sake of safety.

There are a few reasons that QUIC was on Google's mind for a long time:
- **Reducing Delay.** Especially for handshakes over TLS and head-of-line blocking due to packet loss. For example, typical (i.e. old!) TLS would have a total of 3 RTTs for handshake (1 for TCP, 2 for TLS). Of course, this has been addressed to some degree with the new TLS standard, but when we consider that the majority of transactions on the internet are very short, even these small delays become big trouble.
- **Easy Protocol Change.** This is what the paper means about TCP being *ossified* by middle-box systems. TCP is almost always implemented within the kernel, and this means that any significant change to TCP would require an OS upgrade, which is not only slow, but also a bit scary, since it has system-wide impact. This is the main reason that QUIC was always supposed to be a userspace creature.
- **Head-of-Line Blocking.** This again, fundamentally concerned with reducing latency. In the case that we would have to retransmit, all other packets have to wait for a bit. This is because almost all higher level transport protocols (like HTTP and HTTP2) multiplex objects *over a single TCP connection*. QUIC wants to do away with that to resolve this issue.

# QUIC Design

In brief, the main key-points of the design of QUIC are:
- Multiple *streams* for the same flow to prevent head-of-line blocking
- Connection identification without source and destination IP-port pairs to allow for migrating connections from one IP/port to another.
- Combine authentication and connection opening into a single handshake
- **Per-stream** flow control (to make sure a single stream does not exhaust the whole buffer on the client)

## Establishing Connections

![[Pasted image 20241022205309.png|600]]

The handshake in QUIC is a lot leaner and TCP + TLS:
- A client sends a Client Hello (`CHLO`) to a server. The client probably initially has no information about the server configuration (in particular, public keys and signatures), so it explicitly sends and *incomplete* `CHLO` request to the server.
  This is essentially to probe the server for its configuration. The server seeing the incomplete packet, responds by rejecting the packet with a `REJ` packet. This packet contains the security configuration of the server, in particular:
	  - A long-term [[Diffie-Hellman]] public value
	  - A certificate and signature for the server
	  - The **Source-Address Token**, an encrypted block of data (it is encrypted with public key encryption, where the `CHLO` packet contains the public key of the client), which can be used to encapsulate the public IP of the client (as the server sees it of course).
	 This token will be used by the client to prove to the server that it still *owns* that particular IP address for later handshakes.
- Once the client receives all of the configuration above, it will create a short-lived Diffie-Hellman public value that it can use to communicate with the server from now on (this is what we mean by a *completed* `CHLO` packet)

As you see, the above handshake only needs 1 RTT (note that the complete `CHLO` packet can piggy-back an actual client request, so the handshake follows cleanly into the actual communication phase), but the catch is that as long as the long-term Diffie-Hellman public value of the server does not change, the first RTT can be skipped. This means that for any subsequent handshake, **0 extra RTTs are needed**, we just send the completed `CHLO` with our request.

At each stage where a completed `CHLO` is emitted, the sever replies with its own Server Hello (`SHLO`) packet, which is encrypted using the initial keys that the two sides negotiated.
However, with an extra round of Diffie-Hellman calculation on each side (without sending any extra messages), we can generate a set of new keys *for this specific handshake*. The paper calls these the *final*, or *forward-secure* keys. These keys are the ones that are actually used to encrypt packets.

The only exception to the above rule is the packet that we piggyback on top of `CHLO` packets. These may have to use the initial keys on the client to be encrypted, and as such, if we truly want secure, 0-RTT subsequent handshakes, we should also change the initial keys and go through a 1-RTT handshake from time to time.

Eventually, either the keys, the certificate, or the IP assignment will expire or change, and that will cause the server to reject subsequent `CHLO` messages from the client.  In that case, the client will use the content of the `REJ` packet that it gets from the server to update its state and repeat the handshake like before.

>[!NOTE] Version Negotiation
>The server and the client can have multiple QUIC implementations on them. As such, the client has to optimistically choose a version to use for its first packets.
>The server decides if it can handle the version that it sees from the client. If it does, then all is well and version negotiation will not cost anything.
>If that is not the case though, the server must respond with its own supported versions and let the client choose one that it can use, which adds an extra RTT to the handshake.

## QUIC Streams

QUIC multiplexes multiple streams under the same flow. A stream is identified with a *stream identifier* which is an odd number for client initiated streams and even number for server initiated streams.

Each stream treats its data as an array of bytes of length at most $2^{64}$ (so offsets within each stream frame are 64 bit unsigned integers).

A QUIC packet has a header and a series of stream *frames*, which contain data of multiple streams (so a single packet can contain the data for multiple streams if needed).
The header looks like the following:
```
                   +-----------+-------------------------------+
                   | flags (8) | Connection ID (64 - Optional) |
                   +-----------+-------------------------------+
                   |   Version (32 - Optional - Client-Only)   | 
                   +-------------------------------------------+
                   |                NOnce (256)                |
                   +-------------------------------------------+
                   |          Packet Number (8 - 48)           |
                   +-------------------------------------------+
```
- Flags are well, flags, read the protocol to see what the do :)) 
- Connection ID, as mentioned later, can be optionally provided from either side to allow for migrating IP/ports during connection.
- Version numbers are always sent from the client. Version negotiation means that a server should announce its supported versions at most once.
- `NOnce` is the same as in any encrypted packet. A random number generated on the client side that the server or client can use to shuffle keys if needed.
- Packets are numbered, so that drops can be detected easily without following byte boundaries 

Each header is followed by one or more stream frames, each of which have the following format:
```
            +--------------+-----------------------+-------------------+
            |   Type (8)   |   Stream ID (8 - 32)  |  Offset (0 - 64)  |
            +--------------+-----------------------+-------------------+
            | Data Length (0 or 16) |  Stream Data (as much as needed) |
            +-----------------------+----------------------------------+
```
>[!NOTE] Another Use for Connection IDs
>Google also sneakily uses the Connection IDs to route packets within load balancers. Another benefit of designing your own protocols from the ground up I guess :))
>

The paper only says that the version number and the diversification `NOnce` are only available in the first few *early* packets, however early they mean is not explicitly mentioned.
The `NOnce` is only used in the beginning to shuffle keys, NOT for per-packet encryption. For per-packet encryption, the packet number is used as `NOnce` instead (which is why we have packet numbers, not stream numbers, in the header, the should not be encrypted).

QUIC has a reset packet separate from its `REJ` packet, which it uses to tear-down connections or stop connections that fail to decrypt data correctly. To this end, these reset packets are neither encrypted, nor authenticated, which technically means that you can DDoS a cluster of QUIC servers by just shouting reset packets at them with spoofed source IP addresses.
The paper says that the IETF version of the protocol should fix this.

## Loss Recovery

You might have noticed in the previous section that we have both a packet number AND a stream offset, why need both?
In TCP, packets have a single sequence number, which is used both to order packets to retransmit lost ones. This however creates a problem.
If a packet is dropped, the retransmitted packet MUST have the exact same sequence number, so that the server would know where to put it. Once the packet is ACKed, it would be unclear if:

- This is the ACK for the original packet, it was just delayed
- This is the ACK for the retransmitted packet(s), they just were getting dropped

This is a problem if we are trying to track RTTs, which require accurate measurements. To this end