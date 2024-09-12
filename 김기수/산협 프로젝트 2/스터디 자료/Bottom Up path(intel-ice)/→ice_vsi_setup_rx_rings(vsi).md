```c title=ice_vsi_setup_rx_rings()
int ice_vsi_setup_rx_rings(struct ice_vsi *vsi)
{
	int i, err = 0;

	if (!vsi->num_rxq) {
		dev_err(ice_pf_to_dev(vsi->back), "VSI %d has 0 Rx queues\n",
			vsi->vsi_num);
		return -EINVAL;
	}

	ice_for_each_rxq(vsi, i) {
		struct ice_rx_ring *ring = vsi->rx_rings[i];

		if (!ring)
			return -EINVAL;

		if (vsi->netdev)
			ring->netdev = vsi->netdev;
		err = ice_setup_rx_ring(ring);
		if (err)
			break;
	}

	return err
```

> 우선 vsi의 num_rxq를 통해 할당 된 rx queue의 갯수가 0이면 에러를 출력하고, 각각의 큐에 대하여, ice_rx_ring을 선언하여 ice_setup_rx_ring 함수를 호출하여 기본 설정을 시작한다.

→ice_setup_rx_ring(ring) ⇒ drivers/net/ethernet/intel/ice/ice_txrx.c

---

- 코드
    
    ```c title=ice_setup_rx_ring()
    /**
     * ice_setup_rx_ring - Allocate the Rx descriptors
     * @rx_ring: the Rx ring to set up
     *
     * Return 0 on success, negative on error
     */
    int ice_setup_rx_ring(struct ice_rx_ring *rx_ring)
    {
    	struct device *dev = rx_ring->dev;
    	u32 size;
    
    	if (!dev)
    		return -ENOMEM;
    
    	/* warn if we are about to overwrite the pointer */
    	WARN_ON(rx_ring->rx_buf);
    	rx_ring->rx_buf =
    		kcalloc(rx_ring->count, sizeof(*rx_ring->rx_buf), GFP_KERNEL);
    	if (!rx_ring->rx_buf)
    		return -ENOMEM;
    
    	/* round up to nearest page */
    	size = ALIGN(rx_ring->count * sizeof(union ice_32byte_rx_desc),
    		     PAGE_SIZE);
    	rx_ring->desc = dmam_alloc_coherent(dev, size, &rx_ring->dma,
    					    GFP_KERNEL);
    	if (!rx_ring->desc) {
    		dev_err(dev, "Unable to allocate memory for the Rx descriptor ring, size=%d\\n",
    			size);
    		goto err;
    	}
    
    	rx_ring->next_to_use = 0;
    	rx_ring->next_to_clean = 0;
    	rx_ring->first_desc = 0;
    
    	if (ice_is_xdp_ena_vsi(rx_ring->vsi))
    		WRITE_ONCE(rx_ring->xdp_prog, rx_ring->vsi->xdp_prog);
    
    	return 0;
    
    err:
    	kfree(rx_ring->rx_buf);
    	rx_ring->rx_buf = NULL;
    	return -ENOMEM;
    }
    
    ```
    

rx_ring→rx_buf를 kcalloc을 통해 새로 선언하게 됨. 이때, rx_ring→count만큼의 배열로 선언되게 됨.

>이후 ALIGN을 통해 PAGE_SIZE 크기의 단위로 몇 바이트가 필요한지 계산하여 size에 저장하게 됨. 이때, 만약 4500바이트의 rx_buf array이고, PAGE_SIZE가 4096이라면, size는 8182가 됨. 이는 ice_32byte_rx_desc라는 구조체를 할당하기 위함임. rx_ring→dma에 해당 주소를 할당하며, 이 때 담고자 하는 정보는 각각의 버퍼에 대한 디스크립터임.

→dmam_alloc_coherent(include/linux/dma-mapping.h)

---

- 코드
    
    ```c
    static inline void *dmam_alloc_coherent(struct device *dev, size_t size,
    		dma_addr_t *dma_handle, gfp_t gfp)
    {
    	return dmam_alloc_attrs(dev, size, dma_handle, gfp,
    			(gfp & __GFP_NOWARN) ? DMA_ATTR_NO_WARN : 0);
    }
    
    ```
    

→ dmam_alloc_attrs(kernel/dma/mapping.c)

---

- 코드
    
    ```c
    /**
     * dmam_alloc_attrs - Managed dma_alloc_attrs()
     * @dev: Device to allocate non_coherent memory for
     * @size: Size of allocation
     * @dma_handle: Out argument for allocated DMA handle
     * @gfp: Allocation flags
     * @attrs: Flags in the DMA_ATTR_* namespace.
     *
     * Managed dma_alloc_attrs().  Memory allocated using this function will be
     * automatically released on driver detach.
     *
     * RETURNS:
     * Pointer to allocated memory on success, NULL on failure.
     */
    void *dmam_alloc_attrs(struct device *dev, size_t size, dma_addr_t *dma_handle,
    		gfp_t gfp, unsigned long attrs)
    {
    	struct dma_devres *dr;
    	void *vaddr;
    
    	dr = devres_alloc(dmam_release, sizeof(*dr), gfp);
    	if (!dr)
    		return NULL;
    
    	vaddr = dma_alloc_attrs(dev, size, dma_handle, gfp, attrs);
    	if (!vaddr) {
    		devres_free(dr);
    		return NULL;
    	}
    
    	dr->vaddr = vaddr;
    	dr->dma_handle = *dma_handle;
    	dr->size = size;
    	dr->attrs = attrs;
    
    	devres_add(dev, dr);
    
    	return vaddr;
    }
    ```
    

> vaddr는 dma_handle과 똑같은 physical memory를 가르키는 virtual address가 될 것이다. 쭉 함수들을 따라가다보면, dev→dma_mem을 통해 원래 device가 처음 초기화 될 때 할당받은 dma memory pool에서 필요한 크기만큼의 dma 메모리를 할당 받아 이에 대한 virtual address를 가져오게 되는 것이다. dma_get_device_base로 device가 할당받은 dma 메모리의 base address를 가져오게 되고, 앞서 bitmap을 탐색하여 찾은 새로 할당 할 dma 주소의 offset, 혹은 pageno를 가지고 필요한 dma address를 저장하고, device의 base address에 대응하는 mem→virt_base에서 오프셋을 더하여 해당하는 virtual address를 반환하게 된다.

→ dma_alloc_attrs

> dma_alloc_direct를 통해 device가 direct allocating이 가능하면 IOMMU를 bypass로 하고 해당하는 작업을 수행. 아니라면, ops→alloc 함수를 통해 allocation 수행. alloc이라는 함수는 dma_map_ops 내부에 매핑된 함수 포인터로, dma_map_ops는 device 구조체에 있는것을 확인하였다. 따라서 ice_probe 이전에 이미 매핑이 되었다고 보았다. 이를 따라가 보았짐만, 아키텍쳐 종속적인 함수들만 존재하여 우선 포기하였다.

만들어진 모든 tx_ring은 ice_main.c의 ice_vsi_setup_tx_rings 에서 vsi→tx_rings[i]를 가르키는 포인터에서 수행하므로 결과적으로 vsi 의 tx_rings 배열의 tx_ring들을 하나하나 설정하게 됨.

ice_rx_ring struct에서 u16 q_index는 해당 링의 Queue number로, CPU가 수신 링들을 구분할 때 사용하는 번호이다. napi에서 skb를 만들 때 해당 번호 + 1 을 skb→queue_mapping에다가 넣어준다. 다음으로 next_to_use와 next_to_clean이 있는데, 이는 일반적인 큐와 비슷하다. push를 하면 next_to_use를 사용하게 되고, napi로 인해 skb로 만들어져 네트워크 스택으로 전달 될 경우 next_to_clean을 사용하여 인덱스에 접근하게 된다. 마찬가지로 index가 overflow라면 0으로 돌아가게 된다.