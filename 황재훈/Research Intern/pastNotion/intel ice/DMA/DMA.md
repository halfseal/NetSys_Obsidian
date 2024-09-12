  

CPU and DMA addr (physical, virtual, bus addr)

- kernel은 virtual addr 사용
- kmalloc(), vmalloc()에서 리턴되는 주소는 virtual addr 이고, void * 에 저장된다
- virtual addr 는 physical addr로 전환되고 이는 phys_addr_t 나 resource_size_t에 저장된다
- physical addr 만으로는 쓸모가 없다. 반드시 `ioremap()`을 통해 virtual addr로 mapping 해야한다
- I/O device 는 bus addr를 사용한다
- MMIO addr 에 register가 있거나 DMA를 할 경우, bus addr를 사용한다
- IOMMU와 host bridge가 임의로 bus addr를 physical addr와 매핑한다

  

MMIO (mapped mem I/O) IOMMU (input output mem management unit)

![[황재훈/Research Intern/pastNotion/intel ice/DMA/img/Untitled.png|Untitled.png]]

- pci device에 BAR가 있으면 kernel이 그곳에서 bus addr를 읽고 그것을 physical addr로 전환한다
- 이후 driver가 연결할 때, ioremap()을 통해서 physical addr를 virtual addr로 전환한다
- MMIO : device register가 system의 address space와 mapping되어 있는것 (physical addr 사용)
    - cpu가 device에 접근가능
- IOMMU : bus addr를 physical addr로 바꿔준다 (addr translation)

  

1. **Device Enumeration and MMIO Mapping**:
    - 시스템이 켜지면 커널이 IO device와 MMIO 공간을 확인한다
    - pci device의 경우, BAR (base addr register) 안의 bus addr를 읽는다
    - kernel이 이 bus addr를 physical addr로 바꾼다
    - physical addr는 ioremap()을 통해 virtual addr로 바꿔진다
    - 이제 driver는 virtual addr로 자유롭게 device에 접근할 수 있다
2. **Setting Up a DMA Buffer**:
    - The driver allocates a buffer using `kmalloc()`, which returns a virtual address **X**.
    - The virtual memory system maps this virtual address **X** to a physical address **Y**.
    - driver가 버퍼에 kmalloc()으로 공간을 할당한다. 이는 virtual addr를 리턴한다
    - virtual addr는 physical addr로 mapping된다
3. **DMA Operation Without IOMMU**:
    - IOMMU가 없는 경우, device가 dma를 위해 physical addr 사용할 수 있다
4. **DMA Operation With IOMMU**:
    - IOMMU가 있는 경우, bus addr 사용한다
    - `dma_map_single()` 사용, 이는 bus addr를 리턴한다