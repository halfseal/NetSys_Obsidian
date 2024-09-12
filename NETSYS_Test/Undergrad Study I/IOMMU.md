---
creator: 황준용
type: Glossary
created: 2022-05-03
---


**What is IOMMU?**
![[Untitled(23).png]]


The Input-Output Memory Management Unit (IOMMU) is a **component in a memory controller that translates device virtual addresses** (can be also called I/O addresses or device addresses) to physical addresses.

Similar to MMU(Memory Management Unit).

**IOMMU vs MMU**
![[Untitled(24).png]]


IOMMU translates **device virtual addresses** to physical addresses, while MMU translates **CPU virtual addresses** to physical addresses.

A MMU maps virtual memory addresses to physical memory address. While the **normal MMU is used to give each process its own virtual address space, the IOMMU is used to give each I/O device its own virtual address space.** That way, the I/O device sees a simple contiguous address space, possibly accessible with 32 bit addresses while in reality the physical address space is fragmented and extends beyond 32 bit.

**Why IOMMU?**

In a virtualization environment, the I/O operations of I/O devices of a guest OS are translated by they hypervisor (software-based I/O address translation). **This behavior results in a negative performance impact.** The IOMMU is a **hardware component that performs address translation from I/O device virtual addresses to physical addresses**. This hardware-assisted I/O address translation dramatically improves the system performance within a virtual environment.
![[Screenshot_20220510-130056_Adobe Acrobat.jpg]]


- Emulation Model (Hypervisor-based device emulation)
    
    The hypervisor needs to manipulate the interaction between the guest OS and the associated physical device. It implies that the hypervisor translates device address (from device-visible virtual address to device-visible physical, and vice versa), which requires more CPU computation power and impacts the system performance when heavy I/O occurs.
    
- Pass Through Model
    
    The hypervisor does not need to deploy the dedicated software for emulating the physical device and translating device address. This model improves the system performance by means of a hardware-assisted component.
    

**What is DMA?**

_DMA (direct memory access)_  is a hardware feature that allows memory access to occur independently of the program currently run by the micro processor. It can either be used by I/O devices to directly read from or write to memory without executing any micro processor instructions. Or, it can be used to efficiently copy blocks of memory. During DMA transfers, the micro processor can execute an unrelated program at the same time.

**IOMMU Functionalities**

1. DMA remapping functionality manipulates address translation for PCI devices
2. Interrupt remapping functionality routes interrupts of PCI devices to the corresponding guest OSes.

**IOMMU Pros and Cons**

Pros

1. Large regions of memory can be allocated without the need to be contiguous in physical memory
2. Devices that do not support memory addresses long enough to address the entire physical memory can still address the entire memory through the IOMMU, avoiding overheads associated with copying buffers to and from the peripheral's addressable memory space.
3. Memory is protected from malicious devices that are attempting DMA attacks and faulty devices that are attempting errant memory transfers because a device cannot read or write to memory that has not been explicitly allocated (mapped) for it.

Cons

1. Some degradation of performance from translation and management overhead (e.g., page table walks).
2. Consumption of physical memory for the added I/O page tables. This can be mitigated if the tables can be shared with the processor.

**IOMMU Subsystem in Linux Kernel**
![[Screenshot_20220510-130659_Adobe Acrobat.jpg]]


TBD

**Linux Kernel IOMMU: DMA Translation Mode versus Pass-through Mode**

TBD