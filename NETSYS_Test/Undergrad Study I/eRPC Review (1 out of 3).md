---
creator: 김명수
type: Paper Review
created: 2022-08-16
---
until section 3

# RPC(Remote Procedure call)
![[Untitled(123).png]]


-Combining with OSI layer(transport, network…) what does header look like?

-How does server knows incoming data is “that RPC msg”
![[Untitled(124).png]]


Dispatch threads run RPC event loop(… **run request handler through worker threads** … )

Worker threads run request handler and process request

When all handlers run through dispatch threads, tail latency will increase

When all handlers run through dispatch threads → worker threads, communication between dispatch and worker is expensive

### New words

- Lossy Ethernet, lossless Ethernet
    - Lossless ethernet is eliminate loss due to queue overflow
- PFC
    - To achieve lossless ethernet
- RDMA perform poorly when pkt loss…why?