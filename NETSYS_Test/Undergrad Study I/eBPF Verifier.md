---
creator: 김명수
type: Hands-On
created: 2022-08-09
---
## To make sure of safety of the eBPF program!

## There are two phases to determine whether eBPF program is safe or not.

### First phase: DAG checking + Valid instruction checking through DFS

DAG(Directed Acyclic Graph) is a directed graph with no cycles.

⇒ Disallow loops (characteristics of CFG)

*CFG(Complete Finite Graph)

Complete graph is a graph that every pair of distinct verices is connected by a unique edge.

Finite Graph is a graph that has a finite vertices.

⇒각 vertices들이 모두 edge로 연결되어있는 그래프
![[Untitled(95).png]]


visit_insn → push_insn → insn_state[next_insn]

discovered = 0x10 explored = 0x20
![[Untitled(96).png]]


discovered → explored

Checking whether program has unreachable instructions or not

### Second phase: Checking all possible paths of DAG

⇒Disallow unsafe memory access

Simulates execution of every instruction and checks state of registers and stack

🤯

- Every registers have a type and it can be changed by instructions
    
    ```jsx
    type ex)  PTR_TO_CTX, PTR_TO_MAP_VALUE, PTR_TO_STACK, SCALAR_VALUE….
    
    Ex) BPF_REG_1 : PTR_TO_CTX, BPF_REG_10 : PTR_TO_STACK
    
    		BPF_MOV64_REG(BPF_REG_1, BPF_REG_10)
    
    => BPF_REG_1 : PTR_TO_STACK
    
    ```
    
    - r0 contains return value of helper function call.
    - r1~r5 contain arguments which are originally unreadable
    - r6~r9 are callee saved registers
    - r10 is bpf stack pointer which is read-only
- Try to read unreadable register is rejected
    

### Register value tracking with bounds checking

⇒Tracking range of possible values of register and stack

ex) *(ptr + 3) ⇒ is it safe??

### Packet access

[](https://www.notion.so/eBPF-bcc-6ec613a2079a417ca81bcf9b89c80c65?pvs=21)[https://www.notion.so/mingman/eBPF-bcc-6ec613a2079a417ca81bcf9b89c80c65](https://www.notion.so/mingman/eBPF-bcc-6ec613a2079a417ca81bcf9b89c80c65)

```jsx
1:  r4 = *(u32 *)(r1 +80)  /* load skb->data_end */
2:  r3 = *(u32 *)(r1 +76)  /* load skb->data */
3:  r5 = r3
4:  r5 += 100
5:  if r5 > r4 goto pc+16
6:  r0 = *(u16 *)(r3 +80) /* access 12 and 13 bytes of the packet */
```
![[Untitled (1).png]]

![[Untitled(97).png]]


At the beginning, the states of each registers are

```jsx
r1 = PTR_TO_CTX
r4 = PTR_TO_PACKET_END
r3 = PTR_TO_PACKET(id = 0, offset = 0, r = 0)
```

**id** value is the number of variables which is added to that register

**offset** value is constant value which is added to that register

**r** value is range of accessible bytes

After line 5(at line 6), the states of each registers are

```jsx
r3 = PTR_TO_PACKET(id = 0, offset = 0, r = 14)
r5 = PTR_TO_PACKET(id = 0, offset = 14, r = 14)
     *r5 means “skb→data + 14”*
```

Accessible range of r3 is [r3, r3 + 14)

Accessible range of r5 is [r5, r5 + 14(r) - 14(offset))

So, this code ensures that r3 has at least accessible 14bytes

⇒Need “bound checking codes” for memory access in bpf program!!

```python
#!/usr/bin/python

from bcc import BPF
import pyroute2
import time
import sys
import ctypes as ct

bpf_text = """ 
#include <uapi/linux/bpf.h>
#include <linux/in.h>
#include <linux/ip.h>
#include <linux/if_ether.h>
#include <linux/types.h>

BPF_PERCPU_ARRAY(cnt, long, 1);

int xdp_hook(struct xdp_md *ctx)
{
    void* data = (void*)(long)ctx->data;
    void* data_end = (void*)(long)ctx->data_end;
    uint32_t key = 0;
    struct ethhdr *eth = data;
    long *value;
    uint64_t nh_off;
    nh_off = sizeof(*eth);
    if(data + 5> data_end ){

        return XDP_DROP;
    }
    char * ac = data;   
    bpf_trace_printk("%d\\\\n", ac[0]);
    bpf_trace_printk("%d\\\\n", ac[1]);
    bpf_trace_printk("%d\\\\n", ac[2]);
    bpf_trace_printk("%d\\\\n", ac[3]);
    bpf_trace_printk("%d\\\\n", ac[4]);
    bpf_trace_printk("%d\\\\n", ac[5]); /* illegal mem access */

    return XDP_PASS;
}
"""

b = BPF(text=bpf_text, cflags=["-w"])

in_fn = b.load_func("xdp_hook", BPF.XDP)
b.attach_xdp("ens33", in_fn,0)
cnt = b.get_table("cnt")
print("hook start")
while 1:
    try:
        val = cnt.sum(0).value
        print("total dropped pkt:",val)
        try:
            (task, pid, cpu, flags, ts, msg) = b.trace_fields()
        except ValueError:
            continue
        print(msg)
        time.sleep(1)
    except KeyboardInterrupt:
        print("Remove hook")
				break
b.remove_xdp("ens33", 0)

```

_my_mem_access.py_

```jsx
bpf: Failed to load program: Permission denied
0: (b7) r0 = 1
1: (61) r2 = *(u32 *)(r1 +4)
2: (61) r6 = *(u32 *)(r1 +0)
3: (bf) r1 = r6
4: (07) r1 += 5
5: (2d) if r1 > r2 goto pc+50
 R0_w=inv1 R1_w=pkt(id=0,off=5,r=5,imm=0) R2_w=pkt_end(id=0,off=0,imm=0) R6_w=pkt(id=0,off=0,r=5,imm=0) R10=fp0
6: (b7) r7 = 680997
7: (63) *(u32 *)(r10 -4) = r7
8: (71) r3 = *(u8 *)(r6 +0) /* mem access */
9: (67) r3 <<= 56
10: (c7) r3 s>>= 56
11: (bf) r1 = r10
12: (07) r1 += -4
13: (b7) r2 = 4
14: (85) call bpf_trace_printk#6
last_idx 14 first_idx 0
regs=4 stack=0 before 13: (b7) r2 = 4
15: (63) *(u32 *)(r10 -4) = r7
16: (71) r3 = *(u8 *)(r6 +1) /* mem access */
17: (67) r3 <<= 56
18: (c7) r3 s>>= 56
19: (bf) r1 = r10
20: (07) r1 += -4
21: (b7) r2 = 4
22: (85) call bpf_trace_printk#6
last_idx 22 first_idx 15
regs=4 stack=0 before 21: (b7) r2 = 4
23: (63) *(u32 *)(r10 -4) = r7
24: (71) r3 = *(u8 *)(r6 +2) /* mem access */
25: (67) r3 <<= 56
26: (c7) r3 s>>= 56
27: (bf) r1 = r10
28: (07) r1 += -4
29: (b7) r2 = 4
30: (85) call bpf_trace_printk#6
last_idx 30 first_idx 15
regs=4 stack=0 before 29: (b7) r2 = 4
31: (63) *(u32 *)(r10 -4) = r7
32: (71) r3 = *(u8 *)(r6 +3) /* mem access */
33: (67) r3 <<= 56
34: (c7) r3 s>>= 56
35: (bf) r1 = r10
36: (07) r1 += -4
37: (b7) r2 = 4
38: (85) call bpf_trace_printk#6
last_idx 38 first_idx 31
regs=4 stack=0 before 37: (b7) r2 = 4
39: (63) *(u32 *)(r10 -4) = r7
40: (71) r3 = *(u8 *)(r6 +4) /* mem access */
41: (67) r3 <<= 56
42: (c7) r3 s>>= 56
43: (bf) r1 = r10
44: (07) r1 += -4
45: (b7) r2 = 4
46: (85) call bpf_trace_printk#6
last_idx 46 first_idx 31
regs=4 stack=0 before 45: (b7) r2 = 4
47: (63) *(u32 *)(r10 -4) = r7
48: (71) r3 = *(u8 *)(r6 +5) /* mem access */
invalid access to packet, off=5 size=1, R6(id=0,off=0,r=5)
R6 offset is outside of the packet /* error */
processed 49 insns (limit 1000000) max_states_per_insn 0 total_states 3 peak_states 3 mark_read 1
```
![[Untitled(98).png]]

![[Untitled(99).png]]
