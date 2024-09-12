1. Intro

- TSO: TCP segmentation offload
    - NIC가 data를 작은 TCP segment로 쪼갠다
    - 보통 cpu의 역할이라서 offloading이다
- GSO: Generic segmentation offload
    - TSO와 유사하지만 software based이다
- GRO: Generic receiver offload
    - network layer 전에 여러 packet들을 하나로 합친다
    - 이는 cpu가 process해야하는 packet의 수를 감소시켜 offloading한다
- LRO: large receiver offload
    - NIC가 하는 GRO
- Jumbo frame
    - 평소의 ethernet frame보다 큰 프레임
    - packet의 숫자를 감소켜서 cpu processing overhead 감소
- RFS: receive flow steering
    - packet processing을 할 cpu로 packet을 보낸다
    - cache locality 개선
- RSS: receive side scaling
    - hard ware 방식, NIC가 hash function으로 cpu 분배
- ex
    - end-host netw: network traffic의 source or destination이 되는 장치
    - access link bandwidth: user, local netw 사이 bw
    - bandwidth delay product
        - netw이 감당할 수 있는 in-flight data 양
        - data link capacity * rtt

  

1. Summary

- Data copy 중심의 bottleneck
- BDP와 L3 cache size의 차이가 작아지면서 throughput 감소
    - cache에서 application으로의 data copy가 발생하기 전에 새로운 data 가 overwrite한다
    - 이는 cache miss rate 증가시킨다
    - L3 cache size 를 고려하는 transport protocol 이 필요(window size)
    - short flow로 실험하면 안된다
    - 대부분의 cpu cycle은 long flow에서 발생한다
- Host sharing은 나쁘다
    - 하나의 NUMA node에 여러 flow가 들어오면 성능악화
    - generic receiver offload가 하나의 큰패킷으로 합치기 힘들다
- Host layer, packet processing pipeline의 재편이 필요
    - short flow: packet processing 중심
    - long flow: data copy 중심
    - 현재는 flow을 반영하지않고 일괄적인 pipeline 사용한다
- network stack latency는 이 논문에 반영하지 않음

  

1. preliminary

- app (data copy, scheduling)
- socket interface (skb)
    - network packet을 저장하기 위한 자료구조이다
    - 커널의 메모리에서 할당된다
- TCP/IP
- network subsystem (skb management)
- device driver (mem alloc, dealloc, scheduling)
- Sender
    - app write → socket → tcp/ip state → netfilter → xps →queueing discipline → GSO → driver TX
    - network subsystem에서 GSO에 의해 skb가 MTU 크기로 segment된다
    - app 과 같은 cpu에서 network stack이 진행된다
- Receiver
    - NIC에는 Rx descriptor가 존재
    - Rx descriptor은 memory가 할당되어있다(mem addr 보유)
    - NIC이 data copy를 위한 interrupt 발생
    - NAPI에 의한 polling
    - NIC가 IRQ를 할 cpu core를 정한다
    - aRFS가 있을 경우, 모든 processing(irq, tcp/ip, app)은 한 코어에서 발생
    - packet의 data copy는 한번만 발생한다
    - irq handler → RX NAPI → GRO → RPS/RFS → netfilter → tcp/ip state → socket → app read
        - irq: interrupt request
        - RX NAPI: receiver new api, polling + interrupt
            - 너무 많은 interrupt이 발생할 경우 overhead를 줄이기위해 polling방식을 도입
        - netfilter: linux kernel이 제공하는 network 관련 소프트웨어
            - NAT, packet filtering, port translation
- traffic pattern
    - single flow: 1 sender, 1 receiver
    - one-to-one: (1 send, 1 receive) * n
    - incast: n send, 1 receive
    - outcast: 1 send, n receive
    - all-to-all: n end, n receive
- skb: socket buffer, stores packet data
- MTU: maximum transmission unit, max size of frame
- RX descriptor: nic packet을 보관하기 위한 data structure
- xps/rps: transmit packet steering, receive packet steering

  

1. Measurement methodology

- ==RPC workload: remote procedure call, ????==
- IOMMU: input output memory management unit
- Performance metric
    - total throughput, total cpu utilization
    - throughput-per-core-ratio
    - total cpu utilization at bottleneck
    - ~~application workload가 낮은 상황을 가정하는 이유???~~

3.1 Single flow

- aRFS disable 시, numa remote node에 irq를 할당한다
    - app core에 할당, numa node를 공유하는 다른 core, 다른 node의 core
- receiver에서 상대적으로 netw subsystem, mem alloc overhead가 높다
    - data copy
        - arfs비활성상태에 numa remote node로 전송되서 datacopy 증가
    - skb allocation
        - sender는 TSO지만 receiver는 software base 인 GRO
- high cache miss rate
- tcp buffer size 가 커지면 data copy 시간이 늘어나고 이는 cache miss rate를 증가시킨다
    - tcp buffer size는 LLC의 크기를 감안해야한다
- GRO is performed in netw subsystem
- softirq: hardware interrput를 software interrupt로 처리하는 방식
- High cache miss rate
    - tcp receive window size가 클수록 miss
        - data copy와 skb생성 사이의 시간이 커진다
        - 이는 data copy 이전에 새로운 dma가 되어서 cache miss 유발
    - rx descriptor가 많을수록 miss
        - address의 개수가 많아져서 성능저하

3.2 one to one

- optimization 성능의 감소
    - through put per core 감소
        - cache locality 감소( 모든 core가 L3 cache 공유), remote numa
        - GRO이 더 큰 skb로 합치지 못한다
    - 여전히 data copy가 중심이지만 scheduling overhead가 증가
        - flow가 늘어나면 netw 이 saturated되고
        - 이는 cpu가 data가 없어 sleep wake를 반복시킨다
        - 이는 scheduling 증가
    - mem allocate는 감소
        - 코어당 traffic이 감소해서 pageset이 가진 page를 재사용가능
        - 이전에는 더 비싼 global free list를 사용

3.3 incast

- cpu breakdown는 변화없음
- per-core-throughput 감소
    - cache miss rate 상승이 원인
    - receiver-driven protocol이 필요

3.4 outcast

- sender의 bottleneck관찰하낟
- sender가 receiver보다 cpu efficient
    - TSO가 GRO보다 성능이 좋다
        - tso는 하드웨어 gro는 소프트웨어
        - tso는 gro와 달리 항상 64kb skb에 data를 담아서 flow의 개수가 증가해도 성능저하가 일어나지 않는다.
        - ?????왜 그게 가능한가 flow가 많아져도 64넣을수있는이유
    - sender의 cache는 warm한 경우가 많다????????????

3.5 all-to-all

- arfs의 성능감소
    - cache miss rate의 저하
    - 그러나 one to one에 비해 내려가지는 않았다
        - 이미 cache miss rate가 높았다
    - skb의 크기도 감소
        - 네트워크에서 전달되는 packet의 양이 적어 skb도 작다
        - packet processing overehead가 증가된다

3.6 in-network congestion

- netw안에서 랜덤으로 packet drop 한다
- drop rate를 높일 수록 throughput 감소
    - 하지만 throughput이 올라가는 drop rate도 있었다
- ack processing이 증가해서 성능저하
    - duplicate ack도 발생한다
    - sender에서 성능저하가 더 커지는데, ack를 처리해야해서이다

3.7 flow size

- message size 4kB ~ 64kB, long short flow 혼합해서 실험
- short flow에서 DCA는 효과 없다
    - data copy가 bottleneck이 아니다
    - scheduling, tcp/ip processing이 bottleneck이다
- arfs는 아직 유효
    - numa local, numa remote의 cache miss rate에는 큰 차이가 있다
    - 그러나 short flow에서 throughput은 비슷하다
    - short flow는 remote에 long flow는 local에 보내는 방식이 좋을지도
    - 그러나 4kB이상에서 위 관측은 의미없어진다.
    - 다시 data copy가 bottleneck이 된다
- RPC: remote process call
- long short flow를 하나의 코어에 실행하는 것은 성능을 저하

3.8 DCA

- single flow 실험을 dca 없이 실행
- throughput 20%감소
- 여전히 data copy, receiver가 bottleneck

3.10 congestion protocol

- 성능 차이가 거의 없다
- protocol 자체가 sender driven이라 receiver 가 bottleneck인 상태에서는 성능 차이가 없다

  

1. future direction

- zero copy
- cpu efficient transport protocol (receiver driven)
- host stack fixing