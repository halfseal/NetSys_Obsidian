---
sticker: lucide//screen-share-off
---
# iperf 800Gbps ppt 리뷰
---
참고 자료
- https://lpc.events/event/16/contrib,utions/1345/attachments/991/1917/LPC-High-Speed-Linux-Networking.pdf
- https://www.usenix.org/system/files/osdi24-skiadopoulos.pdf
### index
---
- Test Setup
- ZC / DDP
-  Reducing packet Rate Handled by S/W
- Socket Buffers 와 Syscalls
- Huge Pages
- 기타

### Test Setup
---
Ryzen 9 Zen 3 cpu (5950x 16-core), 5GHz, 128GB, DDR4, 3200 MT/s의 하드웨어 스펙을 가지고 있다. 여기서 MT/s란 초당 전송량을 의미하며, PC-25600 스펙의 메모리임을 의미한다.
우분투 버전은 20.04 이고, 5.13.19 커널 버전이다.
NIC는 Xilinx FPGA - VCU1525를 사용하였다.

하드웨어적인 한계점으로는 우선 PCI 연결이 gen3x16이므로 최대 128Gbps일 것이라는 것이다. 그런데 여기서 벤치마크 앱의 페이로드는 의미가 없을 것이다. 왜나하면, 결국 헤더에서 실제 사이즈만큼 작업이 진행 될 것이기 때문에 페이로드는 드랍해도 무방하다.
즉 커널의 네트워크 스택과 NIC에서 다루어지는 영역은 오로지 헤더 영역이고, Payload는 기껏해야 memcpy일 텐데, zerocopy라면은 본 벤치마킹에서는 의미가 없으므로 일부 코드 수정을 통하여 100Gbps 환경에서도 그 이상의 Bandwidth에 대한 모사를 할 수 있다는 것이다.

### Zerocopy / Direct Data Placement
---
memcpy는 반드시 피해야 할 것 중 하나이다. 이는 ~30Gbps의 제한이 있다. 따라서 Zerocopy와 direct data placement는 고대역폭 네트워킹을 위해 반드시 필요한 제반 기술이다.

리눅스에 존재하는 zc api들은 Tx에서는 get_user_pages와 reaping completions(recvmsg)으로 이루어질 수 있고, Rx에서는 특정한 MTU 크기와 헤더 분할로 정확하게 PAGE_SIZE 크기의 페이로드가 되도록 하거나 PAGE_SIZE보다 작은 데이터에 대해서는 memcpy와 함께 side band를 사용한다.

#### modified iperf3 to avoid memcpy
따라서 Tx에서는 ZC API를 지원할 수 있도록 수정해 주었다. 여기서 page들을 pinning과 completion 하는데 추가적인 CPU cycle이 들 것이지만, 하나로 제한하지 않는다.
Rx에서는 ZC가 너무나도 제한적이라 그 의도를 모방하는 형식으로 data를 dropping하는 것으로 실현하였다.
payload가 uperspace로 memcpy가 되는 것을 막기 위해 MSG_TRUNC으로 payload가 사라지도록 하였다.

#### memory management
 4096 L2 MTU일 때 400G는 12+M pages/s 정도의 속도이다. 실제 숫자는 MTU와, S/W와 H/W가 어떻게 버퍼를 다루는지에 따라 달라진다.
FPGA 드라이버는 패킷을 위한 페이지를 다른 고대역폭 NIC 드라이버와 비슷하게 다루게 된다.
Max pps는 테스트동안 31+M pps였고 이것은 드라이버에 의해 초당 31+M개의 page를 다루고 있다는 것이다.
심지어 1500MTU에서 split page 는 16.5 pages per second를 의미한다.
기본 레이어로써 페이지 풀 인프라는 system page allocators이다.
 CPU당 cache 가장 위에서 관리되는 드라이버이다(?)
 skb는 napi cache를 통해 재사용 되게 되는데, 주로 build_skb 너머의 napi_build_skb를 사용한다.
###### Key Point : 오늘날의 버퍼 관리 구조는 너무 많은 오버헤드를 가지고 있다.

### Reducing packet Rate Handled by S/W
---
패킷 당 더 많은 데이터로 인한 S/W stack overhead를 청산하기 위해서는 FIB lookup, socket loockup, tc and netfilter hooks 등의 방법이 있다.
또한 MTU 크기를 늘림으로써 통신선 안에 패킷당 더 많은 payload를 담을 수 있도록 한다.
마지막으로 TSO를 S/W GRO나 H/W LRO 속에 집어 넣는다. 목표는 S/W적인 방법에 의해 64kB까지 확인했었던 그 효율적인 MTU를 만들어내는 것이다.

![[Pasted image 20240731110215.png]]
위 그림은 S/W적인 방법인 GRO를 통해 TSO를 구현한 것이다.

![[Pasted image 20240731110359.png]]
위 그림은 H/W적인 방법인 LRO를 통해 TSO를 구현한 것이다. 여기서 LRO는 TCP/IPv4에 제한되어 있지만, GRO는 일반화된 LRO라고 생각하면 된다. 따라서 GRO는 소프트웨어적으로 구현되어 있는 것이다.

![[Pasted image 20240731112615.png]]
또다른 주요한 그래프로는 위의 그래프가 있다. 이그래프에서 말하는 TSO Efficiency란 주어진 MTU의 배수가 되게 끔 TSO가 셋팅되므로,TSO를 적용하려하는 패킷의 크기가 어떻게 바뀌는지, 그 효율성이 어떤지 그래프로 나타내었다. 예를 들어 MTU가 8000이라면 65,535의 최대크기를 갖는 패킷이 대략 64,000정도의 크기를 가져야 MTU에 맞게 잘려 들어갈 것이고, 최대 전송속도를 유지할 수 있을 것이다.

#### 그래프 해석
GRO/LRO를 적용하지 않은 케이스에서는 최대 pps가 ~3M pps였다. MTU만을 늘렸을 때는 아무런 효과가 없었다. 
또한 MTU가 TSO goal에 영향을 미치므로 결국에는 Ideal한 TSO Packet Size를 가질 수 있도록 최대한 노력하면서 대역폭을 최대화하는 방법을 찾는 것이 중요할 것이다. 
GRO는 도움이 될 수 있겠지만, S/W적인 분석은 추가적인 CPU cycle을 요구하므로 시스템 자원을 소모하게 된다.
LRO는 상당한 pps 성능을 보여주고 있다.

##### Key Point : scale up을 위해서는 강력하고 굳건한 H/W 기반의 LRO scheme가 필요하다.

### Socket Buffers 와 Syscalls
---
한번의 syscall(recvmsg /sendmsg) 더 많은 데이터가 옮겨지도록 해야 할 것이다. 따라서 더 많은 데이터가 socket buffers에 큐잉되게 된다. 따라서 더 높은 대역폭을 달성 할 수 있다.
이는 iperf3 -l 과 -w args로 설정할 수 있다.

추가적으로, 128M socket buffer와 64M socket buffer를 비교하였을 때 ~3300 MTU까지는 별 차이 없이 BandWidth가 증가하다가, 600Gbps를 기준으로 크게 흔들렸다. 그러나 미세하게 128M socket buffer가 더 BandWidth가 컸다.
##### Key Point : 시스템 콜 없이 datapath를 manage할 필요가 있다.

### Huge Pages
---
만약 페이지의 크기가 디폴트인 4KB라면 TSO skb마다 17개의 fragments가 발생할 것이다. *4KBx17 = 68KB > 65KB 이므로*
그러나 만약 2MB의 hugepage를 사용한다면, skb당 2~3개의 fragments만 발생할 것이다. 따라서 ZC가 아닌 경로의 TCP는 skb 당 2~3개의 fragments를 가지게 된다.
실제로 그래프를 확인해 보면 4KB의 페이지 크기로는 600Gbps의 bandwidth를 넘지 못하였지만, hugepage를 적용했을 때는, 상당한 성능 향상을 보이고 있었다.

#### Key Point : 네트워크 스택에서 사용 되는 buffer 구현을 위한 오버헤드를 줄여야 한다.

### 기타
---
1. Congestion Control Algorithm - 최신 버전일수록 더욱더 성능 향상이 컸다. 또한, 800Gbps를 달성하는데 큰 문제가 없었다.
2. 하드웨어 고려사항 - 
   CPU의 속도가 패킷을 처리하는 속도와 직결되어 있다. 또한 L3 cache가 packet processing core와 application core가 서로 같은 L3 cache를 공유할 수 있어야 더 많은 성능 향상을 꾀할 수 있을 것이다. 그러나 이는 CPU 아키텍쳐 디자인마다 다를 것이므로 공통된 성능향상은 기대하기 어렵다.
3. NIC의 ring size를 8192로 키운다. - Irq driven system의 경우 irq handler가 실행되기 전에 너무 많은 패킷이 쌓이기 마련이다. 따라서 ring size를 키워줘야 할 것이다. 
4. Scale out - VCU 1525는 2개의 mac주소를 가지고 있어서 scale out이 가능하다.
5. Kernel Regressions - 최신 커널일 수록 그 대역폭이 줄어들고 있는 모습을 볼 수 있다.

### 요약
---
리눅스 TCP 스택은 높은 대역폭으로 확장이 가능하나, socket API의 병목현상은 제거되어야 한다.
어플리케이션 버퍼에 직접 패킷 페이로드를 전달하는 하드웨어 scheme가 필요로 하다.
더 좋은 memory / buffer management scheme가 필요로 하다.
반드시 견고한 LRO scheme가 필요로 하다.
system call을 줄이거나 없애야 한다.
skb의 application memory의 간단한 표현이 필요하다.
scale out과 up을 허용한는 resource isolation이 필요로 하다.