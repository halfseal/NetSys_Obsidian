### Abstract
오늘날의 고도로 통합되고 비범용적인 정적 패킷 프로세싱 파이프라인은 네트워크 스택들 속에 넓게 분포해있는데, 이는 최신 하드웨어의 성능을 100% 끌어낼 수 없었다.
이 논문에서는 `NetChannel`이라는 것을 제안한다. 비통합적인 네트워크 스택 아키텍쳐로, 테라비트 이더넷을 운용하는 μs-스케일의 application을 위한 것이다.
넷채널은 비-통합 아키텍쳐인데, 각 패킷 프로세싱 파이프라인 레이어마다 독립적으로 스케쥴링되고 스케일링이 되는 자원들이 할당된다.
리눅스 네트워크 스택에다가 end-to-end로 넷채널을 실현함으로써, 논문에서는 넷 채널이 새로운 운영 포인들이 가능케 함을 보여 주었다.
- single application thread가 수백Gb의 대역폭을 전부 사용함.
- application thread의 갯수에 독립적으로 여러 코어를 활용한 small message processing에 선형에 근접한 scalability가 가능함.
- 쓰루풋을 다 쓰는 application이 실행됨과 동시에 latency에 민감한 application의 isolation이 가능함.

### Introduction
오늘날 리눅스 네트워크 스택의 한계점으로 지목되는 가장 큰 점은 비효율적인 패킷 프로세싱 파이프라인이다. 이는 레이턴시에 민감한 것과 쓰루 풋을 다쓰는 application을 서로 분리할 수 없고, 그 구현이 복잡하고 유연하지 못하며 4계층이 비효율적이라는 단점을 가지고 있다.
이러한 문제점들에 대한 해결책으로 대부분 리눅스 네트워크 스택을 새롭게 구성하는 디자인을 중점적으로 개선방향이 나왔다.
그러나 이 논문에서는 리눅스 네트워크 스택의 문제점이 인터페이스, semantics, placement가 문제가 아닌 코어 아키텍쳐에 문제가 있다는 것을 시사한다. 특히, 리눅스가 탄생 된 이후로 리눅스 네트워크 스택은 application에 유연하지 못한 모두 일률적인 아키텍처로 디자인된 추상화된 파이프를 제공했다.
이러한 아키텍쳐는 아래와 같은 특징을 가진다.
- 특정 목적의 파이프 : 각 application은 데이터를 특정 파이프에 집어 넣고, 네트워크 스택은 이를 다른 파이프 끝에 전달하려고 시도한다.
- 치밀하게 통합되어있는 패킷 프로세싱 파이프라인 : 각각의 파이프는 각각 소유하고 있는 소켓에 할당되며, 각 소켓은 독립적인 4계층 동작을 하게 된다. 이는 외부적인 리소스, 혹은 다른 파이프를 고려하지 않는다.
- 정적인 파이프 : 패킷 프로세싱 파이프라인 전체가 파이프가 만들어질 때 동시에 결정된다. 따라서 외부 환경이 바뀌면서 이를 동적으로 사용할 수 없게 된다.
옛날에는 bottleneck이 이러한 아키텍처를 통해 적절하게 처리되었기 때문에 이러한 디자인은 초기 인터넷 환경에 적합하였다.
그러나 오늘날에는 대역폭이 급격하게 증가함에 따라 bottleneck이 host쪽으로 전가되었다.
따라서 타이트하게 짜여진 네트워크 스택을 다시 아키텍처를 만들어내는 것이 본 논문의 목표이다.

##### The NetChannel architecture
넷채널 아키텍처의 경우 크게 3개의 느슨하게 결합된 레이어로 구성되어 있다.
Application과 직접적으로 상호작용하는 레이어는 Virtual Network System이다. 이 레이어는 streaming과 RPC traffic을 위한 system call을 처리하기 위한 표준 인터페이스를 제공한다.
내부적으로 VNS는 인터페이스 그 자체로 정확성을 보장하면서 application buffer와 kernel buffer간의 데이터 이동이 가능하다.

넷채널의 핵심은 NetDriver 레이어이다. 이 레이어는 네트워크와  원격 서버를 multi-queue-device로 가상화시킨다. 이때 channel abstraction을 이용한다.
특히 NetDriver 레이어는 패킷 프로세싱을 각 application buffer와 core에서 분리시키게 된다. 따라서 한코어에서 application에 의해 실행되는 데이터 읽기/쓰기는 하나 이상의 channel들로 매핑되게 된다. 이때 application은 breaking되지 않는다.
각각의 채널들은 protocol-specific한 함수들이 구현되어 있고, 동적으로 하드웨어 큐에 매핑될 수 있다.
따라서 이러한 가상화를 통해 몇 개의 application이 얼마나 많은 코어에서 돌아가고 있는지 상관없이 채널이 매핑될 수 있다.

이러한 NetDriver와 VNS 사이에는 NetScheduler 레이어가 있다. 이는 각각의 application에서 각각의 channel로 각 코어의 utilization과 application buffer occupancy, channel buffer occupancy 등의 정보를 바탕으로 데이터를 멀티플렉싱 / 디멀티플렉싱을 해주는 레이어이다.

##### NetChannel benefits
넷채널의 장점은 현재 존제하는 프로토콜 처리 구현을 수정할 필요 없이 새롭게 동작 포인트를 만들 수 있다는 것이다.
이 포인트는 넷채널의 분리된 아키텍처의 직접적인 결과이다. 이는 각 레이어의 독립적인 scaling이 가능할 뿐만 아니라, 다중 channel을 통해 유연하게 멀티플렉싱과 디멀티플렉싱을 할 수 있다는 점이다.
네트워크 성능을 높이기 위해 application developer들이 더 이상 코드를 튜닝하지 않아도 된다. 본 넷채널에서 최대한의 성능을 뽑아낼 수 있기 때문이다.
또한 넷채널에서는 새로운 4계층 프로토콜의 구현에 있어서 기존의 호스트를 분해하지 않고 새롭게 디자인 할 수 있으며, 이는 실험하는데 있어서 더욱 쉽다.
이 새로운 프로토콜의 구현은 데이터 복사, 레이턴시 기반/ 대역폭 기반 통신의 분리, CPU 스케쥴링, 로드 밸런싱의 문제들을 고려할 필요가 없어진다.
따라서 결과적으로 스토리지 스택과 비슷하다.

##### What this paper is not about
넷채널 아키텍처는 zero-copy mechanisms과 io_uring interface에 대한 보충이다.
io_uring을 넷채널 아키텍처에 사용하였을 때 좋다는 것 또한 보여줄 것이다.
한정된 CPU 속도와, 수백 기가비트의 대역폭에 따라, 멀티 코어 네트워크 프로세싱은 필수불가결이고, 따라서 넷채널 아키텍쳐는 호스트의 모든 리소스를 최대한 사용하는 것에 집중하였다.
두 번째로, 넷채널 아키텍처는 어디에 있든 상관 없다는 것이다. 커널에 있든 userspace에 있든 하드웨어에 있든 상관 없다. 본 연구에서는 리눅스 커널에서 구현하기로 하였다.
추후 연구 과제로는 넷체널이 userspace와 hardware network stacks에 통합되는 것을 탐색할 것이다.

### Motivation
여기서는 기존의 리눅스커널이 다방면으로 부족하다는 것을 보여주기 위해 다양한 전송 디자인에서 그 성능을 측정하였다.
- Understanding of Host network stack 논문에서 보았듯이, 최대 60Gbps까지 성능이 안나왔고, 이에 대한 bottleneck은 receiver side의 core였다. 이는 오늘날의 네트워크 스택이 동적으로 resource를 할당해주지 않기 때문이다.
- 연결의 갯수를 동적으로 지정하지 못하는 것 또한 bottleneck의 원인중 하나이다. 이는 short message들에 대하여 scalability를 달성하지 못하기 때문이다.
- L-app과 T-app에 대하여 steering을 하지 않는 것 또한 오늘날 리눅스커널의 문제이다. 이를 스티어링하는 메커니즘이 존재하지 않아(다른 코어로 옮겨준다던가) 둘의 경쟁이 일어나고 이는 L-app의 성능저하를 일으킨다.
##### 2.1 Measurement Setup
기존 커널들의 다양한 환경에서 성능을 측정할 방법을 제시하고 있다.
##### 2.2 Limitations of Existing kernel Stack
1) Static and dedicated pipeline => lack of scalability for long flows.
	각종 기술들을 적용해도 100Gbps를 saturate하지는 못했다. 
	새로 찾은 것은 aRFS를 끄고 수동으로 패킷을 스티어링 했을 때, 실제 성능이 조금 더 상승했다는 것이다. 이는 NUMA가 활성화 된 시스템에서 같은 NUMA 노드의 다른 코어로 패킷 프로세싱을 맡김으로써 application이 동작하는 core 입장에서는 offload가 된 것과 같은 효과를 가지기 때문이다.
	이러한 노력에도 불구하고 결국에는 Saturate 되지 못했는데, 여전히 data copy가 주요한 오버헤드로 작용하고 있기 때문이다.
	추가적으로 MPTCP를 사용했을 때도 aRFS를 사용했을 때 최고 성능을 보여준다. 몇 개의 subflow를 사용하던지 간에 모든 처리가 application이 실행중인 코어에서 일어나게 된다. 이러한 이유로 인해 오히려 subflow가 늘어나면 처리해야 할 네트워크 프로세싱 작업이 늘어나므로 총 쓰루풋은 감소하게 된다.
	여기서 aRFS가 없다면 NUMA모드에서는 다른 코어에 할당 될 경우가 존재하므로 매우 낮은 성능을 보여주게 된다. 따라서 aRFS가 필수적이다.
	결론적으로 설정에 상관없이, 여기서는 해당 application이 동작중인 코어에서만 packet processing이 이루어지고, 다른 코어들은 idle 상태이기 때문에, host의 절대적인 전체 성능을 다 사용할 수가 없다는 것을 꼬집고 있다. 심지어 zero-copy mechanism을 사용하더라도 여전이 one core processing pipeline으로 인해 수백Gbps 스케일의 대역폭을 다 채울 수는 없었다. 따라서 결론은, 멀티코어 프로세싱이 필수불가결이다.
	
2) Static and dedicated pipeline => lack of scalability for short message processing.
	여기서는 4KB짜리 RPC 요청들을 지속적으로 보냈다. 평균적인 성능은 9.88Gbps였다. 여기서 bottleneck은 sender-side였다. 코어 내부의 커널 함수 호출을 조사하였을 때, 주요한 task는 TCP/IP processing이였고, 거의 절반의 CPU cycle을 사용하고 있었다. MPTCP도 도움이 되지 못했는데, 결국 가장 중요한 bottleneck이 sender side에서, application이 동작하는 하나의 코어에서 이루어지는 protocol processing이였기 때문이다.
	이는 application 개발자가 직접 multi-socket을 사용하면 그 오버헤드를 극복할 수 있지만, 그 개발자가 얼마나 해야 적당한 지를 모르는게 문제이다. 게다가 커널 모드에서만 접근 가능한 congestion 정보, CPU 사용량 등 외부 상황을 알아내기가 매우 까다로워 이를 더욱 어렵게 한다.
3) Tightly-integrated pipeline => lack of performance isolation.
	같은 NUMA 노드 안에서 L-app과 T-app을 동시에 돌렸을 때, isolated와 interference의 tail latency는 많은 차이가 있었다. 거의 37배 정도의 레이턴시 차이를 보여주고 있었다. 한 코어에 L-app과 T-app이 겹칠 수 있는 실험 조건을 설정하여 이를 구현하였다.
	이러한 문제에 대한 해결책으로 prioritization 기술을 적용하더라도 이는 해결할 수 없었는데, qdisc 레이어에서 pfifo_fast scheduling policy를 적용하고, CPU scheduling에서 우선순위를 최우선으로 하더라도 가시적인 개선효과를 보지 못했다. 우선 qdisc에서는 TCP Small Queue feature 때문에 많은 양이 qdisc 레이어에 큐잉되지 않기 때문이고, CPU 스케줄링의 경우 우선 IRQ thread에서 주요한 네트워크 프로세싱이 일어나므로 우선순위가 큰 영향을 미치지 않고, IRQ processing은 non-preemptive 이기 때문에 IRQ 스케줄링을 하더라도 큰 효과를 기대하기가 어렵다.
	따라서 이러한 문제를 우선순위 지정을 통해 해결할 수 없고, 결국은 L-app과 T-app을 분리 된 코어에서 처리해야 할 것이다.
### NetChannel DESIGN
VNS는 application과 통신하는 interface 역할로, 기존의 네트워크 스택과는 다르게 application과 주고받는 데이터를 버퍼링하고, 나머지 패킷 프로세싱 파이프라인과 분리시키는 역할을 한다.
가장 아래의 NetDriver는 네트워크를 channel이라는 일반화 된 가상화를 통해 위쪽 레이어에 제공하게 되고, 이를 multi-queue 장치로 추상화 시킨다.
applicatioin과 channel을 분리함으로써 유연하고, 잘게 쪼개진 멀티플렉싱/디멀티플렉싱과 두 레이어에서 데이터를 스케줄링하는 것이 가능해졌다.
멀티플렉싱과 디멀티플렉싱은 NetScheduler 에서 이루어진다. 이 스케줄러는 pluggable이며, 동적인 스케일링이 가능하다.

##### 3.1 Virtual Network System(VNS) Layer
application의 수정없이 본 Netchannel을 적용하기 위해 virtual socket을 통해 POSIX socket interface를 지원한다. application은 주로 socket과 system call 혹은 io_uring을 통해 상호작용하게 된다.

###### Ensuring correctness of interface semantics
모든 가상 소켓은 내부적으로 Tx와 Rx 버퍼를 유지하고 있다. reliablity는 NetDriver layer의 channel으로 넘기고, VNS에서는 서로 다른 virtual sockets들에서 순서를 보장해야할 것이다. 여기서 생기는 문제는 기존의 TCP에서의 sequence number는 더 이상 유효하지 않다는 것이다. 따라서 이것을 해결하기 위해 sender side에서는 각 virtual socket의 data packet들에 대하여 VNS가 각각의 패킷을 나타내는 시퀀스 번호를 삽입하게 된다. 이는 section4에서 더 자세히 논의 될 것이다. 결국 receiver side의 VNS는 이 시퀀스 번호를 보고 OOO 처리를 하게 된다.

>그렇다면 NetChannel이 활성화되어 있는 시스템끼리만 작동이 되는 건가?
>그렇다면 NetChannel <-> 기존 network stack 끼리 동작은 안되는 건가?
###### Decoupling data copy from application threads
VNS는 코어당 worker threads를 유지하고 있다. 이것은 userspace와 kernel 간의 data copy를 병렬적으로 처리하기 위해 존재한다. 따라서 모든 코어를 사용할 수 있다는 장점이 있다.

##### 3.2 NetDriver Layer
###### Channel abstraction
NetDriver에서, 각각의 channel은 Tx/Rx queue들과 독립적인 네트워크 계층(TCP reliable funcs) 처리 파이프라인을 가지고 있다.  이 때, 각각의 queue들이 create(), destroy()를 통해 생성, 제거되고, enqueue(), dequeue()를 통해 데이터가 전송되고, 데이터를 수신할 수 있다. 앞의 두개는 connection-oriented인 연결에 사용하면 되고, connection-less인 경우 뒤의 enqueue(), dequeue()만으로 동작이 이루어진다.
###### Decoupled network layer processing
이러한 NetDriver의 channel들은 VNS의 virtual interfaces로부터 분리되었는데, 이러한 아키텍처를 선택하여, 각각의 application이 사용하는 소켓과 코어가 하던 네트워크 계층 처리를 분리시킬 수 있다는 장점이 있다. 따라서 얼마나 많은 application이 얼마나 많은 socket/core를 사용하던지 간에 독립적으로 여러 개의 channel을 생성하고 사용할 수 있다. 또한, 이는 동적으로 할당/해제 할 수 있으므로, channel의 load에 따라 dynamic하게 resource를 할당하고 사용할 수 있다.

###### Integrating new transport designs
NetDriver는 이제 multi-queue device의 추상이므로, 새로운 네트워크 프로토콜을 짜는 것은 새로운 device driver를 만드는 것과 동일해진다. 이는 즉, 소켓과 관련된 귀찮은 API들을 고려할 필요가 없다는 뜻이다.
###### Piggybacking on transport-level flow control
virtual socket의 Rx buffer의 오버플로우를 방지하기 위해, NetDriver는 기본적으로 piggybacks을 flow control에 적용하고 있다.
###### Alleviating HoL blocking
하나의 channel이 여러개의 virtual socket들에게 공유 될 수 있기 때문에, 하나의 virtual socket에서 발생한 문제가 같은 channel을 사용하는 다른 virtual socket들에게 까지 영향을 미칠 수 있다. 따라서 특정 application이 장기간 read를 하지 않아 virtual socket과 channel의 queue가 가득 차서 더 이상 사용하지 못하게 되는 문제가 발생한다.
이러한 문제를 해결하기 위해 virtual socket마다 response queue를 두게 되었다. 만약 virtual socket이 가득 찼다면 더 이상 transmit을 안하게 하면 되지만, channel queue에 남아있는 해당 virtual socket의 패킷을 response queue에 넣게 되는 것이다. 따라서 channel queue에 들어온 것은 언제든지 response queue에 들어가게 된다.
이때 response queue의 자원은 channel의 버퍼를 사용하게 하여 channel의 flow control이 response queue에 들어간 entry의 갯수가 증가할 때 작동하도록 하였다.

##### 3.3 NetScheduler Layer
NetScheduler는 3가지 주요 역할이 있다.
1) application data를 channel로 multiplexing 및 scheduling
2) host간의 channel의 갯수를 동적으로 조절함.
3) data copy 요청을 scheduling함.
NetScheduler의 커널 상에서 위치를 보았을 때, 다양한 정보를 활용할 수 있다는 장점이 있다.
또한 본 논문에서는 스케줄러는 제안하는게 아니라, 다양한 스케줄러를 적용시킬 수 있는 프레임워크를 제공하고자 한다.

###### Dynamic sceduling of application data to channels
