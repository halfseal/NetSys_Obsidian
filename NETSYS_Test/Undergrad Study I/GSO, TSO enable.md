---
creator: 김명수
type: Hands-On
created: 2022-05-03
---
<<[[GSO, TSO cntd.|next]]>>
- $ethtool -k [ens33]


![[Untitled(12).png]]
![[Untitled(13).png]]

[https://www.kernel.org/doc/html/latest/networking/segmentation-offloads.html](https://www.kernel.org/doc/html/latest/networking/segmentation-offloads.html)

[https://github.com/torvalds/linux/tree/master/net](https://github.com/torvalds/linux/tree/master/net)

[https://lwn.net/Articles/188489/](https://lwn.net/Articles/188489/)

- _tcp_offload.c_
![[Untitled(14).png]]

![[Untitled(21).png]]


- shb_shinfo() → gso_type : SKB_GSO_TCPV4 or SKB_GSO_TCPV6 정보
- netdev_features_t : net device가 offload 가능한 device인지 판단 / _in linux/inlcude/linux/netdevice.h_
![[Untitled(15).png]]


- callback 구조체 정의 / _in linux/include/linux/netdevice.h_
![[Untitled(16).png]]


- IP header, TCP header의 offset를 결정?? → IPv4인지 IPv6인지에 따라 header가 달리지기 때문?
    
- tcp_offload.c
    
![[Untitled(17).png]]


- segs = skb_segment(skb, features); skb를 mss사이즈의 여러 skb로(segs) segmentation수행
    
- skb_shinfo() → gso_size() : segment하는 size 정보(==mss) / _in linux/include/linux/skbuff.h_
    
- _linux/net/core/skbuff.c_
    
![[Untitled(18).png]]


- skb_segment() : gso_size()기반으로 segmentation수행
    
- tcp_offload.c
    
![[Untitled(19).png]]


- gso_size크기로 나누어진 skb들(segs)에 header 붙이기
    
- _linux/include/linux/skbuff.h_
    ![[Untitled(20).png]]
    
    
    - -skb_shared_info 구조체