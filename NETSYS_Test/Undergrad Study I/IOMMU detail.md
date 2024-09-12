---
creator: 구성현
type: Glossary
created: 2022-06-14
---

### IOMMU (Input/Output Memory Management Unit, 입출력 메모리 관리장치)

- Hardware I/O Virtualization : 하드웨어 기능으로 가상머신이 이용하는 I/O device를 가상화하는 기법
    
    I/O data 전달 : **IOMMU** & Data emulation : I/O device의 SR-IOV
    
    < TMI : I/O Software virtualization은 Device emulation, I/O data 전달 과정에 의한 overhead 🤢 >
    
![[Untitled(30).png]]


가상머신은 Physical addr을 보는 반면 실제 DMA Controller는 Host addr을 기준으로 수행한다는 차이가 있어 IOMMU를 도입함

IOMMU는 CPU의 North-Bridge에 위치하여 I/O device가 보는 Memory 주소를 변환

↔ MMU는 CPU core에 위치하여 CPU에서 보는 Memory 주소를 변환하는 장치

[I/O Virtualization Hardware](https://ssup2.github.io/theory_analysis/IO_Virtualization_Hardware/)

- IOMMU Page Walk
    ![[Untitled(31).png]]

    
    PCI device의 Device number + Function number(in PCI device) → Page walk
    
    Memory 주소 → Page walk
    
    ⇒ 총 2단계의 Page Walk !
## IOMMU (Input/Output Memory Management Unit)
![[Untitled(32).png]]


MMU : CPU에서 보는 Memory 주소를 변환하는 장치 ↔ IOMMU : IO Device가 보는 Memory 주소를 변환하는 장치
![[Untitled(33).png]]


### Q. Virtualization 과의 연관성?
![[Untitled(34).png]]


가상화 환경에서 Guest OS가 DMA를 이용할 수 있게 해줌

일반적인 경우는 쓸 일이 거의 없고, 가상화 환경에서 성능을 향상시킬 수 있다는 장점 때문에 가상화 환경에서 많이 쓰이는 것으로 생각됨. ( 단독으로 쓰는 것은 못찾았습니다 )

## An Introduction to IOMMU Infrastructure in the Linux Kernel

[pdf](https://lenovopress.lenovo.com/lp1467.pdf)

### PCIe Device Virtualization Models

Emulation model(Hypervisor-based device emulation)

VMM needs to manipulate the interaction between the guest OS and the associated physical device

Pass-through model(IOMMU)

is bypassed for the interaction between the guest OS and the physical device
![[Untitled(35).png]]

![[Untitled(36).png]]


1. Device driver invokes `pci_map_page()` or `dma_map_page()` API to get the physical address
2. If the DMA request is directly mapped, DMA subsystem returns the calculated physical address to device driver directly.
3. not directly mapped, the DMA request is forwarded to IOMMU subsystem
4. DMA subsystem invokes iommu_dma_map_page() to request IOMMU subsystem to map virtual address to physical address
5. IOMMU subsystem maps virtual address to physical address and configures the corresponding I/O page table
    1. IOMMU DMA Layer: DMA request를 받아서 IOMMU generic layer로 전달
    2. IOMMU Generic Layer : hardware specific IOMMU랑 소통하는 IOMMU API 제공
    3. Hardware Specific IOMMU later : hardware와 소통하는, IO page table 구성하는 layer
![[Untitled(37).png]]


Pass-through Mode : when DMA address == system physical address

## Virtual Memory

1. Concept
    
    A technique that allows the execution of processes that are not completely in memory.
    
    - Partition each user’s program into multiple blocks
    - Load into memory the blocks that is necessary at each time during execution
        - Noncontiguous allocation
    - Logical memory size is not constrained by the amount of physical memory that is available
2. Methods
    
    1. Paging
    2. Segmentation
    3. Hybrid paging/segmentation