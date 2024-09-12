---
creator: 황준용
type: Paper Review
created: 2022-11-14
---
### 1. WIREFRAME: Supporting Data-dependent Parallelism through Dependency Graph Execution in GPUs

[https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=8686594](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=8686594)

### 2. Efficient Auto-Tuning of Parallel Programs with Interdependent Tuning Parameters via Auto-Tuning Framework (ATF) A Compiler Framework for Optimizing Dynamic Parallelism on GPUs

[https://dl.acm.org/doi/pdf/10.1145/3427093](https://dl.acm.org/doi/pdf/10.1145/3427093)

### 3. A Compiler Framework for Optimizing Dynamic Parallelism on GPUs

[https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=9741284](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=9741284)

propose a compiler framework for optimizing the use of dynamic parallelism in applications with nested parallelism. The framework features three key optimizations: **thresholding, coarsening, and aggregation.**

- Thresholding involves launching a grid dynamically only if the number of child threads exceeds some threshold
![[Untitled(217).png]]


- Coarsening involves executing the work of multiple thread blocks by a single coarsened block to amortize the common work across them.
![[Untitled(218).png]]


- Aggregation involves combining multiple child grids into a single aggregated grid.
![[Untitled(219) 1.png]]
![[Untitled(220) 1.png]]
![[Untitled(221).png]]


### 4. Utilizing dynamic parallelism in CUDA to accelerate a 3D red-black successive over relaxation wind-field solver

[https://www.sciencedirect.com/science/article/pii/S1364815221000013](https://www.sciencedirect.com/science/article/pii/S1364815221000013)

### 5. Characterization and Analysis of Dynamic Parallelism in Unstructured GPU Applications

[https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=6983039](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=6983039)