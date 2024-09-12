---
creator: 황준용
type: Paper Review
created: 2022-11-21
---
### A Compiler Framework for Optimizing Dynamic Parallelism on GPUs

[https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=9741284](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=9741284)

propose a compiler framework for optimizing the use of dynamic parallelism in applications with nested parallelism. The framework features three key optimizations: **thresholding, coarsening, and aggregation.**

| Software     | Hardware  |
| ------------ | --------- |
| Thread       | CUDA core |
| Thread Block | SM        |
| Grid         | Device    |
![[Untitled(234).png]]


- Grid > Block > Thread
- gridDim : 한 그리드 내 블록의 수.
- blockIdx : 몇 번째 블록인지 나타내는 인덱스.
- blockDim : 한 블록 내 스레드의 수.
- threadIdx : 몇 번째 스레드인지 나타내는 인덱스.
- Thresholding involves launching a grid dynamically only if the number of child threads exceeds some threshold

Grid = set of threadsblock launched in one kernel
![[Untitled(235).png]]

![[Untitled(236).png]]


- Coarsening involves executing the work of multiple thread blocks by a single coarsened block to amortize the common work across them.
![[Untitled(237).png]]

![[Untitled(238).png]]


- Aggregation involves combining multiple child grids into a single aggregated grid.
![[Untitled(239).png]]

![[Untitled(240).png]]

![[Untitled(241).png]]


++ Dataset
![[Untitled(242).png]]
