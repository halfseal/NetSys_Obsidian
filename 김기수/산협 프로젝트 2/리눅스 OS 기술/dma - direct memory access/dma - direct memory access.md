[https://www.kernel.org/doc/Documentation/DMA-API-HOWTO.txt](https://www.kernel.org/doc/Documentation/DMA-API-HOWTO.txt) - 커널 공식 API 가이드 (Eng)

[https://yohda.tistory.com/entry/리눅스-커널-DMA-overview](https://yohda.tistory.com/entry/리눅스-커널-DMA-overview)

커널은 주로 virtual address를 사용한다. kmalloc()(물리,연속,빠름,까다로움), vmalloc()(가상) 등의 함수로 할당 할 수 있으며, void * 안에 저장할 수 있다.

TLB나 pagetables등은 virtual → physical로 translate을 하며, 이를 phys_addr_t 혹은 resource_size_t 등의 변수에 저장한다.

커널은 device의 자원을 physical address로써 registers처럼 관리한다.

[[이러한 주소록은 -proc-iomem에 있다]]

물리적인 주소는 드라이버가 바로 사용할 수는 없는데, 반드시 ioremap() 함수를 통해 미리 공간을 매핑하고 새로운 virtual address를 만들어내야 할 것이다.

I/O device들은 3번째 종류의 address를 사용하는데, 바로 “bus address”이다.

만약 device가 mmio address에 레지스터를 가지고 있거나, 시스템 메모리를 읽고 쓰기 위해 DMA가 작동 중이라면, device가 사용하는 address가 바로 bus address가 될 것이다. 몇몇 시스템들은 bus address가 CPU의 physical address와 동일 하지만, 보편적으로는 아니다.

[[MMIO]]

따라서 IOMMU와 host bridge가 physical address와 bus address간에 임의의 매핑을 만들어내게 된다. ==IOMMU는 bus to physical, host bridge는 physical to bus==

device의 입장에서 DMA는 bus address space를 사용하는데, 그러한 space의 subset으로 사용이 제한 될 것이다. 예를 들어, 64bit address를 지원하는 시스템이더라도 iommu를 사용함으로써 32bit dma address만을 필요로 할 것이다.

==BAR : Base Address Register, iommu를 통해 32bit DMA address를 64bit physical address로 변환할 수 있기 때문에 DMA address를 64비트를 전부 사용하지 않아도 된다.==

![[Untitled 6.png|Untitled 6.png]]

virtual address는 프로세스가 보는 가상의 메모리 주소 공간이고, bus address는 디바이스가 보는 가상의 메모리 주소 공간이다. 모두 실제하는 공간은 없으며, 물리 메모리에 매핑되어 접근하게 된다.

bus address (A)에서 physical address (B)로 변환되고, 만약 driver의 요청이 있다면, ioremap() 함수를 통해 physical address (B)가 virtual address (C)에 매핑된다.

이제서야 ioread32(C)와 같은 함수를 통해 bus address (A)에 있는 device register에 접근할 수 있다.

만약 device가 DMA를 지원한다면, driver는 버퍼를 kmalloc(), 혹은 비슷한 인터페이스로 buffer를 셋업하게 된다. (이 때 return은 virtual address (X))

Mmu(virtual memory system)은 (X)를 physical address (Y)에다가 매핑할 것이다.드라이버는 이제 X를 통해 버퍼에 접근할 수 있다. 하지만 device 그 자신은 접근할 수 없는데, DMA는 CPU virtaul memory system을 통해서 작동하지 않기 떄문이다.

몇몇 간단한 시스템들은 device가 physical address (Y)로 바로 향하는 DMA를 할 수 있다.

그러나 대부분은 IOMMU hardware가 존재하고, 이는 DMA address를 physical address로 바꾸게 된다.(Z to Y)

이는 DMA API를 위한 이유중 하나인데, 예를 들어 필요한 IOMMU mapping을 하고 DMA address Z를 반환하는 dma_map_single()과 같은 interface에 virtual address (X)를 driver가 제공할 수 있게 한다.

그럼 driver는 device의 (Z)에게 DMA를 하라고 말할 것이고, IOMMU는 그것을 (Y) 주소의 buffer에 매핑할 것이다.

**⇒ driver가 buffer를 셋팅해서 virtual addr를 받고, 이를 DMA API를 통해 해당 physical addr에 iommu를 통해 매핑된 bus addr를 반환 받을 수 있다.**

위와 같은 방법으로 리눅스는 dynamic DMA mapping을 할 수 있다. 물론 driver의 몇몇 도움이 필요할 것이고, 바꿔 말하면 그것이 실제로 사용되는 시간에만 DMA address가 매핑되고, DMA transfer가 끝난 다음에는 mapping이 해제 된다는 것을 고려해야 할것이다.

DMA API들은 해당하는 hardware API가 없더라도 작동할 것이다.

DMA API는 어떤 마이크로프로세서 아키텍쳐와도 상관없이 독립적으로 어느 bus와도 작동한다.

bus-specific DMA API 보다 이걸 사용 할 것을 반드시 권장한다.

“dma_addr_t”은 DMA address를 담을 수 있는 타입으로, DMA address를 사용하거나 이를 반환 값을 받는 어떤 곳에서든 반드시 써야 한다.

dma를 사용할 수 있는 핵심 조건은 1. 연속된 물리 메모리 주소 와 2. 캐시라인 정렬 이다.

device의 bus address가 32bit인지 64bit인지 모르므로, DMA addressing capablility를 kernel에게 알려줘야 할 것이다.

따라서 DMA mask를 사용하는데, dma_set_mask_and_coherent() 함수를 호출한다.

[[streaming mappings vs consistent allocations →(coherent)]]

  

high memory : kernel virtual address

low memory : kernel logical address → physical address를 base address와 offset으로 접근함.