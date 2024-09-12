  

**Abstract**

- container vs VM
    - container 는 os를 공유한다
    - isolation이 어렵다
- network stack의 computation overhead가 isolation을 방해한다
    - high computation overhead가 같은 서버를 사용하는 container의 computation을 감소시킨다
- IRON
    - container 대신 networking stack에 사용된 시간을 계산한다
    - isolation을 강화(enforcement mechanism)

  

1. Intro

- cgroup이 isolation 담당한다
- resource allocation
    - predtermined bucket allocation 방법은 비효율적이다
- trade-off
    - underprovision: 매출 감소
    - loose isolation: 성능저하
- container가 cgroup으로 할당된 cpu보다 더많이 소비 == isolation 파괴
    - kernel가 interrupt에 쓰는 시간은 container에 부과되지 않는다
    - 다른 container에게 6배 성능저하 유발한다
    - kernel의 overhead가 크다
- IRON
    - monitor, charge, enforce cpu usage for network traffic
    - 리눅스 스케줄러와 통합
    - cpu 할당량을 모두 소비한 후, traffic을 처리하는 것은 isolation을 파괴할 수 있다
        - hardware based으로 packet drop을 한다
    - accurate charging을 위해서는 packet의 종류를 구분해야한다
    - cgroup scheduler와 통합
    - novel packet dropping 방식

2.1 network traffic이 isolation 부순다

- linux scheduler가 network traffic의 interrupt에 쓰인 시간을 제대로 계산안한다
- linux container scheduling (cgroup)
    - cgroup은 container 당 quota만큼 시간을 준다
    - scheduler가 runtime에 container가 실행된 시간을 저장한다
    - runtime이 quota에 도달하면 container 가 throttle된다
- linux interrupt handling
    - top half (hw interrupt)
    - bottom half (sw interrupt)
        - hw interrupt이 끝나고 난 후, kernel이 softirq processing을 승인할때
        - process context이라서 현재 실행된 process가 시간을 들여서 처리한다
        - Ksoftirqd (time이나 packet 예산을 넘어서면 실행된다)
            - 남아있는 softirq를 처리한다
            - 실행되는 시간은 그 어떤 container에도 할당되지 않는다
            - container에게 할당될 수 있는 시간을 줄인다
- kernel packet processing
    - pacekt processing은 process context라서 올바른 container에 할당된다
    - isolationd 이 깨지는 2가지 경우
        - nic 가 pkt transmisson을 마치고 pkt resource를 free할때
            - 이는 softirq로 실행된다
        - stack에 buffering할때
            - buffer에 pkt을 저장한 후, kernel이 종료되면
            - pkt을 dequeue할때 softirq 사용한다

2.2 putting iron in context

- 기존에는 LRP를 이용
    - pkt을 받더라도 즉시 interrupt하지 않는다
    - container가 cpu 할당을 했을 때, pkt processing을 한다
- LRP의 단점 3가지
    - receiver에만 적용가능
    - socket 당 thread가 필요하다. thread를 늘리면 overhead도 증가
    - scheduler가 pkt thread를 처리해야한다는 것을 우선시 해야한다
        - 그렇지 않으면 pkt가 drop되거나 latency 떨어진다
- IRON이 좋은 이유
    - transmission에 쓰인 resource를 정확히 계산한다
    - 기존 linux interrupt에 통합된다
        - linux에서는 모든 core traffic은 core의 softirq handler로 처리한다
        - shared manner 로 처리, per thread manner가 아니다
- redesign os
    
    - packet demultiplexing
    - problem of library os ?????
        - threaded protocol processing
        - netw processing을 kernel에서 없애는 것은 관리에 어려움을 줄수있다
        - NIC 기반 기술들은 flow를 증가시키는데 어려움이 있다
    
    2.3 impact of network traffic
    
    - methodology
        - 1 victim은 sysbench, 나머지도 sysbench
        - 1 victim은 sysbench, 나머지는 network flooding
        - 두 시나리오의 성능차이(penalty factor)를 측정한다
    - UDP sender
        - no rate limit의 경우, 성능저하가 없다. app 의 수요가 bandwidth보다 작아서이다
        - rate limit은 최대 1.18의 성능저하를 유발한다. app의 수요가 더 크고 이는 queueing 유발
        - ==HTB는 1-3Gbps에서 큰 overhead 경험한다.?????????????????????==
    - TCP sender
        - tcp overhead가 udp overhead보다 크다
            - ack 전송해야하고, tcp layer에 buffer해서
            - 모두 softirq로 실행된다
        - flow가 증가할수록 overhead가 증가한다
            - flow가 증가하면 ack도 증가 (protocol processing)
            - flow가 증가하면 traffic pattern도 복잡해져서 queueing을 발생한다
    - UDP receiver
        - receiver가 sender보다 penalty 심하다
            - netw stack을 softirq로 이동해서이다
    - TCP receiver
        - input rate이 커질수록 penalty도 커진다
        - 최대 6배 성능저하
            - ksoftirq가 cpu의 54% 소비
    
    3.1 accounting
    
    - receiver based accounting
        - kernel은 언제든지 interrupt이 가능하다
        - 그래서 scheduler data 를 사용해야한다 (cumtime, swaptime, now)
        - time = cumtime + (now - swaptime)
        - 6가지 종류의 softirq handler (HI, TX, RX, TIMER, SCSI, TASKELT)
        - ack overhead는 start and end timestamp에 의해 제대로 계산된다
    - sender based accounting
        - pkt을 전송할 때, kernel은 nic queue의 lock을 얻어야한다 == overhead
        - batch (pkt 모음)을 전송하는 시간을 계산하고 , 그 비용은 pkt당 분배한다???
    - container mapping and accounting data structure
        - sender에서 skb는 그것의 cgroup과 연관되어있다
        - receiver에서는 IRON이 ip addr의 hash table을 저장한다

3.2 enforcement

- cpu scheduler와 통합, pkt drop 기능도입
- schedule integration
    
    - cfs scheduler
    - container는 quota만큼 실행되며, 이는 period 보다 작다
    - global level에서 quota가 runtime만큼 설정된다
    - runtime이 떨어질때까지 실행, 떨어지면 runtime을 다시 설정
    - rt_remain
    - gained는 다른 container의 softirq를 실행한 만큼 저장된다
    
    ![[황재훈/Research Intern/pastNotion/Iron- isolating netw-based cpu in container environment/img/Untitled.png|Untitled.png]]
    

- cpuusage: container에 부과되어야하는 시간, 음수이면 오히려 시간을 추가해줘야한다

![[황재훈/Research Intern/pastNotion/Iron- isolating netw-based cpu in container environment/img/Untitled 1.png|Untitled 1.png]]

  

- dropping pkt

  

1. related work

- isolation of kernel level resource
    - cpu time, cache, memory bandwidth, energy
- resource management and isolation in cloud
    - paragon, borg ,quasar, heracles, retro
- vm network-based accounting
    - gupta
        - hypervisor로 resource 소비 측정
        - 모든 pkt에 일괄적인 cost를 부과한다
        - software based
- real time kernel

  

수업

- container 는 process
- prcoess context, kernel context
- device가 core에 interrupt를 보낸다
    - interrupt를 랜덤으로 보내서 발생한다
- DMA의 반대 cpu가 메모리에 접근
- pcie bus
- ring descriptor (메모리 내에 고정되어있다), packet buffer
- netw subsystem????
- dma의 3가지 방식

  

- intel ixgbe
- ice.c
- dma alloc coherent
    - dma를 가능하게 해주는 기능
    - meta data 생
- iperf