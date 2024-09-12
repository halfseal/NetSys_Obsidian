---
creator: 김명수
type: Hands-On
created: 2022-07-19
---
<<[[eBPF Code (bcc)|previous]]>>
# **🧐**

# BPF Internals

### Calling convention

**x64 Architecture :** [https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/x64-architecture](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/x64-architecture)

**eBPF instruction set :** [https://www.kernel.org/doc/html/latest/bpf/instruction-set.html](https://www.kernel.org/doc/html/latest/bpf/instruction-set.html)

**bpf design Q&A :** [https://www.kernel.org/doc/html/latest/bpf/bpf_design_QA.html#q-is-bpf-a-generic-instruction-set-similar-to-x64-and-arm64](https://www.kernel.org/doc/html/latest/bpf/bpf_design_QA.html#q-is-bpf-a-generic-instruction-set-similar-to-x64-and-arm64)

- BPF has stack space(maximum 512 bytes)
- BPF has 11 registers and each register has own purpose(r0~r10)
- r0 contains return value of helper function call.
- r1~r5 contain arguments
- r6~r10 are callee saved registers
- BPF registers can map one to one to HW CPU registers
![[Untitled(81).png]]

![[Untitled(82).png]]


Most of bpf instructions can mapped one-to-one to native instructions

⇒ **low overhead**
![[Untitled(83).png]]


## ⇒ BPF registers are not real HW

## XDP + BPF

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

static inline int is_icmp(void *data, u64 nh_off, void *data_end)
{
    struct iphdr *iph = data + nh_off;

    if((void*)&iph[1] > data_end)
        return 0;
    if(iph->protocol == IPPROTO_ICMP){
        bpf_trace_printk("DROP!!!\\\\n");
        return 1;
    } 
    return 0;
}

int xdp_hook(struct xdp_md *ctx)
{
    void* data = (void*)(long)ctx->data;
    void* data_end = (void*)(long)ctx->data_end;
    uint32_t key = 0;
    struct ethhdr *eth = data;
    long *value;
    uint64_t nh_off;
    nh_off = sizeof(*eth);
    if(data + nh_off > data_end){
         return XDP_DROP;
    }

    if(eth->h_proto == htons(ETH_P_IP)){
        if( is_icmp(data, nh_off, data_end) ){
            bpf_trace_printk("DROP at look_info()\\\\n");
            value = cnt.lookup(&key);
            if(value)
                *value+=1;
            return XDP_DROP;
        }
    }
    return XDP_PASS;
}
"""

b = BPF(text=bpf_text, cflags=["-w"])

in_fn = b.load_func("xdp_hook", BPF.XDP)
b.attach_xdp("ens33", in_fn, 0)
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

-_my_xdp_drop.py_

To excute : sudo ./my_xdp_drop.py
![[Untitled(84).png]]


If packet is ICMP, increase cnt value by 1 and return _**“XDP_DROP”**_

else return _**“XDP_PASS”**_

If there is no boundary check, verifier return false
![[Untitled(85).png]]


```
               *-/uapi/linux/bpf.h*
```
![[Untitled(86).png]]

![[Untitled(87).png]]

```
               *-/uapi/linux/if_ether.h*
```
![[Untitled(88).png]]


Check _**iph→protocol**_ to determine whether the packet is icmp or not
![[Untitled(89).png]]


```
               *-/uapi/linux/ip.h*
```
![[Untitled(90).png]]


```
               *-/uapi/linux/in.h*
```
![[Untitled(91).png]]

![[Untitled(92).png]]


ping 8.8.8.8 doesn’t work while other networks(website) work well 😲

### Additional….
![[Untitled(93).png]]

![[Untitled(94).png]]


134744072 → 00001000 00001000 00001000 00001000 → 8.8.8.8

1677764800 → 01100100 00000000 10101000 11000000 → 100.0.168.192

⇒ Maybe Little Endian/Big Endian…?