---
creator: Maike
type: Hands-On
created: 2022-08-23
---
## Driver

Driver functionalities seem to be located in the src/transport_impl folder.

Options are

- DPDK
- Infiniband
- Raw
- Fake

Fake is just empty placeholder functions, so definitely not functional.

Raw could be functional, but given that the options for execution are DPDK or Infiniband, it does not seem to be used.

> A userspace NIC driver is required for good performance.

When using DPDK, automatically the DPDK driver will be used, which I guess qualifies as a userspace driver. (?)

There definitely is **no custom driver** included in eRPC. The files in the transport_impl folder seem to be more like a communication layer between the application and DPDK/Infiniband.

## Zero Copy

Examining process mentioned in 4.2.2 Message Buffer Ownership describing the TX flush upon retransmission.
![[Untitled(125).png]]
![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/87b94199-7199-4300-9bfd-b646b31ccfc9/Untitled.png)

(eRPC/src/rpc_impl/rpc_pk_loss.cc:55)

I was curious at what points the TX queue was flushed and there are five instances:

1. When the client detects packet loss (/src/rpc_impl/rpc_pkt_loss.cc:57)
2. When the client detects node failure (/src/rpc_impl/rpc_pkt_loss.cc:45)
3. When the server receives a duplicate small request (/src/rpc_inpl/rpc_req.cc:101)
4. When the server receives a duplicate large request (/src/rpc_inpl/rpc_req.cc:199)
5. When the server processes an out-of-order RFR(/src/rpc_impl/rpc_rfr.cc:61)

**Node Failure vs Packet Loss**

How does one differentiate? Checking the code, the node failure code has been deactivated:

So I guess node failure is now handled just like a normal packet loss. Maybe because it is hard to differentiate the two…
![[Untitled(126).png]]
![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/4a3c7cbc-d9a1-4a67-a465-d890643e9d82/Untitled.png)

## Addition: Receiving a Packet in eRPC - Traced

RX-related transport functions:

- rx_burst()
- drain_rx_queue()

Main function: rx_burst()
![[Untitled(127).png]]
![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8d93779c-fc24-4ef2-9d69-c35338688b4d/Untitled.png)

Packets are received in process_comps_st.

eRPC seems to have little to no IP or UDP processing at all.

General Flow:

1. In event loop, call process_comps_st()
2. In process_comps_st() call the rx_burst() function associated with the chosen transport protocol (here DPDK)
3. Receive burst of packets by using the rte_eth_rx_burst() function
4. Copy packet data into rx_ring using rte_pktmbuf_mtod()
5. Temporarily store packet data of each packet in a pkthdr structure
6. If DEBUG is on, check IP address and UDP port
7. Repeat Step 4 through 6 for every packet in the burst.
8. Check Magic, meaning to drop any packet that is not eRPC.

_End of “conventional” packet processing_

What about

- Checksum? (Is it offloaded to the NIC? DPDK?)
- Multiple eRPC instances running on one node?
- Routing Errors?