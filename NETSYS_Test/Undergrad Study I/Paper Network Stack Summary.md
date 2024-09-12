---
creator: 구성현
type: Paper Review
created: 2022-06-21
---

## Summary

datacenter access link bandwidth를 증가시켜왔는데

host resource 가 침체됨

요새 논문이 실패함

1. short flow에 대해 연구한게 많은데 long flow를 더 많이 씀
    
2. datacenter workload는 여러 flow size를 mix해서 씀
    
3. network stack은 최대 42GB 코어당 처리량
    
    그러나 data copy 에서 오버헤드가 나옴
    
    그래서 receiver쪽에서 병목현상 많이 발생
    
    ⇒ implications : zero copy
    
4. BDP가 커지면 cache overwrite가 많아져서 drop이 발생
    
    ⇒ implications : host 자원의 조정 = latency, cache miss rate ⬇️, window size
    
5. 같은 NUMA node에서 한 flow 데이터는 다른 flow 데이터를 오염시킴
    
    즉, 큰 패킷을 모으는 기회를 잃음. overhead가 증가
    
    그래서 같은 자원을 sharing 하면 나빠질것
    
    ⇒ implications: host resource를 잘 조정 to minimize contention between active flows.
    
6. short와 long flow의 병목현상이 다른데도 같은 packet processing
    
    전자는 높은 packet processing overhead ↔ 후자는 높은 data copy overhead
    
    ⇒ implications: application-aware CPU scheduling, host packet processing pipelines
    

Sender-side : app → write syscall → skb → GSO(TSO) → Tx queue

Receiver-side : RX queue → Rx descriptor → DCA → IRQ → skb → GRO(LRO) → app

1. Single Flow
    
    1. 단일 코어는 더이상 불충분
        
        왜?
        
    2. receiver쪽 CPU가 bottleneck
        
        (1) data copy : 다른 NUMA node에서 data copy하기 때문에 오버헤드 ↔ sender는 L3 cache
        
        (2) skb allocation : 드라이버에서 MTU 크기의 skb를 할당, GRO layer에서 합쳐짐 ↔ TSO
        
    3. CPU cycle
        
        TCP/IP 프로세스에서 오버헤드가 주로 발생, aRFS disabled : lock 오버헤드가 높음
        
    4. 단일 flow도 높은 cache miss
        
        (1) BDP > L3 capcity
        
        (2) suboptimal cache utilization
        
    5. DCA가 NIC-local NUMA node를 제한
        
        aRFS enabled : NIC이 remote NUMA node의 CPU 메모리에 DMA, L3 cache에 DCA 불가능
        
2. One-to-one
    
    1. Host optimization은 flow 수의 증가에 비효율적
        
    2. Processing 오버헤드가 네트워크 포화에 따라 변동
        
        network is saturated : receiver thread가 새로운 data가 올때까지 sleep → scheduling 오버헤드
        
3. Incast
    
    1. 코어마다 flow가 증가할때 data copy 오버헤드도 증가
    2. sender의 TCP는 receiver의 scheduling을 무시
4. Outcast
    
    1. sender 쪽 processing pipeline은 코어당 89Gbps까지
        
        (1) efficiency of TSO : no CPU overhead & no degrade
        
        (2) aRFS : receiver에 비해 flow 수가 너무 많지 않음
        
5. All-to-All
    
    1. 많은 수의 flow 때문에 flow 당 batching 기회가 감소

In-network congestion은 스위치에서 패킷 drop을 야기

코어 당 throughput의 영향은 최소

증가된 ACK processing 때문

sender쪽에서 높게 감지

Flow 크기의 영향은

DCA는 extremely short flow에 대해 도움이 되지 않음

long 과 short을 섞는 것은 harmful

DCA의 영향은

DCA disabled일때 19% degration이 발생 : data copy benefit 감소

IOMMU의 영향은

memory management 오버헤드는 심각한 degration을 초래

Congestion control protocol의 영향은

TCP CUBIC, BBR, DCTCP : minimal impact on throughput-per-core