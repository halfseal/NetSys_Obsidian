[https://byeo.tistory.com/entry/네트워크-스택의-비용에-관한-이해-3](https://byeo.tistory.com/entry/%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EC%8A%A4%ED%83%9D%EC%9D%98-%EB%B9%84%EC%9A%A9%EC%97%90-%EA%B4%80%ED%95%9C-%EC%9D%B4%ED%95%B4-3)

- 관련 한국어 리뷰 페이지

### ABSTRACT / INTRODUCTION

Short flow에 의한 overhead의 예비분석은 전체적인 상황과 환경을 보여주는 것에 실패했는데, 그 두가지 이유 중 첫 번째는 short flow만 분석했다는 점이다. 데이터센터 네트워크에서 큰 크기의 데이터는 대부분 long flow에 담겨져 있기 때문이다. 따라서 아무리 short flow가 많더라도 대부분의 CPU cycle은 long flow를 처리하는데 쓰이게 된다. 또한, 데이터센터의 작업은 이분법적으로 나누기가 힘들다. 섞여있다. 다양하다. -> CPU의 특성은 다양한 트래픽 패턴과 다양한 flow size가 섞여 있어 현저하게 바뀐다. 따라서 본 논문은 이를 모두 고려한 오늘날 네트워크 스택의 overhead를 분석하고자 한다.

[[RDMA - 원격 다이렉트 메모리 액세스]]

---

### PRELIMINARIES

Flow size (short flow, long flow)는 특정 연결의(identified by 5 tuples) 크기를 이야기한다.

결론은 100Gbps의 고대역폭 링크는 프로토콜 프로세싱에서 데이터 복사로 병목이 이동된다는 것이다. (원래는 프로토콜 프로세싱이 병목이였으나, 데이터 복사가 이제 새로운 병목이 된다는 의미)

현재까지는 모든 최적화를 통해 가장 최상의 조건에서 코어당 42Gbps의 속도를 낼 수 있음.(single long flow) 이 때 병목은 항상 kernel buffer에서 application buffer로의 데이터 복사였다.

앞서 연구에서는 Short flow/low-bandwidth에서는 프로토콜 프로세싱이 병목의 주 원인이였음.

또한 receiver side에서 패킷 프로세싱이 sender 보다 더 빨리 bottleneck이 되었음을 관찰함.

  

I/OAT -> I/O acceleration technology 하드웨어 입출력 가속 기술. DMA엔진으로, 메모리복사 일을 덜어주어 오버헤드를 크게 줄여줌

Hardware offloading -> 해당 기능을 CPU가 하지 않고 외주를 맡기는 것. 대표적인 예가 그래픽카드.

하드웨어 오프로딩 (h/w offload): NIC 하드웨어가 Linux 커널의 할 일을 일부 대신 처리해주는 방법

DCA -> direct cache access, intel의 ddio는 dma packet을 바로 L3 cache에 올려줌

근데 여기서 cache miss rate가 상당히 높다는 것을 발견함. (49%, single flow)

DCA로 오는 데이터는 L3캐시의 일부만 쓸수 있음. 1/10

병목현상이 된 호스트의 프로세싱은 호스트의 레이턴시를 증가시키게 되는 것이다.

이러한 레이턴시의 증가가 BDP의 값을 증가시키는데, 이 증가 속도가 L3 cache의 크기 증가 속도보다도 빠르다. 이말인 즉슨, DMA기술이 너무 빨라서 application이 가져다가 쓰는 속도보다 빨라서 미쳐 쓰이기 전에 overwritten이 되는 것이고, 이것이 cache miss를 불러 일으킨다 결국 window size는 L3 sizes 또한 고려해야 할 것이다.

Multiple flows sharing host resources? -> 호스트 자원을 공유하는 많은 플로우들

[[김기수/산협 프로젝트 2/논문 리뷰/Understanding Host Network Stack Overheads/NUMA]]

NUMA node -> 모든 메모리에 균등하게 접근 하는 것이 아닌, local과 remote로 메모리가 나뉘어 각각 CPU의 코어 혹은 소켓마다 할당 되는 것. 접근속도 차이가 발생함.

같은 host resources를 사용해도 flow가 multiple이라면 DMA 할 때 캐시에서 pollution이 일어나게 됨. 다른 flow가 해당 flow에 덮어쓰기를 하는 경우를 일컬음. 이것은 다른 flows에도 영향을 미침. 현존하는 최적화(패킷을 합치는 것) 때문에 패킷 처리 오버헤드가 악화됨.

이는 결과적으로 바이트당 처리 오버헤드와 스케줄링 오버헤드가 늘어나는 결과를 초래하게 됨.

- Host layering과 packet processing pipelines을 재정립 하고자 함.

Long flow에 비해 short flow가 43% 정도의 throughput-per-core 감소가 있음. 그런데 이 둘은 병목이 작용하는 이유가 다름에도 같은 processing pipeline을 사용하기 때문임. 마지막으로 NUMA domain이 다른 CPU core 위에서 작동중인 long flow는 추가적으로 20% 정도의 성능 허락이 생기고 있음.

이는 CPU scheduling에 있어서 NUMA node를 고려하여 application-known CPU scheduling을 고려하고 packet processing pipelines를 재검토해야 할 것이다.

![[images/Untitled.png]]

### Sender side(application to NIC)

application이 write 함수를 통해 system call을 실행함.

==**→ userspace에서 kernelspace로 복사하는 오버헤드가 상당히 큼.**==

Gso -> Generic Segmentation offload -> 점보프레임 등을 호스트가 사용할 수 있도록 해줌. MTU 이상의 패킷들에 대하여 CPU가 고려하는 것이 아닌 NIC가 고려하게 됨. CPU의 오버헤드를 줄일 수 있는 부분

IRQ handler -> IRQ(Interrupt Request) Handler -> 하드웨어에서 발생한 인터럽트를 처리한다는 의미

RX NAPI -> NIC가 receive에서 인터럽트와 폴링을 동시에 사용하여 효율성을 증가시킨 API임.

Gro -> generic RX offload

TSO -> Tcp segmentation offload

RPS / RFS => RSS(*Receive Side Scaling, 수신 측에서 해당 네트워크를 처리하는데 CPU 자원을 골고루 사용할 수 있도록 분산하는 방법)을 소프트웨어적으로 구현한 것. RFS는 분산처를 더 최적화 한 것이다.

[[RSS - receive side scaling]]

[[Skb - 소켓 버퍼]]

[[Rx NAPI]]

대부분의 NIC는 segmentation을 offload로 지원하게 됨

보내는 쪽에서는 application과 같은 코어에서 처리되게 됨.

각 프레임마다 skb 버퍼를 만들게 되는데, 만약 NIC가 Rx descriptor를 다 쓴다면 페이지 풀링을 통해 새롭게 descriptor를 할당하게 된다. 그리고 네트워크 하위 시스템은 GRO, LRO를 통해 merging을 하여 skb 버퍼의 개수를 줄이려고 할 것이다.

RPC -> Remote Procedure Call 현재 실행 중인 프로세스의 주소공간 내부가 아닌 외부의 프로세스 또는 원격지의 프로세스와 상호작용하기 위한 기능. TCP/IP 상위에서 작동하며, 마치 자신의 내부 프로시저를 호출하는 것처럼 보이도록 추상화 계층을 제공한다.

Flow의 종류로는 single, One-to-one, Incast, Outcast, All-to-all이 있다. 이 때 구분하는 기준은 코어로, 결국 어떻게 flow를 각 코어가 처리하는지가 관건이다.

Single flow에서는 aRFS를 껐을 때, 기본 RSS 메커니즘이 4-tuple로 결정되므로, 이를 다시 구현 가능한 결과로 도출하기가 힘들어 worst case인 application이 돌아가고 있는 NUMA node와 다른 NUMA node에서 perform될 때를 상정하고 측정하였다.

?

When aRFS is disabled, lock overhead is high at the receiver-side because of the socket contention due to the application context thread (recv system call) and the interrupt context thread (softirq) attempting to access the same socket instance.

?

BDP 값이 L3 cache보다 커지는 경우 -> 큰 TCP 버퍼는 host latency를 늘리게 된다. 이는 코어가 패킷을 처리할 때 병목이 되게 된다.

또한 TCP buffer가 작고 NIC ring buffer가 크더라도, 많은 Rx descriptor가 있다면 이 또한 userspace buffer로 data copy가 일어나지 않은 캐시를 침범할 가능성이 있다는 것이다.

결론은 오늘날의 리눅스 커널에서의 DCA는 overshooting되어서 오히려 오버헤드가 발생하게 됨. Userspace로 copy하기도 전에 계속 캐시에 쓰게 되므로 높은 cache miss가 발생하게 되는 것임.

One to one에서는 network saturation으로 overhead가 옮겨 갔음을 보여줌. 특징적인 것은 메모리할당/해제가 줄어 들었고, flow가 늘어나면서 Total Throughput이 100Gbps에 근접한 모습을 보여줌.

Incast에서는 flow가 늘어날수록 total throughput이 감소하는 모습을 보여주었는데, CPU 오버헤드에서는 큰 변화가 없었으나, 서로 다른 flow가 같은 L3 cache에 집어넣으려고 하면서 cache miss rate을 올렸기 때문에 오버헤드가 발생함. 이를 receiver 쪽에서 flow를 컨트롤 할 수 있다면 해결할 수 있겠지만, TCP의 특성상 sender-driven이므로 방해를 받게 됨.

Outcast에서는 sender를 보았는데, 8flow에서 오히려 89Gbps정도로 상승한 것을 관찰할 수 있었음. 이때 TSO의 최적화가 상당히 효율적인 것을 볼 수 있었고, 이는 TSO는 하드웨어 기반이고, GSO는 소프트웨어 기반이라는 차이점이 있다. 이는 CPU 자원을 소모하지 않는다는 장점이 있다. 또한 flow의 개수에 상관 없이 항상 64KB 사이즈의 skb들에 집어넣으므로, TSO의 효율성이 flow가 증가하더라도 변하지 않게 된다. 또한 aRFS가 상당한 효율성을 주게 되는데, 이는 receiver side와는 달리 보내야 할 data가 항상 warm 상태이기 때문에 cache miss가 날 일이 거의 없기 때문이다. 또한, Outcast는 서로 다른 NUMA-node에 전송하므로, 같은 L3 cache에서 compete이 일어날 일이 거의 없을 것이다. 따라서, Outcast에서의 bottleneck은 sender에서의 data copy이다.

마지막으로 All to all은 명시적으로 IRQ를 매핑하지 못했는데, 이를 다루기 위한 NIC의 성능이 부족하기 때문이다. 그러나 이는 통계적으로 유의미하므로, 이를 해석하였다. 여기서는 one-to-one과 비슷하게 네트워크 용량이 문제가 되는데, 각각의 flow는 작은 bandwidth을 가지게 된다. 이는 일괄처리하는 optimization등의 효율성을 떨어뜨리게 된다. 특히 GRO의 경우 flow당 동작을 하므로 더욱 그렇다. 이는 작은 skb 여러 개를 upper level로 전달하게 되고, CPU가 처리하는데 상당한 오버헤드를 일으킨다. 여기서는 다양한 flow에 의해 작아진 Skb의 크기에 의한 효과가 두드러지게 나타나고 있다.

또한 패킷 손실에 의한 오버헤드도 보여주고 있다. Receiver 측에서는 패킷 손실이 일어날 경우 ACK가 중복되어 보내지므로 이를 처리하기 위해 CPU 오버헤드가 늘어나게 된다. 또한, packet drop이 많이 일어날 수록 sender와 receiver간 CPU 사용률이 비슷해져 가는데, 이는 sender 측에서 더 많은 오버헤드가 일어나고 있음을 가리킨다.

그 다음은 flow size에 따른 overhead를 관찰하였는데, 매우 짧은 flow는 Data copy보단 TCP/IP processing과 Scheduling에서 많은 resource를 요구하였다. 이는 DCA가 전체 성능에 많은 영향을 못 주기 때문이며, cache miss 또한 local이 remote보다 유의미하게 적었다. 세 번째로, DCA에 의한 이득이 매우 짧은 flow에서는 적었는데, 다른 cache locality benefits of aRFS는 여전히 적용된다. 예를 들어, skb에서 packet processing이 이루어 질 때 이며, 이는 application이 작동 중일 때와는 독립적이다.

여기서 중요한 핵심 성능 향상 방법은, long flow는 local node에서, short flow는 remote node에서 실행한다. 또한 short flow를 잘 Scheduling 한다면 이는 성능 항상으로 이어질 수 있다.

여기서 또다른 주목할 만한 점은 long flow와 short flow가 섞였을 경우 성능 하락이 상당히 심각하므로 절대 섞이면 안될 것이다.

DCA에 의한 영향은 당연히 성능의 증가이다. DCA를 끄게 되면 NIC가 DMA를 통해 바로 L3 cache로 데이터를 복사하는 이득이 줄어들기 때문이다.

IOMMU는 가상환경에서 효율적으로 가상화 된 IO devices들을 사용하기 위함이다. 가상환경이 아니더라도, 메모리 보호를 위해 유용하다. 이는 가상화 하는데 CPU 자원을 필요로 하므로, 성능이 감소하게 된다.

Congestion control에 의한 영향은 미미하다. 이는 sender에 의해 바뀌는데, 실질적으로 성능은 receiver가 bottleneck이기 때문이다.

100Gbps를 달성하는 것은 zero-copy를 통해 달성할 수 있는데, sender-side에서는 이미 구현을 통해 되었으나, receiver-side가 여전히 bottle-neck으로 남아있다. 불행히도, receiver-side의 zero-copy는 기존의 메모리 관리 방법론을 수정해야 하므로, 어플리케이션을 일일이 수정해야한다.