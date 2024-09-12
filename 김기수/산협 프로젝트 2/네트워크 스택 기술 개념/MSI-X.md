[https://www.kernel.org/doc/html/v5.8/PCI/msi-howto.html](https://www.kernel.org/doc/html/v5.8/PCI/msi-howto.html)

Message Signaled Interrupt - device가 spectial address에 쓰기를 하여 인터럽트를 일으킬수 있게 하는 것.

- 사용하는 이유 3가지
    1. Pin-based PCI 인터럽트는 자주 몇몇 device들끼리 공유하게 된다. 따라서 커널이 이를 지원하려면 이와 관련된 모든 interrupt handler를 모두 실행해야 하므로 성능하락이 동반된다.
    2. device가 data를 메모리에 쓰고 나서 pin-based interrupt를 일으키면, 데이터가 메모리에 도착하기도 전에 인터럽트가 도착하는 가능성이 있다. 따라서 이를 보장하기 위해 interrupt handler는 반드시 인터럽트를 일으킨 device의 레지스터를 읽어야할 것이다. PCI 전송 정렬은 레지스터에 반환되는 값 이전에 메모리에 모든 데이터가 도착해야한다는 규칙이 필요할 것이다.
    3. 또한 함수당 하나의 pin-based interrupt를 지원하는데 드라이버는 자주 device에게 어떤 이벤트가 일어났는지 물어보게 되고, 이는 대채적으로 interrupt handling에서 느려지게 하는 원인이다.

PCI device들은 pin-based interrupt로 초기화 되기 때문에, device의 드라이버가 MSI를 사용할 수 있도록 set up 해야 한다.

pci_alloc_irq_vectors를 통해 셋팅 가능함.

특정 메모리 주소에 특정 데이터를 쓰는 방식으로 이루어지게 됨. 이 메모리에 쓰기를 하는 동작 자체가 인터럽트를 트리거 함.

  

---

msi_desc 구조체 - MSI 기반 인터럽트들을 위한 Descriptor 구조체이다.

~/include/linux/msi.h에 있다.

irq 번호, 사용된 벡터의 갯수, dev 포인터, msg 구조체 등이 있다.

msi_msg는 address_lo, address_hi, data 변수들이 u32타입으로 각각 선언되어 있다.

  

이러한 msi 도메인은 xarray라는 저장소에 저장되고, 다시 불러와진다 xarray는 key-value값이 저장되며, 인덱스와 키 모두 접근 가능한 배열이다. 리눅스 커널에서 효율적이고 scalable한 데이터 구조를 사용하기 위해 설계된 자료 구조이며, 멀티쓰레드 환경에서 동시 접근을 효율적으로 처리하며 높은 성능을 유지한다. 배열과 해시테이블의 장점을 결합하여 설계되어 있다.

[https://kernel.bz/boardPost/118679/19](https://kernel.bz/boardPost/118679/19)

  

각 장치마다 기본 MSI-X 주소가 있다. 여기서 각 인터럽트 벡터마다 nr이라는 offset을 통해 해당하는 MSI-X 인터럽트 주소를 얻을 수가 있다.

만약 msi를 사용한다고 하면 nr이라는 오프셋이 없이 그냥 irq 번호만 반환하게 된다. 이 경우 multiple interrupts에 대하여 하나의 descriptor만 가지고 있다는 뜻이다. msi-x는 인터럽트당 하나의 descriptor를 가질 수 있다는 뜻이다.

msi_domain_get_virq() 함수에서 xarray에서 msi_desc 정보를 꺼내오고, msi-x의 경우 index를 더하여 해당하는 linux interrupt number를 반환하는 것을 볼 수 있다.

==~/kernel/irq/msi.c==