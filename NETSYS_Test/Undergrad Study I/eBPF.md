---
creator: 김명수
type: Glossary
created: 2022-06-28
---
## Extended Berkeley Packet Filter

pakcet filtering : filtered, matched → packet accept or deny

but, not only filtering packet

- Virtual machine in kernel
- Run program with kernel
- Observability, tracing

# Why?

- Run program with kernel
- Additional capabilities to kernel at runtime without changing kernel source code
- Event-Driven
    - event ⇒ action
    - event : kernel function(kprobes), userspace function(uprobes), system calls…

Originally

To test….
![[Untitled(50).png]]


1. insert “printf” code in kernel code
2. re-compile entire kernel code
3. delete “printf” code after testing
4. re-compile entire kernel code

When using eBPF
![[Untitled(49).png]]


## Process

### Make eBPF program

1. eBPF code written in C
2. compile
3. bytecode of eBPF program
![[Untitled(48).png]]


### Load eBPF program

Load eBPF program into kernel using bpf() syscall

1. verifiy eBPF program (check whether it is safe or not)
2. translate bytecode to machine code
![[Untitled(47).png]]


# Architecture

### Map

- Share data, information between eBPF < - > eBPF and eBPF < - > user program
- Key-value store
- Keep program state
![[Untitled(46).png]]


- man bpf()
![[Untitled(45).png]]

![[Untitled(44).png]]

![[Untitled(43).png]]


[https://github.com/torvalds/linux/blob/master/samples/bpf/sock_example.c](https://github.com/torvalds/linux/blob/master/samples/bpf/sock_example.c)
![[Untitled(42).png]]


[https://github.com/iovisor/bcc](https://github.com/iovisor/bcc)

### Helper function

- For interact with kernel
- map_lookup, update / manipulate network pkts
- [https://manpages.ubuntu.com/manpages/focal/man7/bpf-helpers.7.html](https://manpages.ubuntu.com/manpages/focal/man7/bpf-helpers.7.html)

# Direction

- XDP : programmable packet processing, custom packet processing
- fast packet processing
![[Untitled(41).png]]
