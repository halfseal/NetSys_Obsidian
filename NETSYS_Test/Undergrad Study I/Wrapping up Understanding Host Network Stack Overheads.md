---
creator: 황준용
type: Paper Review
created: 2022-06-28
---
### Information to share:

1. Figure 1
    ![[Untitled(52).png]]
    
    
2. Table 2
    ![[Untitled(51).png]]
    
    
3. An important concept
    
    “this leads to increased per-byte data copy overhead and reduced throughput-per-core”
    
4. How DCA works
    
    Data is copied into cache regardless accesses from apps. (Cache eviction happens with very fast packet arrivals)
    
5. throughput-per-core
    
    the maximum throughput achievable by a single sender core
    
    - [**Understanding-network-stack-overheads-SIGCOMM-2021](https://github.com/Terabit-Ethernet/Understanding-network-stack-overheads-SIGCOMM-2021)/run_experiment_sender.py**
        
        ```c
        output.append("{:.3f}".format(total_throughput * 100 / receiver_results["cpu_util"]))
        ```
        
6. Small flow with large flow. Why perf degrads?
    
    A: GRO not available, mainly.
    

### Research topic (Will be added)

- The above three observations suggest interesting opportunities for orchestrating host resources between long and short flows: while executing on NIC-local NUMA nodes helps long flows significantly, short flows can be scheduled on NIC-remote NUMA nodes without any significant impact on performance; in addition, carefully scheduling the core across short flows sharing the core can lead to further improvements in throughput-per-core.
- Lock free implementation of the critical section above.