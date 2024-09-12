---
creator: Maike
type: Glossary
created: 2022-06-21
---
**What is NAPI polling?**

Name: **N**ew **API** First introduced at the 5th Annual Linux Showcase & Conference 2001 in the paper “Beyond Softnet”([https://www.usenix.org/legacy/publications/library/proceedings/als01/full_papers/jamal/jamal.pdf](https://www.usenix.org/legacy/publications/library/proceedings/als01/full_papers/jamal/jamal.pdf))

> NAPI offers a middle ground between interrupts and polling. Interfaces are allowed to interrupt on the arrival of the first packet in a batch. They then register to the system that they have work. The interfaces subsequently turn off any interrupts. At some later point, a softirq is activated to poll all devices that registered to offer packets. All interfaces are given an opportunity to send up to a (configurable) number of packets known as quota. 

**Side Note: What is a softirq?**

Softirq == Software Interrupt

> The softirq mechanism is meant to handle processing that is almost - but not quite - as important as the handling of hardware interrupts.

Great Reference: [https://lwn.net/Articles/520076/](https://lwn.net/Articles/520076/)

**Why NAPI?**

Originally: Inform the host of **every packet** that arrived on the NIC.

That means a highest priority non-preemptable hardware interrupt for every single packet.

Keep in mind: The packets are already in host memory. All the interrupts do is tell the host they arrived.

Host does not need to know if there are 1,2 or 15 packets.

Host needs to know: Are there packets? Yes or No?

Former requires x interrupts, latter only one.

**The Process of Receiving Packets**

Detailed Explanation: [https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-receiving-data/](https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-receiving-data/)

The Gist:

1. Driver is loaded and initialized.
2. Packet arrives at the NIC from the network.
3. Packet is copied (via DMA) to a ring buffer in kernel memory.
4. Hardware interrupt is generated to let the system know a packet is in memory.
5. Driver calls into NAPI to start a poll loop if one was not running already. 
6. `ksoftirqd` processes run on each CPU on the system. They are registered at boot time. The `ksoftirqd` processes pull packets off the ring buffer by calling the NAPI `poll` function that the device driver registered during initialization.
7. Memory regions in the ring buffer that have had network data written to them are unmapped.
8. Data that was DMA’d into memory is passed up the networking layer as an ‘skb’ for more processing.
9. Incoming network data frames are distributed among multiple CPUs if packet steering is enabled or if the NIC has multiple receive queues.
10. Network data frames are handed to the protocol layers from the queues.
11. Protocol layers process data.
12. Data is added to receive buffers attached to sockets by protocol layers.

**Code Reading - e100**

(linux/drivers/net/ethernet/intel/e100.c)

Driver initialization: from e100_probe()

Driver interrupt routine: from e100_intr()

NAPI polling: from e100_poll()

**Softirqs during NAPI**

This article ([https://lwn.net/Articles/687617/](https://lwn.net/Articles/687617/)) touches down on NAPI polling and its relation to softirqs.

1. Softirqs are an old mechanism of the kernel. People use them as scarcely as possible. (/linux/include/interrupt.h)
2. When looking for the NET_RX_SOFTIRQ (which I assume to be the softirq for incoming packets), it is only used four times in dev.c The function that it is called in are:
    1. net_rx_action()
    2. ____napi_schedule
    3. rps_ipi_queued
    4. net_dev_init

In net_dev_init, we define the action for the NET_RX_SOFTIRQ to be the function net_rx_action. Inside the net_rx_action function, the napi_poll function is called, initiating the polling.

The question remains: We KNOW that it is not always the case that a single thread handles all traffic. So when does the packet processing get split?