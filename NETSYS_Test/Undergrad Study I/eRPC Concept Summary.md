---
creator: Maike
type: Paper Review
created: 2022-09-05
---
# Goal

Provide similar performance to specialized systems with a general system

## Primary Design Goals

1. Make the common case fast
2. Prevent congestion through restriction

# General Protocol Design

eRPC is exclusively client driven. This means that only the client can actively send messages to the server. The server can only respond. This simplifies design, since only the client needs to be truly “smart”. For example, congestion control is simplified to be only run on client, since the client has full control over the amount of packets the server sends onto the wire.

This also means that some control packets need to be added to assure functionality. For example, if the server needs to send a response to a single-packet client request that is longer than one packet, it cannot do so, since it would require to send at least one packet that is not in direct response to another one.

Therefore there are explicit control packets in eRPC that facilitate the one-to-one mapping of request/response.
![[Untitled(133).png]]


# Structure
![[Untitled(134).png]]


# Dispatch vs Worker Thread:

## Dispatch Thread:

Same thread as event loop, therefore executes tasks without expensive context switching and inter-thread communication, BUT makes the critical path of the event loop longer and therefore making the networking part less reactive.

## Worker Thread

is the exact opposite trade-off.

## Solution:

What is the common case? → Request handlers are usually short. Therefore, use dispatch threads by default to avoid expensive inter-thread communication when not necessary, and only dispatch designated worker threads when handlers are unusually long.

# Sessions:
![[Untitled(135).png]]


## Information regarding Sessions:

1. A session can either take the role of a client or a server.
2. Each session can process a constant amount of requests in parallel (default is 8). Information regarding the requests is stored in an array of slots inside the session.
3. Requests can be processed out of order, preventing head-of-line blocking potentially caused by long requests handlers.
4. Since eRPC is a client-driven protocol, congestion control variables are only maintained in client sessions.
5. Each session is identifiable by a token that is unique inside a cluster.

## The Credit System

A credit system keeps a system from producing more than it is allowed. In this setting, a sender is only allowed to produce a certain amount of outstanding packets. Therefore, each session gets a set amount of credits assigned. A credit is used when sending a packet and a credit is returned when a response has been received.

### Background

1. **RX descriptors -** If there is not enough RX descriptors available in the RX queue, packets will be dropped.
![[Untitled(136).png]]


1. **One-to-One nature of eRPC -** Every request yields a response, therefore requests sent equals to expected responses.

With those two aspects in mind, preventing packet drop due to RX queue exhaustion can be achieved through limiting sent requests on the client side.

### How many credits?

> Allowing BDP/MTU credits per session ensures that each session can achieve line rate.

<aside> ❓ Discussion Time! Why is that? (Because I really don’t know)

</aside>

### How many sessions?

Credits are assigned per session and determined by the BDP. All sessions share one RX queue in their shared RPC endpoint. Hence:

Credits per session * Number of Sessions ≤ RX descriptors

# Zero Copy:

## Background

### Why is Zero Copy important?

As a lot of other paper highlight (recall Understanding Host Network Stack Overheads), memory copy makes up for the biggest portion of CPU usage for any network traffic except for small packets.

### Why is Zero Copy difficult?

Zero copy is implemented by passing pointers to buffers rather than moving the data into a new buffer. This comes with certain challenges:

1. Memory regions might not be universally accessible by all parties due to isolation or different mappings of virtual to physical memory.
2. By passing pointers to buffers instead of moving data to new buffers, the original buffer cannot be reused immediately which can cause conflict with pre-existing mechanisms or slow down progress.
3. Sometimes, data is just not in a consecutive place in memory and needs to be properly arranged through copy.

## Zero Copy Transmission

Data produced by an application can be bigger than what can be transported in a single packet. In this case, the data needs to be broken up into multiple packets, each with their own header.

Often, packet data is stored in buffers at “packet granularity”
![[Untitled(137).png]]


which most accurately presents the way the packets will be transmitted. Like this, packets can be easily moved to the NIC with a single DMA.
![[Untitled(138).png]]


However, since the data produced by the application exists as contiguous data, splitting it up into data chunks requires data copy.

To avoid this, in eRPC the buffer layout is altered to feature a contiguous data section that can be used by the application to store the data in that is to be transmitted.
![[Untitled(139).png]]


This complicates DMA from host to NIC, since the raw packet data is not in order of transmission.

<aside> ❓ Discussion Time! The paper says: “Non-first packets require two DMAs […]; this is reasonable because the overhead for DMA-reading small headers is amortized over the large data DMA.”. How can the data be DMAd as one? Should it not be associated with its respective headers and therefore be broken up into multiple small DMAs?

</aside>

But, we remember the design policy: Make the common case fast.

Most RPC transmissions consist of one packet, which still only requires one DMA.

## Zero Copy Reception

The issue with reception is that data arrives and is DMAd to the host in the order that packets arrive (same as depicted earlier)
![[Untitled(140).png]]


The application, however, needs the data without the headers, which unfortunately requires _copying_ the data into a contiguous buffer.

But again, since the goal is as always to make the common case fast, the single packet receptions can be processed using zero copy, since the data-section for a single packet is contiguous and can be directly passed to the application.

The only requirement for this is that the RX buffer that the packet arrived in cannot be reused until the application finishes.

Congestion Control: