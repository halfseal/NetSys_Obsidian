---
creator: 황준용
type: Glossary
created: 2022-10-24
---
<<[[Kernel Launch Process|previous]]>>
# Kernel Launch Process

## GPU Scheduling on the NVDIA TX2: Hidden Details Revealed

### 1-1. Needs of the Paper

- Mass-market vehicles evolve towards providing full autonomy
- Ever more critical that GPU-using workloads are amenable to strict certification.
- Many aspects underlying their design either are not documented / described at such a high level that crucial technical details are not revealed.

→ Present an in-depth study of GPU scheduling (validation of real-time constraints)

- In embedded multicore + GPU platform, usually use SOC, which are less capable than those in desktop system

→ Crucial to fully utilize thee GPU processing capacity that is available

done documents + black-box experiment.

### 1-2. Exampler : NVDIA TX2
![[Untitled(206).png]]


- Single-board computer containing multiple CPU and integrated GPU (Share DRAM with host CPU Platform)
- Marketed for “autonomous everything”, sharing GPU architecture with PX2

1. provides significant computing capacity
2. Meets reasonable limits on monetary cost as well as size, weight, and Power (SWaP)

- 2 streaming multiprocessors (SMs)

### 1-3. Introduction

- NVDIA CPU Scheduling differs from whether work is submitted to a GPU from:
    1. A CPU executing OS **threads** that share an address space
        1. Ordering within and among streams for requests for GPU operation
        2. Ability of the GPU to co-schedule operations
        3. Selection mechanism to determine the order requests are handled
        4. resource limits that constrains the GPU ability
    2. A CPU executing OS **processes** that have different address space
        1. less deterministic

### 2-1. Background : CUDA Programming

- **CUDA**: API for CPU Programming by NVDIA
- **GPU** : a co-processor that performs operations requested by CPU Code.
![[Untitled(207).png]]


- Typical structure of CUDA:
    
    1. Allocate GPU-local memory for data
    2. Use GPU to copy data from host memory to GPU device memory
    3. launch kernel (≠ OS kernel) to run on the GPU cores to compute function
    4. Use the GPU to copy output data from device memory back to host memory.
    5. free device memory
- Input data is partitioned among hardware threads on GPU by:
    
    1. number of threads in a thread block
        
    2. total number of thread block
        
        —> given in kernel-launch command.
        
- On TX2, 128 core per Streaming Multiprocessors (total 256 core), each SM runs 4 units (warp) of 32 threads each using warp scheduler
    
- Warp Schedulers take advantage of stalls, such as waiting to access memory, to immediately begin running a different warp.
    
    → hide memory latency via warp scheduler
    

Thus, we can think of GPU consisting:

1. one or more copy engines (CEs) that copy data between host-device memory
2. Execution memory (EE) that has one or more SMs to execute GPU kernels.

→ EE can execute multiple kernels simultaneously, but CE can perform only one copy operation at a time.

→ CE and EE can operate concurrently.

### 2-2. Background : Ordering and Executions of GPU operations
![[Untitled(208).png]]


- Figure is distinguished in:
    1. block time: time to execute a single thread block
    2. kernel time: time to execute single kernel
    3. total time : time to execute entire CUDA program (copy not depicted)
- In CUDA, GPU operations (copy + kernel)can be ordered by associating them with a **stream**.
- operation within a stream are executed in **FIFO** , order across different streams is determined by the **GPU scheduling policy**
- order across different streams can run concurrently, else is determined by the GPU scheduling policy. (can be out of launch-time) →discussed in section 3
- By default GPU operations from CPU are inserted into a default stream called “NULL STREAM”. Programmers can create/use other streams too.
- Kernel launches are asynchronous with respect to CPU program.
- By default copy operations are synchronous, yet can be asynchronous with CPU program. (still have to wait for kernel launches to finish)

### 3. Basic GPU Scheduling Rules

Under assuming: GPU is accessed only by CPU tasks that share address space using user-specified streams.

Experiment:
![[Untitled(209).png]]


- Running instances of a synthetic benchmark that launches several kernels
    
    1. Each block of benchmark kernel spin for one second.
    2. Three kernels K1, K2, K3 were launched by a single task to a single stream, and K4, K5, K6 were launched by a second task to a two separate stream. (launched by #)
    3. Copy operations occurred after K2 and K5, before and after K3, and after K6, in their respective streams.
- Arrows depict kernel launch time
    

### 3-1. GPU Scheduling Rules

- **GPU Schedulers determine which thread blocks can be scheduled at any given time.**
    
- Impacted by availability of limited GPU resources that blocks utilize such as GPU shared memory, registers, and threads
    
- Required resources are determined for the entire kernel when it is launched
    
- All blocks in a given kernel have the same resource requirements.
    
- When block is scheduled to SM, it is _assigned_
    
- when at least one of the block is assigned, kernel is _dispatched_
    
- When all block is assigned, kernel is _fully dispatched_
    

**General Scheduling Rules:**

3 queues are used:

1. FIFO EE queue per address space (= EE stream)
2. FIFO CE queue to order copy operations to the GPU’s CE (= CE stream)
3. FIFO queue for CUDA streaming (= stream queue)

- **G1** A copy operation or kernel is enqueued on the stream queue for its stream when the associated CUDA API function (memory transfer or kernel launch) is invoked.
- **G2** A kernel is enqueued on the EE queue when it reaches the head of its stream queue.
- **G3** A kernel at the head of the EE queue is dequeued from that queue once it becomes fully dispatched.
- **G4** A kernel is dequeued from its stream queue once all of its blocks complete execution
![[Untitled(210).png]]

![[Untitled(211).png]]


2 CUDA programs executing as CPU tasks t0 and t1 on CPU 0 and 1 (share address space)

t0 use single stream, t1 use two streams

2 SM single CE/EE stream are depicted.

ex (a)

t0 : issued K1,K2,K3 and C20, C3i, and C3o

t1: issued K4, K5 to stream S2 and S3, copy C5o to stream S3.

1. G1에 의거 CUDA 순서대로 enqueue
2. G2에 의거 EE에 순서대로 enqueue
3. K1 dispatch라서 shaded
4. SM 별로 K1의 block 하나씩 나눠가짐
![[Untitled(212).png]]


ex (b)

1. K1과 K4의 remaining blocks들이 dipsatch 되었고, fully dispatch 되었으므로 EE Queue에서 제외
2. 하지만 Stream Queue에는 남아있는다 (blocks complete execution이 아니므로)

**Non-preemptive execution**

The kernel at the head of the EE queue will non-preemptively block later-queued kernels:

- **X1** Only blocks of the kernel at the head of the EE queue are eligible to be assigned.
![[Untitled(213).png]]


ex (a)

K4의 block들이 K1의 block과 함께 execute할 수 있지만, K4가 EE의 head에 있지 않기 때문에 assign되기에 부합하지 않다.

**Rules governing thread resource**

Under TX2, thread count of all block is limited to 2048 per SM / each block is limited to 1024 per SM. These resource limit can delay the kernel at the head of EE queue:

- **R1** A block of the kernel at the head of the EE queue is eligible to be assigned only if its resource constraints are met.
- **R2** A block of the kernel at the head of the EE queue is eligible to be assigned only if there are sufficient thread resource available on some SM.

ex) (a) & (b)
![[Untitled(214).png]]


768 * 2 = 1536

→ X1과 used together!

**Rules governing shared-memory resources**

On TX2, shared memory is limited to 64KB per SM and 48KB per block.

- **R3** A block of kernel at the head of the EE queue is eligible to be assigned only if there are sufficient shared-memory resource available on some SM.
![[Untitled(215).png]]


ex (b) & (c)

Although there are available thread on SM1 , no block of K5 is assigned untill the blocks of K4 is complete.

**Register Resources**

On TX2, each thread can use up to 255 registers, and each block can use up to 32,768 registers (regardless of its thread count (≠ 255*2048=522240**)**), and each SM can use up to 65,536 registers

Unfortunately, using synthetic kernels makes it difficult to demonstrate limits on registers as NVIDIA compiler optimizes register usage

→ Decrease the number of registers used by kernel makes its block easier to schedule at the expense of potentially greater execution time?

**Copy operation**

TX2 has one CE to process both host-to-device / device-to-host copies.

- **C1** A copy operation is enqueued on the CE queue when it reaches the head of its stream queue
- **C2** A copy operation at the head of the CE queue is eligible to be assigned to the CE
- **C3** A copy operation at the head of the CE queue is dequeued from the CE queue once the copy is assigned to the CE on the GPU
- **C4** A copy operation is dequeued from its stream queue once the CE has completed the copy
![[Untitled(216).png]]


ex (d)

Once K2 and K5 complete execution and dequeued from S1 and S3,

1. C5o and C2o are enqueued to CE Queue (C1)
2. C5o is immediately enqueued to CE (C2)
3. C5o is dequeued from CE queue (C3)
4. Only one CE, so C2o remains at CE queue.

Yet, additional complexity occurs in:

- **NULL Stream**
- **Stream priorities**
- **Multiple Processors**