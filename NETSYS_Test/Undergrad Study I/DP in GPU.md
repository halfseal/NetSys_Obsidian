---
creator: 김명수
type: Glossary
created: 2022-09-26
---
## Dynamic Parallelism(Kepler)

- Before DP, only CPU can launch GPU kernel
- Using DP GPU kernel can generate new kernel
![[Untitled(170).png]]


- Can generate new GPU kernel without CPU intervention

### Cons of DP

- Generating new kernel can degrade original kernel’s performance
- Dynamically generated kernels are not always efficient

# SPAWN

- Decide whether generate new child kernel or not
    
- if(performance will be better) ⇒ generate new kernel
    
- if(performance will be worse) ⇒ don’t generate new kernel
    
- Estimate “launch overhead + queuing latency + execution time” to determinie whether launch new kernel or not
    
    🤯
    
    - What is launch overhead, Where queuing latency happened?
- Can reduce launch overhead and queuing latency
    

# B.G.

[https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html)


![[Untitled(171).png]]

![[Untitled(172).png]]
## Block vs Warp?

[https://forums.developer.nvidia.com/t/difference-between-a-block-and-a-warp/7956](https://forums.developer.nvidia.com/t/difference-between-a-block-and-a-warp/7956)

So **the block is the 'scope' within which sets of threads can communicate.**  **A warp is a hardware detail which is important for performance, but less so for correctness**

A block is made up of warps. A warp is what executes on each SM at any given timestep.

1 block = 512 threads

⇒ 0~ 512 thread can communicate each others

1 warp = 32threads

⇒After SM executes 16 warps ⇒ then next block

ex) block0- warp0 ~ warp15 → block1- warp0 ~ warp15
![[Untitled (7)(2).png]]


Single application can launch several kernels

`MatAdd<<<numBlocks, threadsPerBlock>>>(A, B, C);`

## Simple logic of DP
![[Untitled(173).png]]


- Pick up local workload at each threads in parent

🤯

Maybe **cudaStreamCreateWithFlags(&c_stream, cudaStreamNonBlocking);**

is right?

- cudaStreamCreate(cudaStream_t* pStream)
    
    ⇒ Create an asynchronous stream
    
- cudaStreamCreateWithFlags(cudaStream_t* pStream, unsigned int flags)
    
- Both are creating new async stream but only “NonBlocking” do not sync with default stream.
    
    ⇒ Create an asynchronous stream
    
    **:cudaStreamDefault**
    
    :**cudaStreamNonBlocking**: Created stream run concurrently with work in stream 0(NULL stream), No implicit sync with stream 0
    
![[Untitled(174).png]]

![[Untitled(175).png]]


Host to device op is pushed into **H2D queue**

Kernel op is pushed into **compute queue**

Device to host op is pushed into **D2H queue**

[https://developer.download.nvidia.com/CUDA/training/StreamsAndConcurrencyWebinar.pdf](https://developer.download.nvidia.com/CUDA/training/StreamsAndConcurrencyWebinar.pdf)

[https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__STREAM.html#group__CUDART__STREAM_1gb1e32aff9f59119e4d0a9858991c4ad3](https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__STREAM.html#group__CUDART__STREAM_1gb1e32aff9f59119e4d0a9858991c4ad3)

만약 stream 1과 stream 2사이에 default stream 이 있으면 blocking stream일 경우에는 default stream이 끝날 때까지 sync가 맞게 된다.

### What is Launch Overhead, Queing latency?

[nvidia.com/content/dam/en-zz/Solutions/Data-Center/tesla-product-literature/NVIDIA-Kepler-GK110-GK210-Architecture-Whitepaper.pdf](http://nvidia.com/content/dam/en-zz/Solutions/Data-Center/tesla-product-literature/NVIDIA-Kepler-GK110-GK210-Architecture-Whitepaper.pdf)

**Process of kernel launching**

[https://docs.nvidia.com/pdf/Kepler_Tuning_Guide.pdf](https://docs.nvidia.com/pdf/Kepler_Tuning_Guide.pdf)

1. DP app starts to runnig in host
    
2. Generated GPU kernel is tagged with a SWQ id(stream id)
    
    1. Software managed Work Queue
    2. Child kernels with the same SWQ id execute **sequentially**(argument of launching child kernel)
    
    ---
    
3. Pushed into **Pending Kernel Pool** in GMU
    
    1. Grid Management Unit
    2. Kernels with same SWQ id are mapped into same HWQ(HW work queue)
4. CTAs from HWQ are dispatched to the GPU multiprocessor unit
    
    1. The number of maximum concurrent execution is depends on the number of HWQs.
    
    ⇒**Remained CTA in HWQ have queing latency**
    
![[Untitled(176).png]]

![[Untitled (16).png]]


Invoking **child kernel creation API** is not free but it executed as asynchronous.

Parent thread do not wait until child kernel to be launched

If all of parent CTAs are waiting for sync, parent relinquishes GPU resources.

🤯About 4-a.

⇒CUDA driver mapped streams to hardware work queues through connections, it is possible to allocate more streams than there connections, this means that the driver will alias several streams to some or all of thosse connections.
![[Untitled(177).png]]

주황색의 스트림id는 가운데 파란색의 스트림id와 동일하게 됨

> The CUDA_DEVICE_MAX_CONNECTIONS environment variable can be used to specify the preferred number of connections to be allocated to the driver. The default is 8 (or fewer if CUDA Multi-Process Service is in use); the architectural maximum for GK110 is 32.

So called _“**launch overhead**”_ is invoking the API + pushing the child kernel into Pending Kernel Pool

(child kernel이 실행되기 전까지의 준비과정)

🤯

> This launch overhead can potentially be **hidden by overlapping** the execution of other available warps on SMXs. However, in cases where a majority of running parent threads launch child kernels within a short period of time, such high number of API calls cannot be serviced simultaneously

It means during launching child kernel, other warp can be scheduled..?

And if parent kernel launch child kernel frequently, performace can be degraded?

**Observation**
![[Untitled(178).png]]


BFS_graph 500 benchmark

1. Until (i)Only the parent CTAs are running
2. From (i) to (iii), reached maximum total concurrent CTAs due to child kernels
3. From (ii) parent CTAs start to relinquish resources and more child CTAs can be executed
    1. Because child CTAs usually tend to be lightweight, resource utilization is decreased.
    2. maybe something like context switch(scheduling) will decrease the resource utilization.
4. From (iii) to (iiii),
    1. Concurrent kernel limitation due to limited number of HWQs.
        
        ⇒Not concurrent CTA limitation, because lots of child kernels has a few CTAs.
        
    2. Launch overheads…? what is trailing child kernel???
        
        > Second, the trailing child kernels have long latencies before they can start executing, resulting in system idleness due to launch overheads
        

## Insight

- Workload distribution between parent and child is the most important
- ⇒THRESHOLD

## Challenges

Which one is better?

Launch new child kernel vs Only parent(no more child)
![[Untitled(179).png]]


## Child CTA Queuing System(CCQS)

- Monitors the launched kernels and give information to SPAWN controller
- Regard GMU as **queue**, CTAs as **job** and SMXs as a **server**
- Arrival rate of the jobs is denoted by λ : spawning rate of CTAs from the new child kernels into the system
- Throughput of CCQS is denoted by T : rate of processing the child kernel CTAs
- Total number of CTAs is denoted by n : Running + Pending
- If CTA arrival rate is higher than throughput, queuing latency will be high

## SPAWN Controller

- Estimating the benefit of launching child kernel and make a decision on launching or not
- Invoked at each kernel launch call
- launch overhead + queuing latency + execution time

> Launch overhead is modeled as the time to push child CTAs from SPAWN controller to CCQS.

> Queuing latency is modeled in CCQS as queuing time, and is calculated by examining throughput (T) and the number of jobs (n) residing in CCQS

T = (avg number of concurrent CTAs) / (avg child CTA execution time)

avg child CTA exe time is updated when a CTA finishes its execution and leaves CCQS

avg number of concurrent CTAs is computed in window of 1024 cycles. And at every cycle, add concurrent CTAs. At the end, get avg.

> Execution time on cores is modeled as the service time in CCQS, and is calculated using throughput (T) and the number of CTAs (x) that the new kernel has.

Launch new child kernel & finish assigned job
![[Untitled(180).png]]


---

If only parent kernel do everything
![[Untitled(181).png]]


t warp is calculated same as avg number of concurrent CTAs(window fashion)

Compare _**t parent**_ and _**t child.**_

## Evaluation
![[Untitled(182).png]]


- normalized to non DP
- offline-search: 여러가지 workload 비율 중에 제일 좋은 성능
![[Untitled(183).png]]


- Offline search and SPAWN do not make so many child kernels
    
- ⇒Reduce launch overhead
    
- In case of AMR(Adaptive Mesh Refinement) SPAWN can acheive high performance even though it does not create many child kernels ⇒ Prefers work doing in parent kernel
    
- in case of SA(sequence alignment) there are no big difference ⇒ Prefers work doing in child kernel
    
- In case of BFS-graph500, GC-graph500, MM(matrix mult)-small, SPAWN is slightly better than offline-search
    
- ⇒ This because SPAWN can dynamically adjust workload distribution
    
- SSSP(single source shortest path) SPAWN performs worse.
    
- ⇒SPAWN state can be updated when the child CTA is finished, but the decision is done before updating the state of the SPAWN
    
![[Untitled(184).png]]


- Baseline DP gives more jobs to child kernels than SPAWN
    
- ⇒Parent threads do not have much edges to traverse.
    
- ⇒Can not execute child kernel immediately due to launch overhead
    
- ⇒Because of HW limit, all of child kernels can not executed concurrently and queuing latency will be increased
    
- But in SPAWN, parent thread do more jobs than baseline DP
    
- ⇒Parent CTA execution can hide the child kernel launch overhead
    
- ⇒Reduce queuing latency and increase resource utilization
    

## Launching fewer child kernel leads to performance increase