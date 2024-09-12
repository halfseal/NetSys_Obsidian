---
creator: 황준용
type: Hands-On
created: 2022-09-26
---
<<[[Kernel Launch Process Cntd.|next]]>>
# Kernel Launch Process

## GPU Scheduling on the NVDIA TX2: Hidden Details Revealed

### 1-1. Needs of the Paper

- Mass-market vehicles evolve towards providing full autonomy
- Ever more critical that GPU-using workloads are amenable to strict certification.
- Many aspects underlying their design either are not documented / described at such a high level that crucial technical details are not revealed.

→ Present an in-depth study of GPU scheduling.

### 1-2. Exampler : NVDIA TX2
![[Untitled(164).png]]
- Single-board computer containing multiple CPU and integrated GPU (Share DRAM with host CPU Platform)
- Marketed for “autonomous everything”, sharing GPU architecture with PX2
- provides significant computing capacity
- Meets reasonable limits on monetary cost as well as size, weight, and Power (SWaP)

### 1-3. Introduction

- NVDIA CPU Scheduling differs from:
    1. work is submitted to a CPU from a CPU executing OS threads that share an address space
    2. work is submitted to a CPU from a CPU executing OS processes that have different address space

### 2-1. Background : CUDA Programming

- CUDA: API for CPU Programming by NVDIA
![[Untitled(169).png]]
- Typical structure of CUDA:
    1. Allocate GPU-local memory for data
    2. Use GPU to copy data from host memory to GPU device memory
    3. launch kernel (≠ OS kernel) to run on the CPU cores to compute function
    4. Use the CPU to copy output data from device memory back to host memory.
- Input data is partitioned among hardware threads by:
    1. number of threads in a thread block
    2. total number of thread block
- 128 core per Streaming Multiprocessors (total 256 core), each SM runs four units (warp) of 32 threads each using warp scheduler
- → hide memory latency via warp scheduler
- Warp Scheduler

### 2-2. Background : Ordering and Executions of GPU operations
![[Untitled(166).png]]
- Figure is distinguished in:
    1. block time: time to execute a single thread block
    2. kernel time: time to execute single kernel
    3. total time : “”
- In CUDA, GPU operations (copy + kernel)can be ordered by associating them with a stream.
- operation within a stream are executed in FIFO
- order across different streams can run concurrently, else is determined by the GPU scheduling policy. (can be out of launch-time) →discussed in section 3
- By default GPU operations from CPU are inserted into a default stream called “NULL STREAM”. Programmers can create/use other streams too.
- Kernel launches are asynchronous with respect to CPU program.
- By default copy operations are synchronous, yet can be asynchronous. (still have to wait for kernel launches to finish)
- → All asynchronous operations requires CPU to wait until all CPU operations are done, which is provided by CUDA synchronization / event manager

> For a simpler abstraction, programmers can think of a GPU as consisting of one or more copy engines (CEs) (the TX2 has only one) that copy data between host memory and device memory, and an execution engine (EE) that has one or more SMs (the TX2 has two) consisting of many cores that execute GPU kernels. An EE can execute multiple kernels simultaneously, but a CE can perform only one copy operation at a time. EEs and CEs can operate concurrently.

### 3. Basic GPU Scheduling Rules

Under assuming: GPU is accessed only by CPU tasks that share address space using user-specified streams.

Experiment:
![[Untitled(167).png]]
- Running instances of a synthetic benchmark that launches several kernels
    
    → Used synthetic workload allows flexibility in configuring block resource requirements, kernel duration and copy operations.
    
    → variety of real-world GPU workloads work in same manner.
    
    1. Each block of benchmark kernel spin for one second.
    2. Three kernels K1, K2, K3 were launched by a single task to a single stream, and K4, K5, K6 were launched by a second task to a two separate stream. (launched by #)
    3. Copy operations occurred after K2 and K5, before and after K3, and after K6, in their respective streams.
- Arrows depict kernel launch time
    

### 3-1. GPU Scheduling Rules

- GPU Schedulers determine which thread blocks can be scheduled at any given time.
    
- Impacted by availability of limited GPU resources that blocks utilize such as GPU shared memory, registers, and threads
    
- Required resources are determined for the entire kernel when it is launched
    
- All blocks in a given kernel have the same resource requirements.
    
- When block is scheduled to SM, it is _assigned_
    
- when at least one of the block is assigned, kernel is _dispatched_
    
- When all block is assigned, kernel is _fully dispatched_
    

General Scheduling Rules:

3 queues are used:

1. FIFO EE queue per address space
2. FIFO CE queue to order copy operations to the GPU’s CE
3. FIFO queue for CUDA streaming (stream queue)

- **G1** A copy operation or kernel is enqueued on the stream queue for its stream when the associated CUDA API function (memory transfer or kernel launch) is invoked.
- **G2** A kernel is enqueued on the EE queue when it reaches the head of its stream queue.
- **G3** A kernel at the head of the EE queue is dequeued from that queue once it becomes fully dispatched.
- **G4** A kernel is dequeued from its stream queue once all of its blocks complete execution
![[Untitled(168).png]]
![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/88211a30-d482-459c-b469-d19649d79388/Untitled.png)

explain (a) and (b)tls_registry_

1. 찾아보니 cont_func는 “continuous function”으로 그냥 continue를 할 때 어떤 행동을 할 지 알려주는 function이네요.

아까 보여드린 코드처럼 단순하고 로그를 찍을수도 있고, 상황에 맞춰 재각각 구현할 수 있습니다.

아래처럼 테스트를 위해 num_cont_func_calls_를 늘려줘서 카운팅할 수도 있습니다.

```cpp
static void cont_func(void *_context, void *) {
  auto *context = static_cast<RpcTest *>(_context);
  context->num_cont_func_calls_++;
}
```

2. tls_registry_의 경우 nexus에 private 변수로 정의되어있습니다.

```cpp
TlsRegistry tls_registry_;     ///< A thread-local registry
```

그리고 아까 말했듯이, 아래 코드처럼 background thread 마다 init을 부를 때 cur_etid 값을 inc. 하는 것을 확인할 수 있습니다.

```cpp
#include "tls_registry.h"
#include "common.h"

namespace erpc {

thread_local bool tls_initialized = false;
thread_local size_t etid = SIZE_MAX;

void TlsRegistry::init() {
  assert(!tls_initialized);
  tls_initialized = true;
  etid = cur_etid_++;
}

void TlsRegistry::reset() {
  tls_initialized = false;
  etid = SIZE_MAX;
  cur_etid_ = 0;
}

size_t TlsRegistry::get_etid() const {
  assert(tls_initialized);
  return etid;
}

}  // namespace erpc
```