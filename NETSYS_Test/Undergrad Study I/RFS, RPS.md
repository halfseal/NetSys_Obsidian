---
creator: Maike
type: Glossary
created: 2022-07-05
---
## RPS

### What is RPS?

**R**eceive **P**acket **S**teering

Uses 5-tuple hash to determine CPU core to whose backlog to enqueue the packet.

### Why RPS?

To distribute packet processing across different cores and improve parallelism

RPS published in kernel 2.6.35 in August 2010

CPUs during that time had between 1 and 4 cores.
![[Untitled(53).png]]

[Source](https://www.cpu-world.com/Releases/Desktop_CPU_releases_(2009).html)

### How does RPS work?

RPS uses the 5-tuple hash (either calculated when needed or offloaded to NIC)

Use said hash to access data structure with CPU mappings

Advantages:

1. Different Connections are evenly distributed across available cores
2. Packets for the same application map to the same core, improving locality
3. Further splitting up the packet processing pipeline

**But**, the processing core might not be the core that actually consumes the packet data (aka is running the application)

## RFS

### What is RFS?

**R**eceive **F**low **S**teering

Uses same 5-tuple hash to access into a flow table that indicates which CPU core is running the application and enqueues the packet there.

### Why RFS?

To improve cache locality by ensuring that packet processing and application run on the same core.

### How does RFS work?

Determine CPU based on entries in two different flow tables: Global flow table and local receive queue table

Global Flow Table indicates the _desired CPU_. Every time an application sends/receives data, the mapping between application and core gets updated accordingly, indicating the core who had the most recent interaction with the application.

Global Flow Table is not enough. Desired CPU could be updated during packet processing and changing core mappings during processing could lead to out-of-order packets.

To fight the problem, we have the local receive queue table that saves the information on the last CPU that packets were enqueued on, the so-called _current_ _CPU_. If the desired CPU and the current CPU are different, the current CPU value is updated if either:

1. The current CPU is unset.
2. The current CPU is offline.
3. All packets since last time have been processed.

The last one is determined as such: Every backlog has a counter for how many packets have been dequeued and how many packets are currently on the backlog. The sum of those two is saved as the qtail value. By the time desired CPU and current CPU are compared, the qtail value from the last enqueued packet is compared with the counter for processed packets. If there is an equal or bigger amount of processed packets than there were processed + outstanding packets at the time of enqueueing, there are no packets for that flow pending on the backlog and the CPU can be safely changed.

Code Reading:

Following code from linux/drivers/ethernet/intel/e100.c

e100_poll → e100_rx_clean → e100_rx_indicate → netif_receive_skb → netif_receive_skb_internal → get_rps_cpu

Sources:

[Detailed Overview of all Steering Mechanisms](https://www.kernel.org/doc/Documentation/networking/scaling.txt)

[Good, rather low-level explanation of RFS](https://lwn.net/Articles/382428/)

[Short Description of RPS](https://lwn.net/Articles/362339/)