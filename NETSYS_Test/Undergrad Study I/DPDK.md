---
creator: Maike
type: Glossary
created: 2022-06-28
---
### What is DPDK?

**D**ata **P**lane **D**evelopment **K**it

The DPDK is a set of **libraries** and **drivers** that enable high-speed packet processing in user space by **bypassing the kernel**.
![[Untitled(38).png]]


### What is DPDK _not?_

DPDK is not a replacement for the Linux kernel network stack.

### What does DPDK need to run?
![[Untitled(39).png]]


1. Create EAL **E**nvironment **A**bstraction **L**ayer

What is it and why do we need it? DPDK deals with some low-level system-specific stuff. Its jobs include: - DPDK Loading and Launching - Core Affinity/Assignment Procedures - System Memory Reservation - Trace and Debug Functions - Utility Functions - CPU Feature Identification - Interrupt Handling - Alarm Functions _(List taken from the [official DPDK documentation](http://doc.dpdk.org/guides/prog_guide/env_abstraction_layer.html#environment-abstraction-layer) )_

```
The EAL abstracts those tasks based on the system it runs on, so that DPDK does not have to care about the specifics and remains portable.
```

2. Insert Module of Choice Three choices: IGB UIO, VFIO, and KNI
    
    In short: The V in VFIO stands for virtual, so this is a virtualization option. (Everything you need to know and more is [here](https://www.kernel.org/doc/Documentation/vfio.txt)) The K in KNI stands for kernel, so this is your option if you need some kernel service support (ifconfig etc.) (Explanation in [docs](https://doc.dpdk.org/guides/prog_guide/kernel_nic_interface.html))
    
    IGB UIO is what we usually use. Explanations about it are kind of scarce.
    
    > IGB UIO is a DPDK kernel module which deals with PCI enumeration and handles links status interrupts in user mode, instead of being handled by the kernel.
    
    [Source](https://www.intel.com/content/www/us/en/developer/articles/guide/data-plane-development-kit-dpdk-getting-started.html)
    
3. Allocate Hugepages DPDK needs to have its own version of DMA-able memory. Storing packets needs space.
    

But why hugepages? Don’t we already have normal pages?
![[Untitled(40).png]]


1. Connect Device of Choice to DPDK
    
    DPDK needs to directly communicate with the NIC This job is usually performed by the NIC driver, but drivers are usually managed by the kernel. Solution? Unbind devices from the kernel to the DPDK.
    
    But. How does one unbind a device from the kernel when devices are controlled through the kernel? Devices are identifiable by their physical PCIe address. Use that info to unbind the device from the kernel and bind it to the DPDK.
    
    Friendly Note: Whichever device you bind to the DPDK, it **will** disappear from the kernel’s view.
    

### So, what does DPDK do now?

DPDK essentially does what the normal device driver would have done: It receives the data from the NIC and transports and buffers it in user space.

More specifically, packets are transported into ring buffers provided by DPDK.

> The ring structure provides a lockless multi-producer, multi-consumer FIFO API.

Important Piece of Information:

DPDK uses polling instead of interrupts. DPDK wants high performance and interrupts just create too much overhead for that. So if you run DPDK, know that that CPU core will do nothing else.

The only good and understandable reference I found 🥲:

[https://blog.selectel.com/introduction-dpdk-architecture-principles/](https://blog.selectel.com/introduction-dpdk-architecture-principles/)