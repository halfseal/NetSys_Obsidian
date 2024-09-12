[https://d2.naver.com/helloworld/47667](https://d2.naver.com/helloworld/47667)

![[Untitled 3.png|Untitled 3.png]]

net_open()에서 request_irq()를 실행하여 NIC의 IRQ를 등록하게 된다. 이때 이것은 커널의 전역변수인 irq_desc[]에 등록되게 되어 do_IRQ(할당받은 irq 번호)로 호출 될 때 여기서 찾게 된다.

---

~/linux-6.9/drivers/net/ethernet/realtek/atp.c

인텔 ice.c부터 확인할 것.(100Gb 드라이버)

dma_alloc_coherent → 이것들이 수행되는 곳이 dma가 가능하게 메타데이터를 셋팅해주는 시작점.

DMA-API_HOWTO.txt ⇒ kernel.org

→atp_interrupt(int irq, void *dev_instance) ⇒이 것은 request_irq를 통해 등록된 함수임.

→ net_rx(struct net_device *dev) 실행

→ 만약 패킷이 정상이라면, pkt_len = (rx_head.rx_count & 0x7ff) - 4 로 패킷 길이를 계산하고,

→ struct sk_buff *skb 선언 → Malloc으로 할당함(netdev_alloc_skb(dev, pkt_len +2))

→ read_block(ioaddr, pkt_len, skb_put(skb, pkt_len), dev→if_port) 으로 장치에서 패킷을 읽어옴

skb_put을 통해 sk_buff의 멤버 중에 페이로드를 저장할 수 있는 데이터 포인터의 메모리 영역에 패킷 페이로드를 적재할 수 있음

→ netif_rx(skb)실행

~/linux-6.9/net/core/dev.c

→netif_rx(struct sk_buff *skb)

→netif_rx_internal(struct sk_buff *skb)

→enqueue_to_backlog(struct sk_buff *skb, int cpu, unsigned int *qtail)

![[Untitled 1 2.png|Untitled 1 2.png]]

softnet_data 구조체에 sk_buff_head 구조체가 있고, sk_buff head는 sk_buff의 linked list 형태임.

[[각종 구조체들의 형태 및 관계도]]

패킷마다 malloc을 통해 sk_buff에다가 저장을 하고, 큐는 이러한 동적할당 된 패킷의 linked list로 구현 됨.

따라서 복사가 필요 없고 그저 포인터만 옮겨주면 됨.

~/linux-6.9/include/linux/skbuff.h ⇒sk_buff구조체에 대하여 자세하게 나옴.

→skb_queue_len(&sd→input_pkt_queue)로 큐의 길이를 잼

→__skb_queue_tail(&sd→input_pkt_queue, skb)

이것이 해당 큐에 패킷을 추가하는 함수임.

→napi_schedule_rps(sd)

![[Untitled 5.png|Untitled 5.png]]

==왜 napi가 qlen이 바로 0보다 클 때는 실행이 안되는지 모르겠음.==

→ __napi_shedule_irqoff(&mysd→backlog)

→ ____napi_schedule(struct softnet_data *sd, struct napi_struct *napi)

napi를 가르키는 쓰레드를 가져와서, 인터럽티블이 아니라면 스케줄링이 되었다고 알리고, 쓰레드를 깨우게 됨.

→list_add_tail(&napi→poll_list, &sd→poll_list) ⇒ linked list 형태의 큐를

~/linux-6.9/kernel/softirq.c

→__raise_softirq_irqoff(NET_RX_SOFTIRQ)

→or_softirq_pending(1UL << nr)

==해당하는 softirq interrupt를 비트마스크를 통해 활성화 시키면 해당 인터럽트가 실행 대기 상태가 되고, 커널이 이를 확인하여 인터럽트를 처리함.==

~/linux-6.9/kernel/softirq.c

→top half에서 hardware interrupt 종료 → irq_exit(void) → __irq_exit_rcu() →invoke_softirq() → do_softirq() → net_rx_action() 과정으로 이루어지게 됨. bottom half에서, 앞서 설정한 softirq 인터럽트 비트맵 매핑에 의해 하드웨어 인터럽트가 끝나고 invoke_softirq()를 통해 do_softirq()가 이루어지고, 이를 통해 net_rx_action()이 실행된다.

~/linux-6.9/net/core/dev.c 위치에 net_rx_action() 함수가 있음.

net_rx_action()은 Iron에서 자세히 정리했듯이 polling 하는 메커니즘을 가지고 있음.