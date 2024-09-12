우선 num_txq를 통해 할당 된 tx queue의 갯수가 0이면 에러를 출력하고, 각각의 큐에 대하여, ice_tx_ring을 선언하여 ice_setup_tx_ring 함수를 호출하여 기본 설정을 시작한다.

struct ice_tx_ring ⇒ drivers/net/ethernet/intel/ice/ice_txrx.h 에서 볼 수 있음

  

→ice_setup_tx_ring(ring) ⇒ drivers/net/ethernet/intel/ice/ice_txrx.c

→devm_kcalloc → devm_kmalloc_array → devm_kmalloc → alloc_dr → kmalloc_node_track_caller==(함수 호출을 추적함)== → kmalloc_node==(NUMA 특정 노드에 메모리를 할당하는 작업을 하는 함수)==

→dmam_alloc_coherent==(include/linux/dma-mapping.h)==

→ dmam_alloc_attrs==(kernel/dma/mapping.c)==

→ dma_alloc_attrs

→ dma_alloc_direct를 통해 device가 direct allocating이 가능하면 IOMMU를 bypass로 하고 해당하는 작업을 수행. 아니라면, ops→alloc 함수를 통해 allocation 수행.


만들어진 모든 tx_ring은 ice_main.c의 ice_vsi_setup_tx_rings 에서 vsi→tx_rings[i]를 가르키는 포인터에서 수행하므로 결과적으로 vsi 의 tx_rings 배열의 tx_ring들을 하나하나 설정하게 됨.