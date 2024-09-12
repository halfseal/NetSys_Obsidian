
msi-x :

ice_vsi ⇒ ice_q_vertor

⇒ ice_ring_container (rx, tx 각각 있음),

⇒ ice_[rx|tx]_ring * 은 링을 가르키는 포인터

→next 포인터는 다음 ice_[rx|tx]_ring을 가르키는 포인터임. 고로 linked list 형태

⇒ msi_map ⇒ ~/linux-6.9/include/linux/msi_api.h virq는 관련된 interrupt number, index는 msi-x의 테이블이나 소프트웨어로 관리되는 인덱스임.

  

각각의 q_vector는 rx, tx ring buffer의 그룹이다. 이러한 큐들의 그룹이 하나의 인터럽트에 할당 되게 되고, 이 때 묶음 큐는 linked list로 ice_ring_container에 포인터로 참조되어 있다.

→ devm_request_irq() ==(include/linux/interrupt.h)==

→devm_request_threaded_irq ==(kernel/irq/devres.c)==

==*devres_alloc은 dma를 실행함. 여기서는 irq_devres를 DMA 설정하고 있는데, 인터럽트 번호와 해당 device id 포인터 두 개의 필드를 가진 간단한 구조체임.==

→ devres_alloc → __devres_alloc_node → alloc_dr → kmalloc_node_track_caller==(함수 호출을 추적함)== → kmalloc_node==(NUMA 특정 노드에 메모리를 할당하는 작업을 하는 함수)==

→dmam_alloc_coherent==(include/linux/dma-mapping.h)==

→ dmam_alloc_attrs==(kernel/dma/mapping.c)==

→ dma_alloc_attrs

→ dma_alloc_direct를 통해 device가 direct allocating이 가능하면 IOMMU를 bypass로 하고 해당하는 작업을 수행. 아니라면, ops→alloc 함수를 통해 allocation 수행.

→request_threaded_irq() ==(kernel/irq/manage.c) / thread_fn은 NULL로 전달됨.==

→irq_to_desc(irq) : irq 번호를 통해 irq_desc 구조체를 반환하는 함수. ==좀더 살펴볼 여지가 있음==

[[→__setup_irq()]]

→ice_set_cpu_rx_rmap()