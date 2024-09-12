**ice probe (struct pci_dev * pdev, const struct pci_device_id __always_unused * ent)**

1. virtual function 인지 확인
2. kdump (panic 상태에서 종료) 인지 확인
3. pci device 켜기
4. BAR (base addr register) 에 할당된 구역을 mapping
5. `ice_allocate_pf` ?
6. DMA mask 설정하기
7. pci device 설정하기
8. hw 설정, device 설정
9. ==`ice_load()`====, devl_lock??==
10. `devlink()`?

  

[[황재훈/Research Intern/pastNotion/intel ice/ice probe()/pci_dev]]

- bus의 list
- bridge 되어 있는 bus

[[pci_device_id]]

- vendor and device ID, 또는 PCI_ANY_ID
- subvendor, subdevice 서브 시스템의 id, PCI_ANY_ID

  

- pdev는 probe되는 device 를 의미한다
- PCI : local computer bus standard, cpu와 mem과 소통, peripheral component link
- pf : physical function, nic와 관련된 물리적 정보를 가지고 있다
- hw : hardware, 디바이스와 관련된 정보를 가지고 있다

  

```Plain
if (pdev->is_virtfn) {
	dev_err(dev, "can't probe a virtual function\\n");
	return -EINVAL;
}
```

- pdev 가 virtual function인지 확인
    - vf이면 probe 못해서 error
- virtual function
    - vf는 probing이 필요없

  

```Plain
if (is_kdump_kernel()) {
	pci_save_state(pdev);
	pci_clear_master(pdev);
	err = pcie_flr(pdev);
	if (err)
		return err;
	pci_restore_state(pdev);
}
```

- ==kdump_kernel???==
    
    - kernel이 이전 kernel의 panic이후에 실행되는 것인지 체크하는 과정
    - kernel이 panic으로 종료하게 되면 elfcore 주소를 저장하는데, 이게 존재하는 확인한
    - kdump_kernel이면 pci 의 상태를 저장
    - pci 초기화??
    - error 가 발생할 경우, 원래의 상태로 복귀
    - pci_clear_master(pdev) : pdev 장치를 정지시켜서 data transaction을 금지한다
    - pcie_flr(pdev) : pci를 flr (Fuction Level Rest) 한다
    - pci_restore_state(pdev) : pdev의 상태를 복원시킨다
    
      
    
    ```Plain
    err = pcim_enable_device(pdev);
    if (err)
    	return err;
    
    err = pcim_iomap_regions(pdev, BIT(ICE_BAR0), dev_driver_string(dev));
    if (err) {
    	dev_err(dev, "BAR0 I/O map error %d\\n", err);
    	return err;
    }
    
    pf = ice_allocate_pf(dev);
    if (!pf)
    	return -ENOMEM;
    ```
    
- pcim_enable_device
    - device register를 건들기 전에 driver는 pci device를 call 해서 실행해야한다
    - device를 깨우고
    - I/O 와 mem 구역을 할당한다 (bios가 하지 않은 경우에)
    - IRQ를 할당한다 (bios가 하지않은 경우에)
- pcim_iomap_regions
    - pci의 mem를 kernel의 virtual address space와 mapping한
    - mask가 가르키는 구역을 request하고 iomap한다
    - mask: 어떤 BAR(base addr register) 를 매핑할지 알려주는 bitmask

[[ice_allocate_pf]]

- ice_pf 의 메모리를 할당하고 초기값들을 설정한다
- ==devlink?==

[[황재훈/Research Intern/pastNotion/intel ice/ice probe()/ice_pf|ice_pf]]

- pci_dev
- devlink_region
- pci를 실행
- ==physical function에 devlink_priv()? 함수를 할당==

  

```Plain
/* set up for high or low DMA */
err = dma_set_mask_and_coherent(dev, DMA_BIT_MASK(64));
if (err) {
	dev_err(dev, "DMA configuration failed: 0x%x\\n", err);
	return err;
}
```

- dma의 설정을 device에 맞게 바꾸는 듯??
- device가 dma 할 때 사용할 수 있는 addr 의 범위를 설정한다

  

```Plain
pci_set_master(pdev);

pf->pdev = pdev;
pci_set_drvdata(pdev, pf);
set_bit(ICE_DOWN, pf->state);
/* Disable service task until DOWN bit is cleared */
set_bit(ICE_SERVICE_DIS, pf->state);

hw = &pf->hw;
hw->hw_addr = pcim_iomap_table(pdev)[ICE_BAR0];
```

- pci_set_master : bus mastering을 활성화해서, data transaction을 가능하게 한다
    - pci command register에서 bus master enable bit을 설정한다
- pci_set_drvdata : device의 특정한 data를 pci에 설정한다
- pcim_iomap_table(pdev) : iomap이 설정된 virtual addr 의 목록을 가져온다

  

```Plain
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
```

- pci의 상태를 저장
- hw의 설정값들을 pci device에서 가져온다

  

```JavaScript
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
```

- ice_init(pf)
- devl_lock(pf)
- ice_load(pf)
- devl_unlock(priv_to_devlink(pf))
- ice_init_devlink(pf)