---
creator: 구성현
type: Paper Review
created: 2022-07-12
---
<<[[Paper XDP Summary|previous]]>>

<aside> 👀 소프트웨어에서 패킷 프로세싱은 커널을 bypass 하는 것임 → bypass하면 시간이 덜든다는 장점과 함께 app을 기능적으로 다시 구현하거나 보안의 문제가 있음 → 고로 논문에서 XDP를 소개함

- 어떻게 고침? → 커널의 네트워크 스택을 처리하기 직전에 eBPF 코드를 사용하는 XDP를 둠
    
- 뭘로 구성되어있는데? → XDP driver hook + eBPF virtual machine + BPF map + eBPF verifier
    
- 각각이 뭐하는데? → XDP driver hook은 커널 들어가기 context object를 보고 어디로 보낼 것인지 결정함 → eBPF virtual machine은 프로그램을 유동적으로 로딩하거나 재로딩함 → BPF map은 eBPF 프로그램들이나 user space와의 소통을 지원 → eBPF verifier는 프로그램이 커널에 harm하는 행동을 하지 않도록 ensure
    
- 그래서 뭐가 좋아짐? → 현재 Linux packet processing보다 좋은 성능을 보여줌 → 현재 OS kernel network과 compatible해서 안전성을 확보 → 따로 특별한 하드웨어나 자원 없이 dynamic하게 re-program 가능
    

</aside>

INDEX

## PERFORMANCE EVALUATION

---

> 실험 환경은 아래와 같습니다. 해당 논문은 3개의 metric에 집중했는데, 하나는 최대 packet processing performance를 확인하기 위한 packet drop performance이고, 나머지 2개는 CPU usage와 packet forwarding performance입니다.

- Setting
    - Intel Xeon E5-1650 v4 CPU running at 3.60GHz with Intel’s Data Direct I/O (DDIO)
    - 2 Mellanox ConnectX-5 Ex VPI dual-port 100Gbps network adapters
    - TRex packet generator
    - Linux kernel 4.18
- Focus on three metrics
    - Packet drop performance
    - CPU usage
    - Packet forwarding performance

### Packet Drop Performance

> 해당 그래프는 코어의 개수에 따라 packet drop performance를 보여줍니다. 한 코어에서 XDP의 baseline performance는 24Mpps이고 DPDK는 43.5Mpps입니다. 둘 다 PCI bus의 global perfomance limit에 도달할 때까지 linearly하게 성능을 증가시킵니다. Linux의 conntrack mode은 한 코어의 성능이 1.8Mpps이고, raw mode에서는 최대 4.8Mpps입니다. hardware bottleneck이 없을 경우 Linux performance는 코어의 수에 따라 linear하게 증가합니다.
![[Untitled(68).png]]


Figure 3 : Packet drop performance

- The baseline performance of XDP for a single core is 24 Mpps for DPDK it is 43.5 Mpps
    - Both scale their performance linearly
    - until they approach the global performance limit of the PCI bus (reached at 115Mpps)
- with conntrack from 1.8Mpps of single-core performance, up to 4.8 Mpps in raw mode
    - Linux performance scales linearly with the number of cores

### CPU Usage

> DPDK가 packet processing과 busy polling으로 모든 코어를 사용하기 때문에 CPU usage가 항상 100%인 것을 볼 수 있습니다. 반대로, XDP와 Linux는 smooth하게 CPU usage를 scale하는 것을 볼 수 있습니다.
![[Untitled(69).png]]


Figure 4 : CPU usage in the drop scenario

- DPDK’s CPU usage is always pegged at 100%
    - since DPDK dedicates a full core to packet processing and uses busy polling
- XDP and Linux smoothly scale CPU usage
- The non-linearity in the bottom-left corner is due to the fixed overhead of interrupt processing

### Packet Forwarding Performance

> XDP는 다른 NIC 뿐만 아니라 같은 NIC으로도 packet을 forward할 수 있어 두가지를 넣었다고 합니다. DPDK는 다른 interface로만 packet forward를 지원합니다. Linux는 minimal forwarding mode를 지원하지 않아 생략했다고 합니다. 같은 interface로 packet을 보낼 때 XDP 성능이 향상된 것을 볼 수 있습니다.
![[Untitled(70).png]]


Figure 5 : Packet forwarding throughput

- XDP can forward packets out the same NIC as well as out a different NIC
- DPDK only supports forwarding packets through a different interface
- Linux networking stack does not support minimal forwarding mode
    - but requires a full bridging or routing lookup
    - omit it from the results
- XDP performance improves, surpassing the DPDK forwarding at two cores and above
    - due to differences in memory handling

## CONCLUSION

---

> XDP는 빠른 programmable packet processing과 OS 커널을 안전하게 합친 시스템입니다. XDP는 raw packet processing performance에서 한 코어에 대해 최대 24Mpps를 도달했습니다. 이는 커널의 안전과 management compatibility를 포함하고 성능을 향상했다는 것을 보여줍니다. 또한, XDP는 dynamic하게 service interruption없이 re-program할 수 있고 특별한 하드웨어나 resource를 필요로 하지 않습니다.

- XDP, a system for safely integrating fast programmable packet processing into the OS kernel
- achieves raw packet processing performance of up to 24 Mpps on a single CPU core

## More …

---

- ~~packet drop performance가 높으면 좋은 건가요?~~
    - measure the performance of the simplest possible operation of dropping the incoming packet. this effectively measures the overhead of the system as a whole … [PERFORMANCE EVALUATION](https://www.notion.so/PERFORMANCE-EVALUATION-528cc92d7ef244b6abf8d8e261428acc?pvs=21)
- PCI bus의 global performance limit가 잘 이해가 안돼요
    - global performance limit of the PCI bus, which is reached at 115Mpps after enabling PCI descriptor compression support in the hardware [Packet Drop Performance](https://www.notion.so/Packet-Drop-Performance-f501f70287704d80a0e5ca34c0d4bbdf?pvs=21)
- 작은 packet rates에서 인터럽트도 작은데 왜 패킷당 CPU usage가 높게 나오는 것인가요?
    - At lower packet rates, the number of packets processed during each interrupt is smaller, leading to higher CPU usage per packet [CPU Usage](https://www.notion.so/CPU-Usage-7e0945e184c94f5ea3e3fb3e3746224b?pvs=21)