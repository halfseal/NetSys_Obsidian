---
creator: 황준용
type: Paper Review
created: 2022-07-07
---
_The eXpress Data Path: Fast Programmable Packet Processing in the Operating System Kernel_

Fast Programmable Packet Processing in the Operating System Kernel

[02-The express Data Path.pdf](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/31b60a9f-105c-43f0-8370-5506666b99d9/02-The_express_Data_Path.pdf)

# Problem

---

Programmable packet processing is increasingly implemented using kernel bypass techniques (DPDK)

_**However, as the operating system is bypassed, so are its application isolation and security mechanisms; and well-tested configuration, deployment and management tools cease to function.**_

But, Necessary to remove any bottlenecks between the NIC and the program performing the packet processing

—> One of the main bottlenecks is: _Interface between the operating system kernel and userspace applications running on top of it_

## Solution

---

_**eXpress Data Path(XDP),**_ where the operating system kernel itself provides a safe execution environment for custom packet processing applications, executed in the device driver context.

Adds programmability directly in the operating system networking stack in a cooperative way. This makes it possible to perform high-speed packet processing that integrates seamlessly with existing systems, while selectively leveraging functionality in the operating system.

## How?

---

### Key Innovation

1. _**The use of a virtual execution environment**_ verifies that loaded programs will not harm or crash the kernel by being supported by the kernel community, thus offering the same API stability guarantee as every other interface the kernel exposes to userspace.
2. _**Can completely bypass the networking stack**_, which offers higher performance than a traditional kernel module that needs to hook into the existing stack.
3. _**Also allows the programs loaded into the kernel to selectively redirect packets to a special user-space socket type, which bypasses the normal networking stack**_, and can even operate in a zero-copy mode to further lower the overhead.
4. _**Programmable hardware devices**_ to achieve high-performance packet processing. XDP can be thought of as a “software offload”, where performance-sensitive processing is offloaded to increase performance, while applications otherwise interact with the regular networking stack.

### Design

✏️ TODO

## Advantage and Evaluation

---

### Advantage

1. vs DPDK:
    
    1. Retains the advantage of removing the kernel-userspace boundary between networking hardware and packet processing code
    2. Keeps the kernel in control of hardware, thus preserving the management interface and the security guarantees offered by the OS.
2. Integrates cooperatively with the regular networking stack, retaining full control of the hardware in the kernel.
    
3. Makes it possible to selectively utilize kernel networks stack features such as the routing table and TCP stack
    
4. guarantees stability of both the eBPF instruction set and the programming interface (API) exposed along with it
    
5. Does not require expensive packet re-injection from user space into kernel space
    
6. dynamically re-programmed without any service interruption
    
7. Does not require dedicating full CPU cores to packet processing
    
    —> lower traffic levels translate directly into lower CPU usage
    

### Evaluation

✏️ TODO

## Questions

---

- [x] Routing table & TCP stack in kernel bypass

> Makes it possible to selectively utilize kernel network stack features such as the _**routing table and TCP stack**_, keeping the same configuration interface while accelerating critical performance paths.

- Routing Table
    ![[Untitled(71).png]]
    
    
    **A routing table is a table or database that stores the location of routers based on their IP addresses**. **This table acts as an address map to various networks,** and is usually stored in the RAM of most routers or forwarding devices. As such, a routing table contains information about various networks, and how to get to them.
    
    [https://www.baeldung.com/cs/routing-table-entry](https://www.baeldung.com/cs/routing-table-entry)
    
- TCP Stack
    ![[Untitled(72).png]]
    
    
    TCP/IP is frequently referred to as a "stack." This refers to **the layers (TCP, IP, and sometimes others) through which all data passes at both client and server ends of a data exchange**.
    
- How DPDK works without using the Routing Table
    
    _**Unanswered**_
    
- [x] Common Off The Shelf (COTS) hardware
    

> To achieve high packet processing performance on _**Common Off The Shelf (COTS) hardware**_, it is necessary to remove any bottlenecks between the networking interface card (NIC) and the program performing the packet processing

- COTS
    
    **Commercial off-the-shelf** or **commercially available off-the-shelf** (**COTS**) products are ackaged or canned (ready-made) hardware or software, which are adapted aftermarket to the needs of the purchasing organization, rather than the commissioning of custom-made, or [bespoke](https://en.wikipedia.org/wiki/Bespoke), solutions.
    
- [x] NetFPGA
    

> One example is the NetFPGA [32], which exposes an API that makes it possible to _run arbitrary packet processing tasks on the FPGA-based dedicated hardware_.

- NetFPGA
    
    The **NetFPGA** project[[1]](https://en.wikipedia.org/wiki/NetFPGA#cite_note-1) is an effort to develop [open-source hardware](https://en.wikipedia.org/wiki/Open-source_hardware) and software for the [rapid prototyping](https://en.wikipedia.org/wiki/Rapid_prototyping) of [computer network](https://en.wikipedia.org/wiki/Computer_network) devices.
    
    ~~Arbitrary packet processing tasks?~~
    
- [x] The interface between the operating system kernel and userspace applications running on top of it
    

> Since one of the main sources of performance bottlenecks is the interface between the operating system kernel and the userspace applications running on top of it (because of the high overhead of a system call and complexity of the underlying feature-rich and generic stack), low-level packet processing frameworks have to manage this overhead in one way or another.

- 1. Context Switch vs Mode switch
    ![[Untitled(73).png]]
    
    
- 1. High overhead of System call
    
    Whenever mode switching occurs, the following happens.
    
    1. Save the status of the current processor.
        
    2. Set the PC (program counter) to the appropriate location.
        
    3. Go to Kernel mode and issue commands that require permission.
        
    
    4.When switching modes, only a part of the PCB needs to be stored, and the image does not need to be organized. In any case, there is an overhead for storing-restoring in steps 1 and 4.
    
    *A process control block (PCB) is a data structure used by computer operating systems to store all the information about a process.
    
    **—> What kind of system call is done in packet processing?**
    
- 2. Complexity of underlying feature-rich generic Stack
- [x] Management Interface and security guarantees done where?
    

> _**Can completely bypass the networking stack**_, which offers higher performance than a traditional kernel module that needs to hook into the existing stack.

- Management Interface and security guarantees
    
    “Applications are written in higher-level languages such as C and compiled into custom byte code which the kernel statically analyses for safety, and translates into native instructions “
    ![[Untitled(74).png]]
    
    
- [ ] Virtual execution environment
    

> _**The use of a virtual execution environment**_ verifies that loaded programs will not harm or crash the kernel by being supported by the kernel community, thus offering the same API stability guarantee as every other interface the kernel exposes to userspace.

—> Virtual execution system?