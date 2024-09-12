---
creator: 김명수
type: Hands-On
created: 2022-07-05
---
<<[[eBPF(bcc) Deeper|next]]>>
# Install bcc

[https://github.com/iovisor/bcc](https://github.com/iovisor/bcc)

**BCC does not support LLVM 7 or earlier anymore with the master branch**

[https://github.com/iovisor/bcc/issues/3881](https://github.com/iovisor/bcc/issues/3881)

[https://github.com/iovisor/bcc/releases/tag/v0.24.0](https://github.com/iovisor/bcc/releases/tag/v0.24.0)

[https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#4-attach_uprobe](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#4-attach_uprobe)

[https://github.com/iovisor/bcc/blob/master/INSTALL.md](https://github.com/iovisor/bcc/blob/master/INSTALL.md)

### Install Packages

`sudo apt-get install bpfcc-tools linux-headers-$(uname -r)`

### Install Source

You need

- LLVM 3.7.1 or newer, compiled with BPF support (default=on)
- Clang, built from the same tree as LLVM
- cmake (>=3.1), gcc (>=4.7), flex, bison

```
# For Bionic (18.04 LTS)
sudo apt-get -y install bison build-essential cmake flex git libedit-dev \\
  libllvm6.0 llvm-6.0-dev libclang-6.0-dev python zlib1g-dev libelf-dev libfl-dev python3-distutils
```

### Install and compile BCC

Down load source code zip at [https://github.com/iovisor/bcc/releases/tag/v0.24.0](https://github.com/iovisor/bcc/releases/tag/v0.24.0)

```
mkdir bcc/build; cd bcc/build
cmake ..
make
sudo make install
cmake -DPYTHON_CMD=python3 .. # build python3 binding
pushd src/python/
make
sudo make install
popd
```

# Kernel probe

# -Tracing _clone_ syscall & count with maps
![[Untitled(56).png]]


[test_send.py](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/bcfbd862-43fa-4b1f-b131-93c7510cdcba/test_send.py)
![[Untitled(57).png]]


chmod a+x test_send.py

sudo ./test_send.py

# -Tracing _send, sendto, sendmsg_ & count with maps
![[Untitled(58).png]]

![[Untitled(59).png]]


# User probe

## Tracing user-level **func
![[Untitled(60).png]]

![[Untitled(61).png]]

![[Untitled(62).png]]


```python
#!/usr/bin/python
from bcc import BPF

# define BPF program
prog = """
#include <uapi/linux/bpf.h>
BPF_ARRAY(call_count, u64, 100);

int hook_send(void *ctx){
    bpf_trace_printk("SEND is called\\\\n");
    int one = 1;
    u64 *val = call_count.lookup(&one);
    if (val) 
        *val += 1;
    return 0;
}
int hook_sendto(void *ctx){
    bpf_trace_printk("SENDTO is called\\\\n");
    int two = 2;
    u64 *val = call_count.lookup(&two);
    if (val) 
        *val += 1;
    return 0;   
}
int hook_sendmsg(void *ctx){
    bpf_trace_printk("SENDMSG is called\\\\n");
    int thr = 3;
    u64 *val = call_count.lookup(&thr);
    if (val) 
        *val += 1;
    return 0;
}

int hook_clone(void *ctx){
    bpf_trace_printk("CLONE is called\\\\n");
    int fou = 4;
    u64 *val = call_count.lookup(&fou);
    if(val)
        *val += 1;
    return 0;
}
"""
# load BPF program
b = BPF(text=prog)
b.attach_kprobe(event=b.get_syscall_fnname("send"), fn_name="hook_send")
b.attach_kprobe(event=b.get_syscall_fnname("sendmsg"), fn_name="hook_sendmsg")
b.attach_kprobe(event=b.get_syscall_fnname("sendto"), fn_name="hook_sendto")
#b.attach_kprobe(event=b.get_syscall_fnname("clone"), fn_name="hook_clone")
call_count = b.get_table("call_count")
while 1:
    try:
        (task, pid, cpu, flags, ts, msg) = b.trace_fields()
    except ValueError:
        continue
    print("%-18.9f %-16s %-6d %s" % (ts, task, pid, msg))
    print("SEND is called :{0}\\nSENDTO is called :{1}\\nSENDMSG is called :{2}".format(call_count[1].value, call_count[2].value, call_count[3].value))
```

```python
#!/usr/bin/python
from bcc import BPF

# define BPF program
prog = """
int hook_rand(void *ctx){
    bpf_trace_printk("RAND is called\\\\n");
    return 0;
}

int hook_func1(void *ctx)
{
    bpf_trace_printk("FUNC1 is called\\\\n");
    return 0;
}
int hook_func2(void *ctx)
{
    bpf_trace_printk("FUNC2 is called\\\\n");
    return 0;
}
int hook_func3(void *ctx)
{
    bpf_trace_printk("FUNC3 is called\\\\n");
    return 0;
}

"""

# load BPF program
b = BPF(text=prog)

b.attach_uprobe(name="c", sym="rand", fn_name="hook_rand")
b.attach_uprobe(name="./a.out", sym="func1", fn_name="hook_func1")
b.attach_uprobe(name="./a.out", sym="func2", fn_name="hook_func2")
b.attach_uprobe(name="./a.out", sym="func3", fn_name="hook_func3")

while 1:
    try:
        (task, pid, cpu, flags, ts, msg) = b.trace_fields()
    except ValueError:
        continue
    
    print("%-18.9f %-16s %-6d %s" % (ts, task, pid, msg))
```

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <unistd.h>

void func1()
{
    printf("1");
}

void func2()
{
    printf("2");
}

void func3()
{
   printf("3");
}

int main(){
    srand(time(NULL));
    for(int i = 0; i < 5; i++){
        int funcnum = rand()  % 3;
	if(funcnum == 0){
		func1();
	}
	else if(funcnum == 1){
		func2();
	}
	else{
		func3();
	}
    }
}
```