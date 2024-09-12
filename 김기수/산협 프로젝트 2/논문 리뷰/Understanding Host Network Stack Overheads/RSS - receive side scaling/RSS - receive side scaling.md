[https://docs.kernel.org/networking/scaling.html](https://docs.kernel.org/networking/scaling.html)

오늘날 대부분의 NIC들은 여러 개의 송/수신 descriptor queue를 사용할 수 있게 지원한다.

NIC는 멀티프로세서를 통해 서로다른 패킷을 서로 다른 큐를 통해 분산 처리가 가능하다.

NIC는 패킷을 필터링을 통해 각각의 논리적인 흐름으로 분산하여 할당하게 된다.

이때 각각의 분리 된 receive queue는 구분 된 CPU들이 각자 처리하게 된다.

이는 4계층 및 3계층 헤더정보를 hash한 값으로 filtering 하게 된다. 보통 128개 엔트리의 간접테이블을 사용하는데, 각각의 엔트리는 큐의 번호를 저장한다.

보통 새로운 패킷이 도착하면 Toeplitz 해시의 LSB 7자리를 기준으로 결정된다.

몇몇 NIC는 symmetric RSS 해싱을 지원하는데, 이는 송/수신자가 바뀌어도 같은 엔트리의 큐로 들어가게 해준다.(해시값이 같음) → 이는 몇몇 application에서 유리함.(IDS, firewalls) 또한 양방향의 flow에서도 같은 Rx queue에 들어가기 위해서 반드시 필요하다.

이때, 두 개의 4 tuples 에 대하여 symmetric-XOR은 서로 XOR을 통해 같은 flow인지 판별할 수 있는 하나의 방법이다.

```C
# (SRC_IP ^ DST_IP, SRC_IP ^ DST_IP, SRC_PORT ^ DST_PORT, SRC_PORT ^ DST_PORT)
```

만약 4개 필드 모두 0이라면 이는 같은 flow라고 할 수 있을 것이다.

더 진보한 NIC의 경우, programmable filters 옵션을 통해 packet steering을 제공한다.

receive queue는 만약 기기가 지원한다면 CPU당 하나, 아니면 메모리 영역(L1, L2, L3, NUMA node, etc..)당 하나씩 할당한다.

### RSS IRQ Configuration

각각의 Rx queue는 구분되어진 연관된 IRQ를 가지고 있다. NIC가 이걸 trigger하여 CPU가 새로운 패킷이 들어왔음을 알린다. PCIe를 통해 인터럽트를 일으키는 것으로 MSI-x를 사용하는데, 흔히 ifconfig를 했을 때 나오는 eth(x) 이 것을 가리킨다. 하드웨어 핀으로 인터럽트를 발생시키는 것은 장치 갯수에 한계가 있어 이를 메시지 형태로 발생시키게 하여 더 많은 장치를 다룰 수 있도록 하는 것이다. 이를 이용하여 NIC는 특정 CPU를 지정하여 처리하도록 보낼 수가 있다. 각각의 큐와 IRQ에 대한 매핑은 /proc/interrupt 에서 설정이 가능하다.

기본적으로는 어떤 CPU이든지 IRQ를 핸들링할 수 있는데, 패킷 프로세싱에서 이 인터럽트 처리가 상당한 부분을 차지하고 있기 때문이다. 추가적으로 몇몇 시스템들은 irqbalance를 사용중일 수도 있는데, 동적으로 IRQ처리를 매핑하여 효율성을 끌어올리는 데몬이다.

RSS는 레이턴시가 우려되거나 수신 인터럽트 처리가 병목현상이 발생할 때 반드시 켜야할 것이다. 이러한 부하를 CPU들 사이에 분산시키는것이 queue의 길이를 줄이기 때문이다. 또한, CPU가 감당가능한 선(오버플로우가 일어나지 않는 선)에서 가장 적은 receive queue를 가진 상태가 최적화된 상태일 것이다.(receive queue가 많을 수록 인터럽트 처리 갯수가 늘어나므로)

추가적으로 하이퍼쓰레딩의 경우 RSS에 의한 이득이 없으므로, 큐의 갯수는 CPU의 코어수를 넘어서는 안될 것이다.

### RPS: Receive packet Steering

RPS는 RSS를 논리적으로 소프트웨어 형태로 구현한 것이다. 따라서 나중에 데이터 경로에서 호출하는 것이 반드시 필요하다. RSS가 큐를 선택하여 하드웨어 인터럽트 핸들러를 실행할 CPU를 선택하는 반면, RPS는 인터럽트 핸들러 위의 프로토콜 처리를 수행하기 위해 CPU를 선택한다.

이후, CPU의 작업 큐에 해당 패킷을 올려놓고, 처리를 위해 CPU를 깨우는 것으로 RPS의 역할이 끝난다.

RPS는 수신 인터럽트 핸들러의 bottom half에서 호출되는데, 드라이버가 패킷을 네트워크 스택으로 netif_rx() 또는 netif_receive_skb()를 호출하며 보내게 된다. 이러한 호출은 get_rps_cpu()를 호출하는데, 패킷을 처리할 큐를 선택하게 된다.

RPS의 첫 단계는 flow의 hash를 계산하는 것이다. 하드웨어가 제공하거나 스택에서 계산될 수 있으며, 유능한 하드웨어는 패킷을 위한 수신 디스크립터 속의 해시를 넘겨줄 수 있다. 이는 RSS의 그것과 같다. 또한 해당 Rx큐의 CPU 리스트 크기의 나머지 연산을 통해 할당할 CPU를 계산할 것이다.

인터럽트의 bottom half가 완료되었다면, 프로세서간 인터럽트(IPI)는 enqueue된 어느 CPU에게나 보내질 것이다.

이를 설정하는 방법은 rps_cpus에서 해당하는 cpu를 bitmap형식으로 활성화하면 되는데, 16진수로 작성하면 된다.

[https://netmarble.engineering/improve-network-performance-at-k8s-pod-using-rps-setting/](https://netmarble.engineering/improve-network-performance-at-k8s-pod-using-rps-setting/)

권장 설정은 싱글 큐의 경우 같은 메모리 영역 CPU로 구성해야 할 것이다. 또한 이미 많은 인터럽트가 일어나는 CPU는 리스트에서 제외하는 것이 더 좋다. 멀티큐의 경우, 만약 RSS가 설정되어 있다면 RPS는 필요하지 않을 것이다. 만약 하드웨어 큐가 CPU 수보다 적다면, 그 때 이득이 발생하는데, 똑같이 같은 메모리 영역의 CPU 들로 구성해 주면 된다.

이 때, 만약 특정 flow가 너무 비대해지면 CPU 부하가 불균형이 된다. 극단적인 경우에는 이 하나의 flow가 트래픽을 나타내게 될 것이다. 만약 기만적인 비대한 flow가 들어온다면, 다른 flow에 할당 될 자원까지 잡아먹게 되므로, 이는 DoS 공격이라고 볼 수 있다.

따라서 Flow Limit이 걸릴 수 있는데, 이는 RPS의 옵션이며, RPS or RFS가 특정 CPU가 과부하가 오기 시작할 때에, 앞선 large flow의 잔여 패킷들 보다 뒤의 small flow의 패킷을 좀 더 우선순위에 두게 한다. 또한, flow의 패킷 갯수가 특정 비율을 넘어선다면(기본적으로는 절반) 해당 Flow의 나머지 패킷들은 드랍하게 된다.

Per-flow rate은 패킷을 해싱할 때마다 해시테이블 버킷에 계산되고 하나씩 증가한다. 버킷은 CPU 수보다 훨씬 많게 잡을 수 있으므로, large flow에 대해 잘 식별할 수 있고, small flow에 대한 오탐율이 작다. 기본은 4096 버킷이다.

권장되는 설정은 많은 동시성 연결이 있을 때, 하나의 flow가 50% 이상의 CPU 자원을 사용할 때 활성화하면 효율적이다.

---

### ⇒ 즉, RSS는 Rx큐에 패킷을 골고루 분산하여 처리할 CPU를 선택하는 것이고, RPS는 각 CPU가 처리할 큐를 선택하게 함으로써 (하나의 큐가 여러 CPU) 더욱 분산처리를 할 수 있는 것이다.

```Plain
RSS (Receive Side Scaling) - 들어온 패킷에 대해 큐를 선택 /해싱을 이용-> 분산
| 패킷 1 | ----> | 큐 1 |
| 패킷 2 | ----> | 큐 2 |
| 패킷 3 | ----> | 큐 3 |
| 패킷 4 | ----> | 큐 4 |

RPS (Receive Packet Steering) - CPU가 처리할 큐를 선택 / 해싱을 이용-> 분산
| CPU 1 | ----> | 큐 1 |
| CPU 2 | ----> | 큐 1 |
| CPU 3 | ----> | 큐 2 |
| CPU 4 | ----> | 큐 3 |
```

---

### RFS: Receive Flow Steering

RPS는 단독으로 해시에 기반하여 패킷을 분산시키므로, 일반적으로는 좋은 부하 분산을 제공한다. 그러나 application locality는 고려하지 않았다.

이는 RFS를 통해 달성할 수 있다.

RFS는 실제 패킷데이터를 필요로 하는 CPU에 해당 패킷을 처리할 수 있도록 CPU를 선택, 분산하여 cache miss를 줄이는(캐시 적중률을 높이는) 목표를 가지고 있다.

실질적으로 RFS의 메커니즘은 특정 CPU에 패킷을 넣고, 해당 CPU를 깨우는 RPS 메커니즘과 같다.

RFS에서는 그 해시값을 사용하되, 바로 큐에 집어넣지 않고 flow lookup table에 집어넣게 된다. 이 테이블은 flow와 CPU간의 mapping을 기록한다.

따라서 패킷들을 flow에 기반하여 해싱을 통해 나누고, 해당 flow 별로 처리하는 CPU를 할당하는 것이다. 만약 valid한 CPU가 아니라면, 해당 패킷의 entry는 일반적인 RPS를 통해 처리되게 된다.

rps_sock_flow_table은 global flow table인데, 각각의 flow에 알맞은 CPU를 포함하고 있다. 이때, 알맞은 CPU란 userspace에서 해당 flow를 최근에 처리한 CPU를 의미한다. 업데이트는 전송을 위해 해당 함수를 호출할 때(inet_recvmsg(), inet_sendmsg(), tcp_splice_read()) 일어나게 된다.

이 때 스케줄러가 쓰레드를 새로운 CPU로 옮기게 된다면, old CPU에 남아있는 receive packet들로 인해 out of order가 발생할 수 있다. 따라서 rps_dev_flow_table을 통해 해결하려고 한다.

rps_dev_flow_table은 각각의 장치의 하드웨어 Rx큐가 entry인 테이블이다. 각각의 entry는 CPU index와 counter를 저장하고, 이때 CPU index는 해당 패킷의 앞으로 커널 처리를 위해 할당된 큐의 가장 최근 CPU이다. 이상적인 상황은 위의 두 테이블에서 CPU index가 같은 것이나, 만약 scheduler에 의해 userspace thread가 이동하였고, 여전히 old CPU에서 커널 패킷 처리가 이루어진다면 두 테이블이 다를 것이다.

counter에는 처리 중인 current CPU의 큐의 길이의 최신값을 저장한다.

따라서, 적절한 CPU를 선택하는 것은 위의 두 개의 테이블을 비교함으로써 결정되는데, 만약 둘이 같다면 해당 CPU에 집어 넣으면 되나, 다르면 특정 조건 하에 current CPU가 desired CPU로 업데이트 되게 된다.

- 해당 조건
    1. current CPU의 헤드 카운터가 테일 카운터보다 크거나 같다 → 이는 모든 큐를 처리하였음을 의미함.
    2. current CPU id가 invalid한 경우
    3. current CPU가 offline인 경우

권장 설정은, rps_sock_flow_entries와 rps_flow_cnt는 모두 RFS를 enabled 하기 전에 설정되어있어야 하며, 모두 2의 거듭제곱 꼴로 반올림된다. 권장되는 flow count는 예상되는 활성화된 연결의 수이며(any given time), 열려있는 연결의 수보다 현저하게 적어야 한다. 주로 2^15이 보통의 서버에 적합하다고 찾았다고 한다.

만약 싱글 큐 장치라면, rps_flow_cnt는 rps_sock_flow_entries와 같으면 되고, 멀티 큐라면, 각 큐의 rps_flow_cnt값은 rps_sock_flow_entries/N이 될것이다. 여기서 N은 큐의 갯수이다.

### aRFS: Accelerated RFS

이것과 RFS와의 관계는 RSS와 RPS의 그것과 같다.

즉, RFS를 하드웨어적으로 구현한 것으로, a hardware-accelerated load balancing mechanism이다. 이때 soft stat를 사용하며, 해당 패킷을 사용하는 application의 CPU를 찾아간다.

이때, NIC의 하드웨어에서 직접 처리되므로, 소프트웨어적으로 처리되는(CPU 자원을 사용하는) RFS에 비해 당연히 빨라야 한다.

이를 활성화 하기 위해서는 ndo_rx_flow_steer driver function을 부르면 되는데, 이는 적절한 하드웨어 큐와 네트워킹 스택간에 통신하기 위해서이다.

이 때 만들어지는 hardware queue는 CPU가 기록한 rps_dev_flow_table로부터 파생되었으며, 네트워크 스택은 CPU에게 NIC 드라이버가 유지하고 있는 hardware queue map에 정보를 제공하게끔 한다.

권장 설정 → RFS를 사용하고 싶을 때, 그리고 하드웨어가 지원한다면 사용하면 된다.

- [ ] hardware queue는 NIC 안에 존재하는 물리적인 버퍼인 것인가? 아니면 어떻게 구현되어 있는지 궁금하다. ↔ Device queue?

### XPS: Transmit Packet Steering

[[Xps - transmit packet Steering]]

XPS란 멀티큐 장치에서 패킷 전송을 위해 전송 큐를 지능적으로 선택하는 메커니즘이다. 이것은 두가지 종류의 매핑이 있는데, CPU를 하드웨어 큐에 매핑하는 것과, receive 큐를 하드웨어 전송 큐에 매핑하는 것이다.

1. **XPS using CPUs map**

CPU를 하드웨어 큐에 매핑하는 것의 목적은 주로 큐를 배타적으로 CPU subset에 할당하기 위해서이다. 이 subset은 해당 subset 내에서 특정 큐들에 있는 특정한 전송이 모두 처리되는 CPU들의 집합이다.

이 방법은 두 가지 장점이 있다. 첫 번째로, CPU간 같은 큐에 경쟁하는 것이 줄어들어 device queue lock이 현저하게 줄어든다.

두 번째로, 전송이 완료될 때의 cache miss가 줄어든다. 특히 sk_buff 구조를 들고 있는 라인의 데이터 cache miss rate이다.

2. **XPS using receive queues map**

