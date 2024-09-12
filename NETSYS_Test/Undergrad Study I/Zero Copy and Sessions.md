---
creator: Maike
type: Hands-On
created: 2022-09-19
---
Every request needs to reserve space for the response:
![[Untitled(190).png]]


Source: eRPC/src/session.h

Every SSlot has preallocated response buffers (this has something to do with zero-copy on receiver)
![[Untitled(191).png]]


Source: eRPC/src/sslot.h

Also maintains pointer to buffer in TX.
![[Untitled(192).png]]


Source: eRPC/src/sslot.h

Then, every SSlot gets server or client specific variables. Both of them feature another message buffer.

Client:
![[Untitled(193).png]]


Server:
![[Untitled(194).png]]


Source: eRPC/src/sslot.h

How are these buffers used?

This is an excerpt from enqueue_request(). A free SSlot is found in the session (line 35) and it is checked that that slot does not still have a request pending(line 37). (If the rx_msgbuf is still pointing to a buffer, it means that the slot is still waiting for a response.) Then, the new request buffer (from the function arguments) is linked to the tx_msgbuf, indicating that the request is now active (line 38). Note that we are only passing a pointer to the tx_msgbuf, meaning that this is our zerocopy. Furthermore, the preallocated response buffer (from the function arguments) is assigned to the slot’s response buffer (line 42). This, too, is just a passing of pointers.
![[Untitled(195).png]]

![[Untitled(196).png]]


Source: eRPC/src/rpc_impl/rpc_req.cc

As for header management:

Recall the structure of the msgbuf

The data fields have been filled by the application. The packet headers have to be filled now since rpc is the transport protocol and is in charge of setting all packet header info. Lines 53 through 59 set header 0’s values. Then, in the rare case that more than one packet is sent, lines 62 through 68 find the position for the next header (which has to be placed after the data field), **copy** the header 0’s values and only change the pkt_num. Interestingly, the packet header does not indicate the size of its own data chunk, but rather the size of the message that it is part of. This saves time in header manipulation, since only the packet number needs to be changed.
![[Untitled(197).png]]


Source: eRPC/src/rpc_impl/rpc_req.cc

At this point, I asked myself if this scheme is really zero-copy, since we might not move the data via copy, but we perform copying of the packet header multiple times. In the paper, they say

> Non-first packets require two DMAs (header and data); this is reasonable because the overhead for DMA-reading small headers is amortized over the large data DMA.

, but the code snippet from above does not depict DMA to the best of my knowledge. Therefore, I feel like they did not mention this copy in their paper.

<aside> ❓ Discussion time: Is this data copy negligible?? aka can it still be considered “zero-copy” + Also: Would it not be faster to just manually set 5 variables rather than invoking memcpy? Or is it the dereferencing that would slow it down too much?

</aside>

So what actually happens in transmission?

First: DPDK Variables. Essential for this code snippet is the rte_mbuf structure (variable name tx_mbufs). Rte_mbuf can be a chain of multiple segments. In that case, the first segment indicates the number of total segments in the variable nb_segs. If it is 1, it means that there is only one segment, otherwise there are multiple and they are chained together using the variable next. Each rte_mbuf indicates two lengths: data_len and pkt_len. Data_len indicates the length of all segments together, whereas pkt_len holds the length of this specific segment. If nb_segs == 1, those variables will have the same value.

Looking at the code below, the if statement in line 45 checks whether it is dealing with a first packet or not. If it is a first packet, nb_segs is set to 1 and pkt_len and data_len are both equal to pkt_size, which indicates the RPC message size. Then, the data is **copied into the rte_mbuf**.

In the case that it is not a first packet, nb_segs is set to 2 and the first segment’s data_len (which is the single segments size) is set to the size of a packet header, since that is the data in the first segment. Then, the packet header is **copied** into the mbuf. After that, the next mbuf is allocated and only the data_len is set, since the rest is already indicated in the first segment. Then, ****the data, too, is copied into the new mbuf. **So, essentially, all data is copied into DPDK data structures. Well done.**
![[Untitled(198).png]]


Source: eRPC/src/transport_impl/dpdk/dpdk_transport_datapath.cc

But, we already established that their DPDK implementation does not fully support zero copy.

Lets look at infiniband:

The relevant datastructure here is ibv_sge, which is defined as such:


![[Untitled(199).png]]
[Source](https://www.ibm.com/docs/en/aix/7.2?topic=management-ibv-post-recv)

where addr and length are pretty self-explanatory. As for lkey, I don’t know and it is not exactly critical to know for this example, I think.

Below is the same code excerpt as for DPDK, but for Infiniband. The big difference is that apparently Infiniband supports to receive pointers to the data regions(line 31, 42 and 47) it is supposed to DMA, rather than having a designated buffer in its data structure for the data as was the case in DPDK, hence making zero-copy a lot easier.
![[Untitled(200).png]]


Source: eRPC/src/transport_impl/infiniband/ib_transport_datapath.cc

I still struggle to see “the large data DMA”, though. Here, it seems as though data is DMA’d at packet granularity, rather than in a big chunk. Unless there is some sort of underlying optimization that detects adjacent buffers and moves them as one, I do not see how this is supposed to be a large data DMA.

So much for transmission, how about reception?

The right sslot is found by getting the header’s request number and performing the remainder operation with the maximum number of slots (here: 8).

- How does that assignment work?
    
    When SSlots are first allocated, their initial request number is set to their sslot_id (line 89)
    ![[Untitled(204).png]]
    
    Source: eRPC/src/session.h
    
    Then, when a new request is enqueued in that SSlot, the request number is incremented by the maximum number of slots in a session (line 39)
    ![[Untitled(205).png]]
    
    
    Source: eRPC/src/rpc_impl/rpc_req.cc
    
    Therefore, every slot will only work on requests whose request number can be represented this way: (req_num * max_slots) + slot_id. Like this, calculating the remainder of the division by max_slots will always yield the slot’s id.
    
![[Untitled(201).png]]


Source: eRPC/src/rpc_impl/rpc_rx.cc

Having found the right slot, the response needs to be put in the right response buffer.

This code snippet again looks suspiciously non-zero-copy. The paper claimed, that zero-copy was implemented for single packet responses, however in the first if brnach (line 99 through 109) there clearly is a memcpy for the size of data minus Ethernet, IP and UDP header.
![[Untitled(202).png]]


Source: eRPC/src/rpc_impl/rpc_resp.cc

Lastly, the tx_msgbuf is set to NULL, indicating that the response for the request has been received and that the request buffer can now be reused safely.
![[Untitled(203).png]]


Source: eRPC/src/rpc_impl/rpc_resp.cc