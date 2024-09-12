---
creator: 구성현
type: Paper Review
created: 2022-07-05
---
<<[[Paper XDP Summary Cntd.|next]]>>

<aside> 👀 소프트웨어에서 패킷 프로세싱은 커널을 bypass 하는 것임 → bypass하면 시간이 덜든다는 장점과 함께 app을 기능적으로 다시 구현하거나 보안의 문제가 있음 → 고로 논문에서 XDP를 소개함

- 어떻게 고침? → 커널의 네트워크 스택을 처리하기 직전에 eBPF 코드를 사용하는 XDP를 둠
- 뭘로 구성되어있는데? → XDP driver hook + eBPF virtual machine + BPF map + eBPF verifier
- 각각이 뭐하는데? → XDP driver hook은 커널 들어가기 context object를 보고 어디로 보낼 것인지 결정함 → eBPF virtual machine은 프로그램을 유동적으로 로딩하거나 재로딩함 → BPF map은 eBPF 프로그램들이나 user space와의 소통을 지원 → eBPF verifier는 프로그램이 커널에 harm하는 행동을 하지 않도록 ensure
- 그래서 뭐가 좋아짐? → 뒷부분은 안읽음

</aside>

## ABSTRACT

---

> Programmable한 패킷 프로세싱이 커널을 bypass하는 기술로 구현되는 방법이 많아졌다. OS를 bypass함에 따라서 app의 isolation과 security machanism이 따로 필요하게 되었는데 이 한계를 극복하기 위해 논문은 eXpress Data Path라는 XDP를 소개한다. XDP는 리눅스 커널 mainline의 한 부분이며 커널의 네트워크 스택과 완전히 통합된 해결책을 제공한다.

- Programmable packet processing is increasingly implemented using kernel bypass techniques
- As the OS is bypassed, so are its application isolation and security mechanisms
- To over come this limitation, paper present eXpress Data Path(XDP)
- XDP is part of the mainline Linux kernel and provides a fully integrated solution working in concert with the kernel’s networking stack.
- XDP achieves single-core packet processing performance as high as 24 million packets per second
- the flexibility of the programming model through three example use cases: layer-3 routing, inline DDoS protection and layer-4 load balancing

## INTRODUCTION

---

> software에서 High-perform 패킷 프로세싱은 패킷을 프로세싱하는 시간의 tight bound를 요구한다. DPDK는 완전히 OS를 bypass한다. 이렇게 커널을 bypass하는 접근은 단점이 있는데, 현존하는 시스템과 통합하기 어렵다는 것과 app이 functionality를 re-implement해야한다는 것이 있다. 이는 시스템 복잡성과 보안의 경계를 흐리게 한다, 커널 bypass 디자인의 대안으로 XDP가 있는데, XDP는 eBPF 코드를 실행하는 가상 머신의 형태에서 한정된 실행환경으로 정의된다. eBPF는 커널이 패킷 데이터를 건드리기 전에 커널 context에서 custom된 프로그램을 실행한다. XDP의 high-level design description과 능력은 다음과 같다.

- High-performance packet processing in software requires very tight bounds on the time
- Data Plane DevelopmentKit (DPDK) bypass the operating system completely
- The kernel bypass approach has the drawback
    - difficult to integrate with the existing system
    - applications have to re-implement functionality
- This results in increased system complexity and blurs security boundaries
- Alternative to the kernel bypass design : eXpress Data Path (XDP)
    - works by defining a limited execution environment in the form of a virtual machine running eBPF code
    - eBPF executes custom programs directly in kernel context before the kernel itself touches the packet data
- High-level design description of XDP and its capabilities
    - Integrates cooperatively with the regular networking stack, retaining full control of the hardware in the kernel
        - retains the kernel security boundary
        - no changes to network configuration and management tools
        - any network adapter with a Linux driver can be supported by XDP
    - selectively utilize kernel network stack features
    - guarantees stability of both the eBPF instruction set and the programming interface (API) exposed along with it
    - not require expensive packet re-injection from user space into kernel space
    - dynamically re-programmed without any service interruption
    - not require dedicating full CPU cores to packet processing
        - lower traffic levels translate directly into lower CPU usage

## THE DESIGN OF XDP

---

Figure 1 : XDP’s integration with the Linux network stack

> 패킷이 도착하면, 디바이스 driver는 main XDP hook에서 eBPF 프로그램을 실행한다. 이 프로그램은 패킷이 어디로 갈지 결정한다.

- On packet arrival, the device driver executes an eBPF program in the main XDP hook
    - can choose to drop packets
    - to send them back out the same interface
    - to redirect them to another interface or to userspace through special AF_XDP sockets
    - to allow them to proceed to the regular networking stack
![[Untitled(54).png]]


- The XDP driver hook is the main entry point for an XDP program, and is executed when a packet is received from the hardware
- The eBPF virtual machine executes the byte code of the XDP program, and just-in-time-compiles it for increased performance
- BPF maps are key/value stores that serve as the primary communication channel to the rest of the system
- The eBPF verifier statically verifies programs before they are loaded to make sure they do not crash or corrupt the running kernel

### The XDP Driver Hook

Figure2 : Execution flow of a typical XDP program
![[Untitled(55).png]]


> 패킷이 하드웨어에 도착하고 커널이 sk_buff를 할당하기 직전 실행된다. context object에 접근하면서 시작된다. context object는 raw 패킷 데이터를 가리키는 포인터를 포함하고 인터페이스와 receive q의 정보를 담고 있는 메타데이터도 포함한다. 먼저 패킷 데이터를 parsing하고 다른 XDP 프로그램에 넘긴다. context object를 이용하여 메터데이터 필드를 읽고 커널의 helper func을 이용해 커널 함수를 사용한다. 어떤 시스템과 소통할것인지 mapping한다. 패킷 데이터의 부분을 적고 마지막으로 패킷 데이터에 대해 verdict한다.

- Executed at the earliest possible moment
- Starts its execution with access to a context object
    - contains pointers to the raw packet data
    - with metadata fields describing which interface and receive queue
    - also gives access to a special memory area, located adjacent in memory to the packet data
- begins by parsing packet data and pass control to a different XDP program
    - splitting processing into logical sub-units
- Use the context object to read metadata fields
- Define and access its own persistent data structure
- Access kernel facilities through various helper functions
    - selectively make use of existing kernel functionality
- Maps allow the program to communicate with the rest of the system
- Write any part of the packet data
    - to perform encapsulation or decapsulation
    - for instance, rewrite address fields for forwarding
- Final verdict for the packet
    - drop the packet
    - immediately re-transmit it out the same network interface
    - to be processed by the kernel networking stack
    - redirect the packet
        - transmit the raw packet out a different network interface
        - pass it to a different CPU
        - pass it directly to a special userspace socket address family AF_XDP

### The eBPF Virtual Machine

> 기존 BPF 가상 머신은 2개의 32 bit 레지스터를 가지고 있고 22의 다른 instruction을 이해할 수 있다. eBPF는 레지스터 수를 11개로 증가시켰고 레지스터 크기를 64비트, 새로운 instruction을 추가했다. eBPF 가상 머신은 dynamic하게 프로그램을 로딩하고 재로딩하며 커널이 모든 프로그램의 라이프 사이클을 관리한다.

- The original BPF virtual machine has two 32-bit registers and understands 22 different instruction
- eBPF extends the number of registers to eleven and increases register widths to 64 bits, also adds new instructions to the eBPF instruction set
- The eBPF virtual machine supports dynamically loading and re-loading programs
    - kernel manages the life cycle of all programs

### BPF Maps

> eBPF 프로그램 persistent한 메모리 영역에 접근할 수 없고 대신에, 커널 helper function을 이용하여 BPF map에 접근할 수 있다.

- eBPF programs do not have access to persistent memory storage
- Instead, the kernel exposes helper function giving programs access to BPF maps
    - key/value stores that are defined upon loading an eBPF program
    - exist in both global and per-CPU variants
    - can be shared
        - between different eBPF programs
        - between eBPF and userspace
    - map types include generic hash map, arrays and radix trees

### The eBPF Verifier

> eBPF 프로그램이 로딩될 때, 처음으로 in-kernel eBPF verifier에 대해 분석된다. eBPF verifier는 프로그램이 unsafe한 action을 취할 수 없도록 ensure한다.

- When loading an eBPF program, first analyzed by the in-kernel eBPF verifier
- ensure that the program performs no actions that are unsafe

### Summary

- The XDP device driver hook
    - run directly after a packet is received from the hardware
- The eBPF virtual machine
    - responsible for the actual program execution
- BPF maps
    - for programs to communicate with each other and with user space
- The eBPF verifier
    - ensures program do not perform any harmful operation to kernel

## More …

---

- XDP driver hook에서 pass to stack이 걸리면 기존 커널에서 packet processing이랑 어떤 차이가 있죠?
    
    ⇒ 똑같음. XDP는 existing system과 compatible한다는 점에서 개선이 있음