---

![[Untitled 11.png|Untitled 11.png]]

vsi의 종류에 따라 irq_handler가 다르게 allocation되고 있는것이 보임. 가장 기본적으로는 ice_msix_clean_rings() 라는 함수가 할당 되게 됨. irq_handler는 함수 포인터임. 이 함수는 같은 파일인 ice/ice_lib.c에 존재함.

  

ice_msix_clean_rings(void *data를 통해 ice_q_vector가 들어오고 여기서 napi struct를 볼 수 있다.) —> hardware interrupt임. napi 스케쥴링 이후 종료.

→napi_schedule(&q_vector→napi) ==(include/linux/netdevice.h)==

→napi_schedule_prep를 통해 napi 스케쥴링이 가능한지 확인하고, 가능하다면

→ __napi_schedule() → ____napi_schedule() ==(net/core/dev.c)==

쓰레드를 새로 만들어서 napi→thread 멤버를 불러와 해당 프로세스를 실행하게 됨. task_struct는 include/linux/sched.h에 있음.

~/linux-6.9/kernel/softirq.c

→lisat_add_tail(&napi→poll_list, &sd→poll_list)를 통해서 softnet_data 구조체의 poll_list에다가 napi_struct 구조체의 poll_list의 내용을 추가하게 됨.

→__raise_softirq_irqoff(NET_RX_SOFTIRQ)

→or_softirq_pending(1UL << nr)

==해당하는 softirq interrupt를 비트마스크를 통해 활성화 시키면 해당 인터럽트가 실행 대기 상태가 되고, 커널이 이를 확인하여 인터럽트를 처리함. softIRQ는 include/linux/interrupt.h를 확인하면 됨.==

``