---
creator: 황준용
type: Hands-On
created: 2022-08-09
---
<<[[DPDK vs XDP Cntd.|next]]>>
## DPDK vs XDP

Experiment Trial 1: Trex using B1 Server
![[Untitled(100).png]]


If you see `UP` but not `RUNNING` in the `ifconfig -a` output, it usually means the NIC is not detecting a link from the cable, i.e. the cable is not properly plugged in or is broken.

Experiment Trial 2: JY Server → B1 Server: 홍권님

Experiment Trial 3: Trex using JY Server: 황준용

1. Trex 동작이 JY Server에서 성공적으로 동작 되는 것을 확인.
![[Untitled(101).png]]

![[Untitled(102).png]]


1. 다만, Trex 서버 구동 후, Console에서 Traffic Generator 실행을 해보니, ARP Table이 임의로 설정되어있어서 그런지, ARP Resolution Fail로 각 port가 destination을 찾지 못하는 일이 발생. 수동으로 destination port을 지정해주어도, Resoloving destination Fail.
![[Untitled(103).png]]

![[Untitled(104).png]]


1. 이에, Trex에서 Layer 2(IPv4) 말고, Layer 3(Ethernet) 형식으로 연결하여 Traffic Generating이 가능한 것을 확인하여, 아래와 같이 진행.

- Layer 2 mode - MAC level configuration
- Layer 3 mode - IPv4/IPv6 configuration
![[Untitled(105).png]]


1. 이후, script 작성, console로 실행

```python
from trex_stl_lib.api import *

class STLS1(object):
    '''
    Generalization of udp_1pkt_simple, can specify number of streams and packet length
    '''
    def create_stream (self, packet_len, stream_count):
        packets = []
        for i in range(stream_count):
            base_pkt = Ether()/IP("1.1.1.1",dst="2.2.2.2")/UDP(dport=12+i,sport=1025)
            base_pkt_len = len(base_pkt)
            base_pkt /= 'x' * max(0, packet_len - base_pkt_len)
            packets.append(STLStream(
                packet = STLPktBuilder(pkt = base_pkt),
                mode = STLTXCont()
                ))
        return packets

    def get_streams (self, direction = 0, packet_len = 64, stream_count = 1, **kwargs):
        # create 1 stream
        return self.create_stream(packet_len - 4, stream_count)

# dynamic load - used for trex console or simulator
def register():
    return STLS1()
```
![[Untitled(106).png]]


Result:

Tx Bandwith was too low, and CP Utilization stood 0%, where RX was 0
![[Untitled(107).png]]

![[Untitled(108).png]]


Experiment Final: JY Server → B1 Server using Iperf

## Firewall

What is a Firewall?

[https://velog.io/@brian11hwang/What-is-a-Firewall](https://velog.io/@brian11hwang/What-is-a-Firewall)

Firewall with Iptables

[https://velog.io/@brian11hwang/Firewall-with-IPTables](https://velog.io/@brian11hwang/Firewall-with-IPTables)

TODO:

**How to drop 10 million packets per second**:

[https://blog.cloudflare.com/how-to-drop-10-million-packets/](https://blog.cloudflare.com/how-to-drop-10-million-packets/)

**Accelerating Connection Tracking to Turbo-Charge Stateful Security:** [https://developer.nvidia.com/blog/accelerating-connection-tracking-to-turbo-charge-stateful-security/](https://developer.nvidia.com/blog/accelerating-connection-tracking-to-turbo-charge-stateful-security/)