---

- 코드
    
    ```C
    /**
     * ice_probe - Device initialization routine
     * @pdev: PCI device information struct
     * @ent: entry in ice_pci_tbl
     *
     * Returns 0 on success, negative on failure
     */
    static int
    ice_probe(struct pci_dev *pdev, const struct pci_device_id __always_unused *ent)
    {
    	struct device *dev = &pdev->dev;
    	struct ice_pf *pf;
    	struct ice_hw *hw;
    	int err;
    
    	if (pdev->is_virtfn) {
    		dev_err(dev, "can't probe a virtual function\n");
    		return -EINVAL;
    	}
    
    	/* when under a kdump kernel initiate a reset before enabling the
    	 * device in order to clear out any pending DMA transactions. These
    	 * transactions can cause some systems to machine check when doing
    	 * the pcim_enable_device() below.
    	 */
    	if (is_kdump_kernel()) {
    		pci_save_state(pdev);
    		pci_clear_master(pdev);
    		err = pcie_flr(pdev);
    		if (err)
    			return err;
    		pci_restore_state(pdev);
    	}
    
    	/* this driver uses devres, see
    	 * Documentation/driver-api/driver-model/devres.rst
    	 */
    	err = pcim_enable_device(pdev);
    	if (err)
    		return err;
    
    	err = pcim_iomap_regions(pdev, BIT(ICE_BAR0), dev_driver_string(dev));
    	if (err) {
    		dev_err(dev, "BAR0 I/O map error %d\n", err);
    		return err;
    	}
    
    	pf = ice_allocate_pf(dev);
    	if (!pf)
    		return -ENOMEM;
    
    	/* initialize Auxiliary index to invalid value */
    	pf->aux_idx = -1;
    
    	/* set up for high or low DMA */
    	err = dma_set_mask_and_coherent(dev, DMA_BIT_MASK(64));
    	if (err) {
    		dev_err(dev, "DMA configuration failed: 0x%x\n", err);
    		return err;
    	}
    
    	pci_set_master(pdev);
    
    	pf->pdev = pdev;
    	pci_set_drvdata(pdev, pf);
    	set_bit(ICE_DOWN, pf->state);
    	/* Disable service task until DOWN bit is cleared */
    	set_bit(ICE_SERVICE_DIS, pf->state);
    
    	hw = &pf->hw;
    	hw->hw_addr = pcim_iomap_table(pdev)[ICE_BAR0];
    	pci_save_state(pdev);
    
    	hw->back = pf;
    	hw->port_info = NULL;
    	hw->vendor_id = pdev->vendor;
    	hw->device_id = pdev->device;
    	pci_read_config_byte(pdev, PCI_REVISION_ID, &hw->revision_id);
    	hw->subsystem_vendor_id = pdev->subsystem_vendor;
    	hw->subsystem_device_id = pdev->subsystem_device;
    	hw->bus.device = PCI_SLOT(pdev->devfn);
    	hw->bus.func = PCI_FUNC(pdev->devfn);
    	ice_set_ctrlq_len(hw);
    
    	pf->msg_enable = netif_msg_init(debug, ICE_DFLT_NETIF_M);
    
    \#ifndef CONFIG_DYNAMIC_DEBUG
    	if (debug < -1)
    		hw->debug_mask = debug;
    \#endif
    
    	err = ice_init(pf);
    	if (err)
    		goto err_init;
    
    	devl_lock(priv_to_devlink(pf));
    	err = ice_load(pf);
    	devl_unlock(priv_to_devlink(pf));
    	if (err)
    		goto err_load;
    
    	err = ice_init_devlink(pf);
    	if (err)
    		goto err_init_devlink;
    
    	return 0;
    
    err_init_devlink:
    	devl_lock(priv_to_devlink(pf));
    	ice_unload(pf);
    	devl_unlock(priv_to_devlink(pf));
    err_load:
    	ice_deinit(pf);
    err_init:
    	pci_disable_device(pdev);
    	return err;
    }
    ```
    

→ ice_allocate_pf() : pf를 할당함. ==(ice/ice_devlink.c)==

→dma_set_mask_and_coherent() : dma를 설정함

→pci_set_master() : PCI 장치를 버스 마스터로 설정하여 DMA를 수행할 수 있도록 해줌

→ice_init()

→ice_init_dev(pf) : pf로 dev를 가져옴

→ice_init_feature_support(pf) :

→ice_request_fw(pf) :

→ice_init_pf(pf) :

→ice_init_interrupt_scheme(pf) :

→ice_req_irq_msix_misc(pf) :

→ice_alloc_vsis(pf) : vsi들을 devm_kcalloc()으로 할당해주고 있음.

→ice_init_pf_sw(pf) :

→ice_pf_vsi_setup()

→ice_vsi_setup() ==(ice/ice_lib.c)==

→ice_vsi_cfg() ==(ice/ice_lib.c)==

→ice_vsi_cfg_def() ==(ice/ice_lib.c)==

[[→ice_vsi_alloc_def()]]

⇒ vsi의 타입에 따라 irq_handler가 정의되고 있는 함수임. ==(ice/ice_lib.c)==

→ice_vsi_alloc_stat_arrays(vsi)

⇒vsi의 링의 상태를 나타내는 포인터의 실제 공간을 정의하는 함수임.

→ice_vsi_get_qs(vsi)

⇒큐를 pf에서 vsi로 할당해주는 함수

→ice_vsi_init()

vsi→type에 따라 달라짐.

→ice_vsi_alloc_q_vectors() ==(ice/ice_base.c)==

→ice_vsi_alloc_q_vector()

→ice_alloc_irq(pf, vsi→irq_dyn_alloc) ==(ice/ice_irq.c)==

return이 msi_map임.

→ice_get_irq_res() ⇒ irq_tracker에서 lowest possible index를 할당해줌. (msi-x에서 할당하는 irq Resource)

→dyn_alloc에 따라 pci_msix_alloc_irq_at() 혹은 pci_irq_vector() 등으로 msi_map을 구성함.

⇒ index는 msi 인덱스이고, virq는 연관된 리눅스 인터럽트 번호임. virq 번호는 vsi의 base address에 저장된 irq로부터 index만큼 떨어져 있다고 보면 됨. 연속적인 인터럽트 번호를 가지는 것임.

→netif_napi_add(vsi→netdev, &q_vector→napi, ==ice_napi_poll==) **//ice driver에서 사용하는 ice_napi_poll이라는 함수의 포인터를 전달하여 napi를 추가하는데 해당 함수포인터를 매핑하는 역할을 함. 그런데 로드 될 때는 netdev가 없으므로 실행되지 않음. 따라서 앞에 if(vsi→netdev)가 붙게 됨.**

→ice_vsi_alloc_rings()

→ice_vsi_alloc_ring_stats()

→ice_vsi_map_rings_to_vectors() (PF일 때만 해당)

→ice_init_wakeup(pf) :

→ice_init_link(pf) :

→ice_send_version(pf) :

→ice_verify_cacheline_size(pf) :

→ice_load() ==(ice/ice_main.c)==

→ice_napi_add()

vsi안에 모든 q_vector에 대하여,

→netif_napi_add(vsi→netdev, &vsi→q_vectors[v_idx]→napi, ==ice_napi_poll==) ==(include/linux/netdevice.h)==

→netif_napi_add_weight(dev, napi, poll, NAPI_POLL_WEIGHT==(=default는 64)==) ==(net/core/dev.c) napi initialize==

→__ice_q_vector_set_napi_queues() ==(ice/ice_lib.c)==

---

- 코드
    
    ```C
    /**
     * __ice_q_vector_set_napi_queues - Map queue[s] associated with the napi
     * @q_vector: q_vector pointer
     * @locked: is the rtnl_lock already held
     *
     * Associate the q_vector napi with all the queue[s] on the vector.
     * Caller indicates the lock status.
     */
    void ___ice_q_vector_set_napi_queues(struct ice_q_vector *q_vector, bool locked)
    {
    	struct ice_rx_ring *rx_ring;
    	struct ice_tx_ring *tx_ring;
    
    	ice_for_each_rx_ring(rx_ring, q_vector->rx)
    		__ice_queue_set_napi(q_vector->vsi->netdev, rx_ring->q_index,
    				     NETDEV_QUEUE_TYPE_RX, &q_vector->napi,
    				     locked);
    
    	ice_for_each_tx_ring(tx_ring, q_vector->tx)
    		__ice_queue_set_napi(q_vector->vsi->netdev, tx_ring->q_index,
    				     NETDEV_QUEUE_TYPE_TX, &q_vector->napi,
    				     locked);
    	/* Also set the interrupt number for the NAPI */
    	netif_napi_set_irq(&q_vector->napi, q_vector->irq.virq);
    }
    ```
    

> napi와 연관된 큐들을 매핑하는 함수. 여기서 net_device에 멤버로 있는 netdev_rx_queue와 netdev_queue에다가 해당 napi를 할당하게 된다. 그런데 이것은 기존의 ice_rx_ring과는 또 다른 Rx_queue이다.

→ice_init_devlink()

  

  

---

→ice_init_pf_sw()

→ice_pf_vsi_setup()

→ice_vsi_setup() ==여기서부터 ice/ice_lib.c로 넘어감==

→ice_vsi_cfg()

→ice_vsi_cfg_def()

→ice_vsi_alloc_def() → vsi의 타입에 따라 irq_handler가 정의되고 있는 함수임. ==(ice/ice_lib.c)==

함수 호출을 역추적함.

  

---

→ice_vsi_cfg_lan()

→ice_vsi_cfg_rxqs()

→ice_vsi_cfg_rxq()