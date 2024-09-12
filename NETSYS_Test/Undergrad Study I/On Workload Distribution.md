---
creator: Maike
type: Paper Review
created: 2022-11-14
---
### Workload Distribution in “**Controlled Kernel Launch for Dynamic Parallelism in GPUs**”

1. Workload Distribution is non-trivial. There is no universally correct answer. It depends on the application and on the input of the application.

![[Untitled(222).png]]

### Other Resources

The section on related works regarding work distribution cites the following sources:

1. [Optimizing Off-chip Accesses in Multicores](https://dl.acm.org/doi/pdf/10.1145/2737924.2737989)
2. [Memory Row Reuse Distance and Its Role in Optimizing Application Performance](https://dl.acm.org/doi/pdf/10.1145/2745844.2745867)
3. [uC-States: Fine-grained GPU Datapath Power Management](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=7756752)
4. [Accelerating Irregular Algorithms on GPGPUs Using Fine-Grain Hardware Worklists](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=7011379)
5. [Scheduling Techniques for GPU Architectures with Processing-In-Memory Capabilities](https://dl.acm.org/doi/pdf/10.1145/2967938.2967940)
6. [Implementing Directed Acyclic Graphs with the Heterogeneous System Architecture](https://dl.acm.org/doi/pdf/10.1145/2884045.2884052)
7. [Improving Performance by Matching Imbalanced Workloads with Heterogeneous Platforms](https://dl.acm.org/doi/pdf/10.1145/2597652.2597675)
8. [Improving Bank-Level Parallelism for Irregular Applications](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=7783760)
9. [CUDA-NP: Realizing Nested Thread-level Parallelism in GPGPU Applications](https://dl.acm.org/doi/pdf/10.1145/2692916.2555254)

Scanning over the abstracts of some of those, most are not concerned with DP but with workload distribution in general.