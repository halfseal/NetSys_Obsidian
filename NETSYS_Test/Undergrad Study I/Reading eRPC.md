---
creator: Maike
type: Paper Review
created: 2022-08-16
---
Unfamiliar Terminology:

- ~~RPC~~ ([https://www.ibm.com/docs/en/aix/7.1?topic=concepts-remote-procedure-call](https://www.ibm.com/docs/en/aix/7.1?topic=concepts-remote-procedure-call))
- Marshalling ([https://www.techopedia.com/definition/16408/marshalling](https://www.techopedia.com/definition/16408/marshalling))
- “opaque” buffer ([https://stackoverflow.com/questions/38919334/how-can-a-destination-buffer-be-opaque](https://stackoverflow.com/questions/38919334/how-can-a-destination-buffer-be-opaque))
- ~~Cache Thrashing~~ ([](https://en.wikipedia.org/wiki/Thrashing_(computer_science))[https://en.wikipedia.org/wiki/Thrashing_(computer_science)](https://en.wikipedia.org/wiki/Thrashing_(computer_science)))
- ~~PFC~~ ([https://info.support.huawei.com/info-finder/encyclopedia/en/PFC.html](https://info.support.huawei.com/info-finder/encyclopedia/en/PFC.html))
- ~~oversubscription~~ ([https://www.noction.com/blog/oversubscription-in-networking](https://www.noction.com/blog/oversubscription-in-networking))
- jitter ([https://www.ir.com/guides/what-is-network-jitter](https://www.ir.com/guides/what-is-network-jitter))

### RPC (Remote Procedure Call)

RPC is a client-to-server message passing protocol that usually relies on lower-level transport protocols to relay said messages. Its purpose is for clients to use services that the server provides by sending requests for certain actions to the server alongside the necessary data, which the server will process and reply with the expected response.

### PFC (Priority-based Flow Control)

PFC is an in-network mechanism that prevents packet loss. It acts on switches in the network. Once a switch’s receive queue is running out of buffer space, the send queue of the upstream device is paused through explicit pause requests, keeping it from overflowing the receive buffer and hence causing packet loss.

### Cache Thrashing

The general term “Thrashing” is best summed up as a type of behaviour when the system gets so caught up in performing preliminary tasks that no actual work is being done. In the case of cache thrashing, it refers to a situation where - due to certain access patterns - cache lines are frequently evicted which leads to excessive cache misses that keep the system occupied with handling the cache misses rather than making progress on the actual process.

### Oversubscription

Generally, oversubscription is the act of giving out more of a resource than is actually available. In multi-layer networks, this often refers to the act of connecting more devices to a single switch port than that port could handle. It is based on the assumption that it is unlikely that all connected devices will require the port at the same time.

### Marshalling [INCOMPLETE]

Marshalling describes the act of changing an objects memory representation.

### Opaque

If some data structure is “opaque”, it means that it can be accessed by the user, but the user has no information on the actual implementation of the data structure and can only interact with it through predefined interfaces. It is the opposite of “transparent”, where the user has full knowledge of an object’s implementation.

Questions: