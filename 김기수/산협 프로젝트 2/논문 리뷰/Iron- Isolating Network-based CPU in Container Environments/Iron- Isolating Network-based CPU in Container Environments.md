### Abstract

컨테이너가 사용되는 환경에서 VM 환경과는 다르게 같은 OS 밑에 있어 각각 독립적이기가 더 어렵다.

대부분 CPU, memory, disk, 네트워크 대역폭의 isolate에 집중하였지만, 네트워크 트래픽을 처리하는데 걸리는 시간으로 인해 islate에 미치는 영향에는 관심이 없었다.

이논문에서는 컨테이너 기반 환경에서의 네트워크 스택과 관련 된 성능적인 오버헤드로 인한 격리의 붕괴에 대하여 보여주고자 한다.

특히, 컨테이너가 큰 용량의 네트워크 트래픽을 가지고 있을 때, 호스트의 성능을 많이 잡아 먹는다.

따라서 본 논문에서는 Iron 이라는 scheme을 소개한다.

컨테이너를 대신하여 네트워크 스택에서의 시간을 소비하는 정도를 설명하고(측정하고)

이러한 측정이 연관 된 컨테이너들에게 불리한 영향을 주지 않는 다는 것을 보장한다.

  

### Introduction

오늘날 많은 컨테이너들이 가상환경 안에서 작동하고 있다. 특히 serverless 컴퓨팅 플랫폼은 이러한 컨테이너에 의존하고 있다.

컨테이너는 하나의 OS 아래서 작동하므로, OS의 자원 배분 및 독립 능력이 매우 중요하다.

최근 control groups는 리눅스에서 자원 isolation을 allocating, metering, enforcing을 통해 커널에서 구현하였다.

최근의 연구에서는 오늘날의 컴퓨팅 작업은 서로 매우 다르고, 미리 예측한 버킷 할당에도 잘 맞지 않는다.

그러므로, 개발자들은 세분화하여 컨테이너 자원을 할당할 수 있어야 할 것이다.

그러나 이를 효율적으로 하기 위해서는 컨테이너의 준비된 자원이 반드시 사용가능해야 할 것이다.

과하게 준비되었거나, 비효율적인 자원 배분 및 격리는 자원의 사용가능성이 타협 될 것이고, 이는 레이턴시가 중요한 application에서 성능 이슈를 겪게 될 것이다.

서버리스 컴퓨팅에서 요금은 시간 당이므로, 이는 사용자에게 실제보다 과한 요금을 청구하게 될 것이다.

클라우드 또한 고밀도의 컨테이너를 켜기 위해 효율적인 컨테이너 조절을 위해서 자원 격리에 의지한다.

그러나 만약 서버 자원 사용의 제한선이 강제적이지 않다면 클라우드 제공자는 trade-off를 마주하게 될 것이다.

각각의 서버에 부족한 자원을 제공하여 잠재적인 수익 창출에 대하여 실패하거나, 느슨한 격리으로 인해 고객들의 클라우드에 퍼포먼스에 안좋은 영향을 끼치게 된다.

이 논문에서는 네트워크 트래픽을 다룰 때 cgroup에서가 할당하는것보다 더 효율적으로 breaking isolation을 하여 더 CPU를 효율적으로 활용할 수 있다는 것을 보여준다.

모던 커널들은 대부분 인터럽트에 의해 트래픽을 처리하는데, 이러한 인터럽트 처리 과정에 드는 시간은 대부분 컨테이너에 부과되지 않는다.

정확한 시간을 측정해야지만 좀더 구체적인 자원 isolation을 제공할 수 있다.

사실 높은 네트워크 트래픽의 컨테이너는 colocated 컨테이너에게 6배 이상 느려지게 할 수 있다.

이러한 오버헤드는 상당히 큰데, 커널이 대부분의 Network processing을 하고 있기 떄문이다.

모던 데이터센터들은 매우 큰 대역폭을 가졌는데, 이러한 곳에서도 network processing이 큰 오버헤드로 작용하고 있음을 많은 연구들이 보여주고 있다. 따라서 연구자들은 CPU를 isolate 시키기 위해 많은 연구를 하였다.

==그러나 이러한 연구들은 오늘날의 컨테이너 base 환경에서는 잘 맞지 않으며, 인터럽트에 의한 오버헤드가 적은 네트워크 서브시스템에서는 비효율적이다.==

**Iron : Isolating Resource Overhead from Networking**

네트워크 트래픽을 처리하는데 CPU의 사용량을 모니터링하고, 측정하며, 이를 해당 컨테이너에 부과하고, 제한한다.

Iron은 커널 명령어를 통해 구현되었는데, 각각의 패킷을 처리하는데 드는 비용을 얻으면서 동시에 인터럽트 핸들링에 영향을 가지 않게 섬세하게 짜여져 있다.

각 비용을 부과하는것만으로는 isolation을 제공하기 힘든데, 각각의 할당을 다 사용하고 난뒤 컨테이너가 받은 패킷을 처리하는 것은 isolation을 깰 수 있기 때문이다.

따라서 Iron은 하드웨어 기반의 할당이 해제된 컨테이너들의 패킷을 드랍하는 목표를 가지고 있다.

컨테이너화된 시스템에서 엄격한 격리를 제공하는 것은 꽤 많은 이유로 힘든데, 우선 컨테이너의 트래픽이 서버 OS의 네트워크 스택 전체에 골고루 분포와어있기 떄문에 정확한 사용율을 측정하는것은 다양한 패킷 타입에 따라 다양하게 측정될 것이다.

주어진 해결책은 가벼운 연산이여야 할텐데, 패킷당 계산이므로 높은 오버헤드를 일으키는 경향이 있고, 코어 사이에서 state를 유지하는 것이 비효율적인 locking을 일으키기 때문이다.

패킷을 받는 것에 있어서 간섭을 제한하는 것은 힘든데, 관리자가 전체 트래픽을 컨트롤 할 수 없을 수도 있기 때문이다.

Iron은 다음과 같은 기여를 할 수 있다.

- 사례연구에서 보여주듯이 네트워크 트래픽의 프로세싱의 연산량이 중요할 수 있다는 것을 보여준다. 이를 고려하지 않았을 때, 몇몇 연산에서 약 6배 느려졌다.
- 엄격한 격리를 제공한다. 리눅스 cgroup scheduler와 통합하여 컨테이너들의 그것을 정확하게 측정하고, 네트워크 기반의 처리를 보증한다. 또한 부작용을 제한하고, 할당된 자원을 다 써버린 다른 컨테이너의 오버헤드를 최소화하기 위해 기발한 패킷 드랍 메커니즘을 제공한다.
- “MapReduce”와 같은 작업의 경우 trace-driven network loads를 이용하면 50%정도 느려지는 경험을 할 수 있었고, 연산기반의 작업의 경우 조절된 셋팅에서 6배 느려지는 것을 볼 수 있다. Iron은 효율적으로 격리를 제공하여 5%미만으로 느려지게 한다.

### Background and Motivation

먼저 간섭 문제에 대하여 이야기할 것이다. 어떻게 컨테이너의 네트워크 트래픽이 다른 컨테이너에 할당 된 CPU를 간섭할 수 있는지 말이다. 이후 이전의 해결책과 Iron을 두고 실제 어떻게 영향을 미치는지 조사하였다.

### 2.1 Network traffic breaks isolation

간섭문제는 리눅스 커널이 네트워크 트래픽의 인터럽트에 대한 정확한 시간측정을 제공하지 않기 때문에 발생하게 된다.

**Linux container scheduling** - Cgroups는 컨테이너에 할당하는 CPU를 제한하는데, 특정 기간동안 얼마나 컨테이너가 오래 실행하는지를 정의하여 사용한다. 고수준에서, 스케줄러는 runtime이라는 변수를 가지고 있는데 그것은 현재 컨테이너가 얼마나 오래 실행되었는지를 저장한다. 만약 quota에 이 값이 도달하면, 컨테이너가 쓰로틀링이 발생한다. 만약 그 기간이 끝난다면, runtime 변수는 quota로 재충전 될 것이다.

**Linux interrupt handling** - 리눅스는 인터럽트를 두 파트로 나누어 제공함으로써 인터럽트 오버헤드를 줄이는데, top half(하드웨어 인터럽트)와 bottom half(소프트웨어 인터럽트)이다. 하드웨어 인터럽트는 언제나 일어날 수 있다. 따라서 top half는 인터럽트를 서비스하기 위해 필요한 중요한 작동만 하여 가볍고 빠르도록 디자인 되어있다. top half는 excute를 위하여 bottom half를 스케쥴 하게 된다. bottom half는 커널이나 I/O장치의 성능에 영향 미치지 않으면서, 딜레이 될 수 있는 작업들을 실행하는 책임을 가지고 있다. 리눅스에서 네트워크는 “softirqs”(software interrupt의 일종)를 bottom half로써 채택한다. 소프트웨어 인터럽트의 경우 하드웨어 인터럽트가 끝날 때 혹은 언제든 커널이 다시 softirq를 처리할 때 실행할 수 있다. 또한 이는 process context에서 실행된다. 이말인 즉슨, 해당 프로세스가 실행되지 않을 것 같아도 softirq를 처리하기 위해 스케줄이 되어야 한다는 것이다. 따라서 특정 컨테이너의 CPU time에 다른 컨테이너의 트래픽을 처리하게 되는데, 이러한 이유로 인해 CPU 격리가 제대로 이루어지지 않는다.

물론 커널도 이러한 softirq handling을 줄이기 위해 노력하는데, 고정된 시간에 하거나, 패킷의 처리량을 정해두기도 한다. 만약 처리량을 초과하였을 때는 당장 실행을 멈추고 ksoftirqd 를 스케쥴 하여 실행되게 한다. 이는 커널 쓰레드인데, process context가 아니다. 따라서 process context를 소모하지 않고 softirq를 처리할 수 있게 된다. 여기에 프로세서당 하나의 ksoftirqd가 있는데, 이러한 시간 소모는 어떠한 컨테이너에게도 부과되지 않기 때문이다. 이것 또한 격리를 방해하는데, 다른 컨테이너들을 스케쥴링하는데 가용한 시간이 제한되거나, 컨테이너가 quota를 다 소모하여 추가적인 자원을 얻게 하는 것을 막지 못하기 때문이다.

**kernel packet processing** - 보통은 커널에서 전송이 완료되므로 적절한 컨테이너에 시간이 부과되지만, 두가지 격리를 무너뜨리는 케이스가 있다.

- NIC가 패킷 전송이 끝나면, 패킷 리소스를 free 할당 해제하기 위해 인터럽트를 스케쥴하게 된다. 이러한 작업은 softirq context에서 이루어지므로 트래픽을 보내지 않은 컨테이너에 부과될 여지가 있다.
- Buffering the Packet에서, TCP 혼잡제어나 traffic shaping(통신량 제어)에서 일어난다. 이때는 패킷이 소켓에서 버퍼로 올라오는데, process context에서 이루어진다. 문제점은, 버퍼에 패킷을 올리고 나면 커널 시스템 콜이 종료된다. 따라서 Software interrupts가 버퍼에서 패킷을 삭제하는것과 네트워크 스택으로 내려가는 책임을 가지고 있다. 이는 앞서 ksoftirqd로 다뤄진다는 것과 잘못된 컨테이너에 부과되는 문제점을 그대로 가지고 있다.

패킷 송신자 측보다 패킷 수신자 측이 더 큰 softirq overhead에 빠지게 된다. 패킷을 받을 때, 패킷은 softirq context 상에서 driver’s ring buffer에서 application socket으로 이동하게 된다. 이 때, 네트워크 스택을 지나야 하므로, 많은 처리 과정을 거쳐야 할 것이다. protocol handler(IP, TCP), NFVs, NIC offloads(GRO) 등이 있으며, 송신자 측도 마찬가지이다. (TCP ACK, ICMP message)

요약하자면 전체 수신 처리 과정은 softirq context상에서 처리되며, 이러한 실질적인 사용 시간이 해당 컨테이너에 부과되지 않을 것이다.

  

### 2.2 Putting Iron in context

앞서서의 작업들은 간섭문제를 줄일 수 가 있는데, 컨테이너 자원 소모에 대한 부과를 새로운 추상화를 디자인하거나, OS를 다시 디자인하여 달성할 수 있다. 아래 내용은 Iron과 비교한 것이다.

**System container abstraction -** Banga의 [논문](https://cs.uwaterloo.ca/~brecht/courses/856-Internet-Server-Performance-2006/readings-new/lrp-1996.pdf)은 LRP를 통해 시스템 리소스를 잡고 부여하는 추상화된 **resource container**를 제안하였다. 프로세스가 스케쥴 되었을 때, 수신자 측은 커널에서 느긋하게 프로토콜 프로세싱을 수행한다. 이를 통해 정확한 프로세스에 걸리는 시간이 부과되었다. 수신 처리와 프로토콜 처리를 분리하여, application이 해당 데이터를 요청할 때 프로세스를 수행하게 하여 정확한 비용을 청구한다.

그러나 이러한 해결방법은 TCP에는 적합하지 않은데, 기껏해야 하나의 윈도우만이 연속되는 시스템 콜들 사이에서 소모될 수 있기 때문이다. 따라서 LRP는 소켓 당 쓰레드를 도입하였고, 각각의 쓰레드는 receiving pross와 연관되어있다. 이 때 비동기 protocol processing 과 서로 비동기식으로 처리되게 하였고, CPU 소모량이 적합하게 부과되었다.

LRP는 부과 문제를 해결하였으나 여러 이슈들이 고려되어야 한다. 먼저 이름에서 그러하듯이 오로지 수신자 측만 커버가 가능하고, 전체 트래픽에 대하여는 해결하지 못하였다. 두 번쨰로, LRP는 비동기 TCP processing을 위하여 소켓당 쓰레드를 필요로 했다. 이러한 추가적인 쓰레드를 유지하는것은 추가적인 context switching을 요구하고, 이는 큰 flow들에 있어서 많은 오버헤드를 일으킬 수 있다. 세 번째로, 스케줄러는 프로토콜 프로세싱 중인 쓰레들을 알고 있어야하고 잠재적으로 우선순위를 두어야 할것이다. 아니라면 소켓 쓰레드가 스케쥴 될 때까지 TCP는 레이턴시가 증가하고 심하면 drop 될 수도 있다.

---

Iron은 명시적으로 위의 문제점들을 다룰 수 있다. 첫 번쨰로, Iron은 정확하게 전송에 걸리는 시간을 부과한다. 두 번째로, Iron은 원활하게 리눅스의 인터럽트 과정과 통합되어 효율성과 반응성을 유지할 수 있다. 리눅스에서 모든 코어의 트래픽은 그 코어의 softirq handler에 의해 처리된다. 공유 된 것에서 인터럽트를 처리하는 방법은 쓰레드 당 방법보다 context switching을 최소화하여 효율성을 유지한다. 추가적으로 process context에서 hardware interrupt를 제공하여, 프로토콜 처리가 대응되어 처리된다. 그러나 리눅스의 디자인은 직접적으로 간섭 문제에 직면한다. 그러므로 인터럽트 처리가 shared 방법으로 동작할 때도 네트워크 프로세싱을 위한 시간 측정이 정확하다는 것을 보여줄 것이다.

**Redesigning the OS** - OS를 수정하는 방법은 네트워크 프로토콜 처리를 커널이 아닌 application의 라이브러리로 옮겨서 해결하려 하였다. 이 계획에서는 패킷의 demultiplexing이 네트워크 스택에 낮은 레벨에서 수행되게 된다. 주로 NIC가 직접 어플리케이션과 공유 중인 버퍼로 복사하는 방법으로 수행된다. 따라서 직접적으로 어플리케이션이 직접 버퍼에서 패킷을 끌어와 처리하므로, network-based processing이 올바르게 부과된다.

그러나 Library OSes는 여러 실질적인 우려들이 있다. 첫 번째로, 이러한 작업은 LRP와 마찬가지로 쓰레드화 된 프로토콜 프로세싱이라는 문제가 있다. 두 번째로, 커널으로부터 명시적으로 네트워크 프로세싱이 사라지는 것은 관리가 어려워진다는 점이다. 또한 다양한 사람들이 동시에 차지하며 사용하는 데이터 센터의 경우 서버 호스트는 사용율 제한, 가상 네트워크, 요금 부과, 트래픽 관리, 상태 모니터링, 보안등의 서비스를 제공하여야 한다. 그런데 Library OS를 사용하면, admin-defined network processing은 NIC 혹은 user-level software에서 이루어져야 할 것이다. 따라서 관리자가 네트워크를 정책이나 기능을 추가하여 관리하는 것이 사실상 불가능하므로 현실적이지 못한 방법일 것이다. NIC-based 기술은 더 많은 비용이 들며, 더 적은수의 flow와 service를 제공하게 될 것이다. 앞으로 미래에는 NIC가 유연해지는 만큼, network management는 admin-controlled software와 hardware의 조합으로 구성 될 것이다.

---

Iron은 software-based network processing에서 추적하고 강제하는것에 도움을 줄 수 있다. 결과적으로 Library OS를 상용 클라우드 서비스에 제공하는것은 클라우드 제공업자와 소비자 모두에게 큰 진입장벽이 되게 될 것이다.

### 2.3 Impact of network traffic

여기서는 UDP와 TCP의 네트워크 작업이 격리된 컨테이너 환경에서 어떤 영향을 미치는지 측정하는 실험을 보여줄 것이다.

victim은 컨테이너당 하나이며, CPU를 측정하는 sysbench를 돌리게 된다. 나머지는 interferes로, sysbench를 똑같이 돌리게 된다. 아무것도 안돌리는 것이 기본 케이스이며, 실험군에서는 inteferes가 가능한 많은 패킷을 보내는 간단한 네트워크 흐름 application을 실행할 것이다. 여기서 penalty factor는 sysbench와 경쟁하는 것과, traffic과 경쟁하는 것의 비율로 구할 것이다. 만약 페널티 팩터가 1보다 크다는 것은 격리가 깨졌다는 의미일 것이다. 왜냐면, 트래픽이 victim에게 안좋은 영향을 미쳤다는 뜻이기 때문이다.

penalty factor = traffic / sysbench

수신자 테스트의 경우 하나의 코어에 할당하며, NIC interrupt도 같은 코어에서 이루어진다. 여기서는 interfere가 simple receiver를 실행하는데, 멀티쓰레드의 센더가 1400바이트의 패킷을 수신하는 컨테이너간에 골고루 보낸다.

### Sender Side Penalty Factor

**UDP**

첫 번째로, 만약 TC queue rate에 제한이 없다면 패널티 팩터가 없다는 것이다. → 어플리케이션이 요구하는 대역폭보다 전체 대역폭이 더 크기 때문이다.

두 번째로, 대역폭 제한이 클 수록, 패널티 팩터가 점점커진다는 것이다. → 이는 대역폭이 전체 요구치보다 적었음을 나타낸다. 따라서 패킷들이 queued 되었음을 나타낸다.

이때, Softirq가 interferes를 victim의 프로세싱 time동안 다루게 되기 때문에, 높은 패널티 팩터로 이어진다.

세 번째로, HTB는 1-3Gbps에서 더 큰 패널티를 체감할 수 있었는데, 4Gbps이상 일때는 송신자가 CPU 한계이므로 더 큰 트래픽을 만들 수 없어 더 이상 대역폭이 패널티팩터의 이슈가 아니게 된다.

이 때, ksoftirqd의 CPU usage도 확인해보면 대략적으로 패널티 팩터와 대응된다는 것을 알 수있다. 이는 컨테이너의 시간 차감에 들어가진 않지만, victim의 작업 완료시간이 늦어지는 결과를 가져온다.

**TCP**

tcp는 core당 flow의 수가 달라짐에 따라 그 결과가 달라진다. 통상적으로 TCP가 UDP보다 오버헤드가 더 크다. 이는 ACK를 보내는 것과, TCP 레이어에 패킷을 버퍼링하는 것 때문이다. 이 것들은 softirqs에 의해 이루진다. 따라서, 대역폭의 제한이 없더라도 TCP의 오버헤드가 UDP보다 더 크게 된다.

두 번째로, flow가 많아질수록 오버헤드가 커진다는 점인데, 이는 두 가지 이유로 인한 것이다. 첫 번째로, ACK의 수가 늘어날 것이고, 또한 많은 flow일 수록 프로토콜 프로세싱이 더 많이 존재할 것이다. 두 번째로, 싱글 flow는 대역폭 제한에 적용될 수 있지만, multiple flow는 다양한 트래픽 패턴으로 인해 rate limiter에 queuing이 증가할 것이다.

### Receiver Side Penalty Factor

**UDP**

여기서는 $i$﻿개의 receiver가 있다면 10-$i$﻿개의 컨테이너가 sysbench를 돌리게 된다. 이 때 코어당 수신 속도가 x-축이 되는데, 이는 드랍으로 인해 전송속도와 다를 수도 있다.

첫 번째로, 패널티 팩터는 수신쪽이 송신쪽보다 크다는 것이다. 패킷이 softirq context 속에서 전체 네트워크 스택을 지나야하기 떄문에 더 크다고 분석할 수 있다.

다음으로, receiver가 증가할 수록, 처리하는 연산량을 더 확보할 수 있으므로, 성능이 증가하고, 패널티 팩터가 1인 구간이 늘어나게 된다.

**TCP**

UDP와는 다르게 가능한 많은 트래픽을 전송할 수 있게 셋팅하였다. TCP는 congestion control이나 flow control에 의해 알아서 그 대역폭에 적응할 것이다. 앞서서와 마찬가지로, 수신 속도가 늘어날 수록 패널티 팩터도 증가하였다.

결과적으로 이것들은 격리 기술이 반드시 ksoftirq와 process context에서의 softirq에 의한 시간 소모를 잡아내야한다는 것이다.

### Design

먼저, 아이언은 softirq context에서 패킷을 프로세싱하는데 걸리는 시간을 측정한다. 이러한 패킷 소요값을 얻고 나면, 아이언은 리눅스 스케쥴러와 통합되어 컨테이너에게 softirq processing을 위해 컨테이너에 charge 혹은 credit을 하게 된다. 만약 컨테이너의 런타임을 다 썼다면, 아이언은 컨테이너에 성능제한을 통해 강력한 격리를 강제하고, 들어오는 패킷을 하드웨어 기반의 방법으로 드랍시킨다.

### 3.1 Accounting

**Receiver-based accounting**

리눅스에서는 서로 연결된 일련의 함수 호출을 통해 패킷이 네트워크 스택을 돌아다니게 된다. 따라서, call stack의 낮은 곳에 있는 함수에서 시간 측정을 통해 패킷이 처리되는 시간을 구할 수 있을 것이다.

아이언은 `netif_receive_skb`라는 함수를 패킷당 비용을 계산하기 위해 사용한다. 왜냐하면, 이것은 프로토콜에 상관없이 드라이버의 바깥에서 각각의 패킷을 다루는 가장 첫 번째 함수이기 때문이다.

이 때, 단순히 시간 차이를 구하는 것은 정확하지 않을 수가 있다. 커널이 선제적으로 작업할 수도 있고, 함수가 call tree에서 언제든지 인터럽트를 처리하러 갈 수있기 때문이다. 오로지 packet processing에 쓰인 시간만을 측정하기 위해, 아이언은 scheduler data를 이용한다. 스케쥴러는 `cumtime`과 `swaptime`을 항상 가지고 있기 때문에, 현재시간 `now`와 연관지으면, 시작시간과 끝 시간을 계산할 수 있다.

$Time = cumtime + (now - swaptime)$

패킷당 비용을 제쳐두고도, 여전히 트래픽을 처리하는 것과 관련된 고정비용이 있는데, hardware interrupts(do_IRQ)를 처리하는 함수에 진입하는 것과, softirq를 처리하는 것, 그리고 skb garbage collection등이 있다. 아이언에서는 이러한 오버헤드들을 한데 묶어서 패킷 비용에 할당한다.

리눅스에서는 6가지 종류의 softirq가 softirq handler(do_softirq)에 의해 처리되는데, HI, TX, RX, TIMER, SCSI, TASKLET이 있다. 각각의 인터럽트에 대하여 전체 do_IRQ 비용을 얻을 수 있다. 이를 $H$﻿라 하자. 그리고 각각의 비용을 $S_{HI}, S_{TX}$﻿ 등등으로 나타낸다. 이 때, 소프트웨어 인터럽트는 하드웨어 인터럽트 이후에 처리되므로, $H > \varSigma_iS_i$﻿ 임을 알 수 있다.

이 때, 인터럽트를 처리하는 것과 관련된 오버헤드는 다음과 같이 정의 한다. $O = H-\varSigma_iS_i$﻿ 그리고 수신자 오버헤드의 공평한 공유 값은 $O_{RX} = O \frac{S_{RX}}{\varSigma_iS_i}$﻿로 나타낼 것이다. 마지막으로 $O_{RX}$﻿의 값은 각각의 패킷에 추가된 고정된 비용을 얻기 위해 부른 do_softirq에 의해 처리된 패킷마다 공평하게 나누어져 있다.

이러한 방법은 TCP에서도 올바르게 작동하는데, 중간에 ACK를 보내기 위해 해당 함수를 호출하는 것도 account에 포함되게 된다. 또한, 버퍼링도 정확하게 다루어진다. 재전송이 이루어질 때, seq num이 모두 올바르게 채워지면 소켓으로 패킷이 밀려올라가는데, 이 때 올바르게 전송된 패킷도 같이 비용이 계산되기 때문이다.

**Sender-based accounting**

패킷을 보낼 때, 커널은 NIC queue에 lock을 얻어야 한다. 소켓당 lock을 얻는 것은 높은 오버헤드를 일으키므로, 패킷은 주로 리눅스에서는 전송을 위해 묶여서 처리된다. 따라서 아이언은 이러한 묶음을 전송하는 비용을 측정하고, 각각의 패킷에 공평한 몫의 비용을 분배하게 된다. `do_softirq`함수는 `net_tx_action`호출하게 되는데, 전송 softirqs를 처리하기 위해서이다. 그러면 `net_tx_action` 은 패킷을 찾기 위해 qdisc 레이어를 호출하게 된다.

qdisc는 간단한 FIFO 큐 메커니즘임. 리눅스에서 Traffic control을 위해 사용 됨.

Multiple qdisc queue는 dequeue 될 수 있으며, 각각의 큐는 multiple packet을 return할 수 있다.

결과적으로, skb의 링크드 리스트가 생성되고, NIC로 전달된다.

receiver와 비슷하게, `net_tx_action` 이 송신 batch에 대한 시작과 끝 시간을 얻는다. $O_{TX}$﻿는 전송의 고정된 오버헤드들을 나눠줌으로써 얻을 수 있다. 오버헤드들은 코어마다 계산되는데, HTB는 작업을 보존하므로 패킷이 큐에 들어간 코어와 다른 코어에서 패킷을 큐에서 제거할 수 있기 때문이다.

**Container mapping and accounting data structures**

아이언은 각 컨테이너에 비용을 부과하기 위해 패킷이 어느 컨테이너 것인지 구분해야한다. 송신자는, skb가 qdisc 레이어에 들어갔을 때 그것의 cgroup과 연관되어있다. 수신자는, 아이언이 패킷을 소켓으로 복사할 때 채워지는 IP address에 대한 해시 테이블을 유지하게 된다.

아이언은 각각의 프로세스에서 코어당 처리 된 패킷 리스트와 그 비용을 저장했다가, 결국에는 cgroup별 글로벌 구조로 병합되게 된다.

![[images/Untitled 2.png|Untitled 2.png]]

cgroup당 가지는 구조는 gained라는 변수를 유지하는데, 이것은 만약 cgroup이 네트워크 처리를 위해 크레딧 (추가적인 할당시간을 부여)되어야하는지를 나타내는 변수이다.

### 3.2 Enforcement

격리는 측정한 비용 데이터를 리눅스의 CFS 스케줄러의 CPU 할당과 통합하여 이루어질 수 있다.

==CFS : 모든 프로세스가 똑같은 시간을 할당받는 스케줄링 기법==

**Scheduler integration**

CFS는 cgroup을 위한 CPU 할당을 하기 위해 하이브리드 방법을 사용하는데, 이는 local( i.e., per core)과 global state를 유지하는 것이다. 컨테이너들은 period동안 quota만큼의 시간동안만 실행 될 수 있도록 허락되어 있다. 스케줄러는 오버헤드를 최소화하기 위해 세세한 부분에서 local state를 업데이트하고, 큼직큼직한 부분에서 golbal state를 업데이트하게 된다. global level에서 runtime 변수는 period가 시작할 때, quota로 초기화 된다. 스케줄러는 slice만큼을 runtime에서 빼고, 그것을 local core에 할당하게 된다. runtime은 0이 되거나 period가 끝날 때까지 지속적으로 줄어들게 된다. local level에서, rt_remain 변수는 slice 시간간격에 할당되게 된다. 스케줄러는 rt_remain을 해당 cgroup이 CPU를 소모하며 일을 할 때 줄이게 된다. 만약 rt_remain이 0이 된다면, 스케줄러는 global pool에서 새로운 slice를 얻어오려고 시도할 것이다. 새로 얻으면 rt_remain은 초기화 되겠지만, 아니라면 해당 cgroup은 쓰로틀링에 걸릴 것이고, period가 끝날 때까지 아무것도 못한다.

아이언의 global scheduler는 Algorithm1과 같다. gained 변수는 컨테이너가 반드시 돌아와야하는 시간을 추적하고 있는데, 왜냐하면 다른 컨테이너의 softirq를 처리하였기 때문이다.

![[images/Untitled 1.png]]

아이언의 local algorithm은 Algorithm2와 같다. 스케줄러는 이 함수를 rt_remain $\leq$﻿ 0 이거나 적절한 lock을 얻었을 때 부른다. cpuusage 변수가 local accounting을 유지하기 위해 더해지게 된다. 양수 값은 컨테이너가 측정되지 않은 네트워크 처리 싸이클에 대한 비용을 부과해야 함을 나타내고, 음수인 경우 반대이다.

**Dropping excess pacekts**

스케줄러기반의 강제하는 것이 격리를 향상시키는 동안, 패킷은 여전히 드랍됨으로써 쓰로틀링이 걸린 컨테이너가 네트워크 기반의 처리를 할 수 없게 해야한다. 아이언은 명시적으로 송신자 측에서 패킷을 드랍하지 않는다. 왜냐하면 쓰로틀링이 걸린 컨테이너가 이미 더 이상 송신할 트래픽을 만들어낼 수 없기 때문이다. 그러나 컨테이너가 매우 큰 패킷을 보내면서, 적은 런타임이 남았을 경우 같은 극단적인 상황도 존재한다. 최근에는, 스케줄러가 이러한 비용은 다음 quota 리필 때 부과하게 된다. 아이언에서는 패킷 전송에 대한 비용을 계산하는 것을 미리 부과하는 계획을 짰고, 이는 충분히 성능에 영향을 미치지 않았다.

아이언은 오늘날의 아키텍쳐와 통합할 수 있는 하드웨어 기반의 패킷 드랍 알고리즘을 작성하였다. 오늘날, NIC는 들어오는 패킷을 다중 큐에 삽입한다. 각각의 큐는 특정 코어들에 할당 될 수 있는 인터럽트를 가지고 있다. 격리를 향상시키기 위해, 발전된 RFS를 통해 패킷들이 해당 컨테이너가 실행되고 있는 코어로 향하게 된다. (FlexNIC도 작동한다.) 수신 받았을 때, NIC는 DMA를 통해 패킷을 공유 메모리의 링 버퍼에 올려 놓는다. 그러면, NIC는 큐를 위해 IRQ를 생성하게 되는데, 이 것은 드라이버의 interrupt handler를 작동하게 한다. 현대의 시스템들은 NAPI로 네트워크 인터럽트들을 관리한다. 새로운 패킷을 받을 때, NAPI는 하드웨어 인터럽트를 끄고, OS에게 패킷들을 가져오게 polling method를 스케줄하도록 알리게 된다. 그러는 동안 추가적으로 수신받은 패킷들은 그저 NIC에 의해 링 버퍼에 쌓이게 된다. 커널이 NIC를 poll할 때, NIC는 미리 정한 값 안에서 가능한 최대한 많은 패킷을 링 버퍼에서 제거한다. 제거된 패킷의 숫자가 미리 정한 값보다 적다면, NAPI의 폴링이 끝나고, 인터럽트 기반의 수신이 다시 재개된다.

```Plain
|----------------------------------------------------| <- 링 버퍼
|===========================|                        | <- 버퍼에 쌓인 패킷들
|=====|=====|=====|=====|===  |                      | <- polling된 패킷들, budget은 5개의 패킷임. 맨 마지막에 budget보다 적은 패킷이 제거되었으므로, polling을 멈추고 interrupt 기반의 NAPI 재개 됨.
```

아이언의 하드웨어 기반 패킷 드랍 메커니즘은 다음과 같다. 첫 번째로, NIC가 컨테이너당 큐를 가지고 있다고 가정한다. 아이언은 NAPI 큐의 맵 구조체를 큐에서부터 그 것의 컨테이너까지 늘린다(task_group). 만약 스케줄러가 컨테이너에 쓰로틀링을 건다면, 그것은 task_group의 boolean을 수정하게 된다. default NAPI와는 다르게, 아이언은 쓰로틀링이 걸린 컨테이너의 큐에서 패킷을 풀링하지 않는다.

커널의 관점에서 보았을 때, 해당하는 큐는 풀링 리스트에서 제거되므로, 항상 다시 풀링되지 않는다.

==NIC 관점에서 보았을 때, 커널은 큐에서 풀링하지 않으므로, 커널은 계속 풀링 모드를 유지할 수 있고 하드웨어 인터럽트가 계속 꺼져있게 된다.==

만약 새로운 패킷이 도착했다면, 단순하게 링버퍼에 넣기만 하면 될 것이다.

이 기술은 효율적으로 수신 오버헤드를 줄일 수 있는데, 왜냐하면 커널은 쓰로틀링이 걸린 컨테이너를 위해 어떠한 인터럽트나 작업을 하지 않아도 되기 때문이다. 스케줄러가 컨테이너의 쓰로틀링을 해제 했을 때, NIC는 task_group의 boolean을 리셋하게 되고, 큐에 들어온 패킷을 처리하기 위한 softirq를 스케줄링할 것이다.

약간의 최적화를 통해 아이언은 컨테이너가 쓰로틀링이 걸리기 전에 패킷을 드랍할 수 있다. 그것은 만약 컨테이너가 많은 양의 트래픽을 받는 중이고, 컨테이너가 quota의 $T$﻿%를 사용하였다면, 패킷이 드랍될 수 있다. 이것은 컨테이너가 남은 할당된 시간을 수신하는 패킷의 흐름을 멈추는데 사용하는 것이 가능해진다.

하드웨어 기반의 패킷 드랍은 NIC당 큐가 많을 때 효율적이다. 심지어 NIC들이 점점 더 큐의 크기가 커지고 있다.

실제 환경에서는 컨테이너의 갯수와 큐의 갯수가 다를 수 있따. 아이언은 코어당 고정된 갯수의 큐를 할당할 수 있으며, 동적으로 문제가 있는 컨테이너를 그들의 큐에 매핑할 수 있다.

매우 많은 트래픽이 없는 컨테이너는 `__netif_receive_skb` 함수를 softirq call stack에 일찍이 추가하여 소프트웨어 기반의 드랍이 초래될 수 있다. 이러한 동적 할당은 비슷한 접근법인 NIC기반의 대역폭 제한을 확장하는 SENIC에서 영감을 받았다. 대신에 컨테이너는 미리 할당된 대역폭을 기반으로 큐에 매핑될 수 있다.