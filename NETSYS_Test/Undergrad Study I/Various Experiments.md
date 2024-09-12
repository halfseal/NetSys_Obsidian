---
creator: 황준용
type: Hands-On
created: 2022-08-23
---
## eRPC

- Just Read

## Packet Filtering using GPU

### Papers

1. Acceleration of Packet Filtering using GPGPU
    
    - Paper Review
        
        ### Problem
        
        1. Packet filtering is core funcitonality in computer networks.
        2. With the introduction of new types of services and applications, there is a growing demand for larger bandwidth and also for improved security.
        3. However, "larger bandwith" and "improved security" conflict, since providing security partly relies on screening packet traffic implying an overhead.
        4. Thus, to not be susceptible to Dos attacks, high processing power is needed.
        
        > Therefore, this paper presents and analyse various parallel implementations of packet filtering running on cost effective GPGPU. It describes an approach to efficiently exploit the massively parallel capabilities of the GPGPU.
        
        ### Background
        
        - Packet filtering involves finding a match based on packet's metadata from the header for all ingress and egress packets.
            
        - The packet filtering decides to either DROP or ACCEPT.
            
        - Attackers can attack this firewall by:1. Find the "Last matching ruleset", remotedly.2. Since the "Last matching ruleset is located at the bottommost rule, it makes the server's firewall consume the most CPU power.3. This attack can be used in Dos attack, bringing firewall to its knee.
            
        - This kind of an attack can be minimized by increading the computing power.
            
        - Many optimization techniques are employed to increase throughput or decrease packet latency of firewalls:
            
            1. Algorithmic optimization
                
            2. Implementing packet filtering in FPGA/ASIC
                
            
            3. **Parallel Firewall**
            
        - Parallel Firewalls can be implemented by _dividing either the traffic or the workload across an array of firewall nodes._
            
        
        ### CUDA Programming Model && Linux Netfilter
        
        ### CUDA
        
        - GPU, with the parallel structure, has high computational power and memory bandwith, showing more effective than general purpose CPU for many applications.
        - CUDA is a parallel programming model that leverages the computational capacity of the GPUs for non-graphics applications.
        - Programming in CUDA entails the distribution of the code for execution between the host CPU and the device GPU.
        
        ### Linux NetFilter
        
        - Under Linux, packet filtering is built into the kernel as part its networking stack. Netfilter is a framework for packet mangling, outside the normal socket interface.
        - Each protocol defines hooks, well-defined points in a packet's traversal of that protocol stack, and kernel modules can register to listen to the different hooks for each protocol and get to examine the packet in order to decide the fate of the packet - discard, allow or ask netfilter to queue the packet for userspace.
        - These rules can be determined using the [iptables tool](https://velog.io/@brian11hwang/Firewall-with-IPTables)
        
        ### Implementation
        
        This paper uses CUDA to implement packet filtering that will execute in parallel on GPGPU. Yet, nVidia does not provide any drivers which allow the direct control of their cards from within the Linux kernel,which forces all interactions to originate from userspace.
        
        Thus, the model:
        
        1. queues packets to userspace
            
        2. processes them in the GPGPU
            
        3. reinject back to the kernel networking stack with action decided from filtering.
            
        
        The test-be created uses a set of rules, each containing 5-tuple:1. source IP address2. source port number3. destination IP address4. destination port number5. protocol, and specifies the action (Accept / Deny) for all packets across the network.
        
        ![https://velog.velcdn.com/images/brian11hwang/post/fd6349e5-7a90-42f9-8537-da371f547871/image.png](https://velog.velcdn.com/images/brian11hwang/post/fd6349e5-7a90-42f9-8537-da371f547871/image.png)
        
        ### Data-parallel Firewall
        
        - Data parallel is a design that distributes the data (packets) across the firewall nodes.
        - Data parallel firewall consists of multiple firewall node, where each ith node has the whole rulset, working independantly.
        - Arriving packets are distributes across the nodes such that only 1 firewall processes the given packet.
        - The set of packets are accumulated, copied to device global memory, processed in parallel by different threads on GPGPU, and result is written to verdict array.
        
        ![https://velog.velcdn.com/images/brian11hwang/post/82802b0e-6171-46ad-a85f-0c0241fcd45b/image.png](https://velog.velcdn.com/images/brian11hwang/post/82802b0e-6171-46ad-a85f-0c0241fcd45b/image.png)
        
        ### Function-Parallel Firewall
        
        - A function parallel Firewall design consistsof an array of firewall nodes.
        - In this design, arriving traffic is duplicated to all firewall nodes. Each firewall node 'i' employs a part of the security policy. After a packet is processed by each firewall node 'i', the result of each Ri is sent to the calling program running on the CPU.
        - To ensure no more than one firewall node determines action on a packet only the sequential code can execute an action on a packet.
        - Verdict array is copied from device to host after all the threads complete execution.
        
        ![https://velog.velcdn.com/images/brian11hwang/post/1ed9243d-555e-4a91-94b5-6752d7cbbf8d/image.png](https://velog.velcdn.com/images/brian11hwang/post/1ed9243d-555e-4a91-94b5-6752d7cbbf8d/image.png)
        
        ### Hybrid Model
        
        - In Hybrid Model, a set of packets are accumulated and each packet is processed in parallel by a different block on GPGPU.
        - Within a block, a packet is processed by multiple threads simultaneously with each thread checking a part of the ruleset. Rule database is kept in device global memory.
        - Verdict array is copied from device to host after all the threads complete execution.
        - Using massively parallel computing capabilities of GPGPU we have achieved a better model with increased throughput and reduced packet processing delay compared to data/function-parallel firewall.
        
        ![https://velog.velcdn.com/images/brian11hwang/post/ea214763-737b-4d6f-9b19-e1091a731f80/image.png](https://velog.velcdn.com/images/brian11hwang/post/ea214763-737b-4d6f-9b19-e1091a731f80/image.png)
        
        ### Results
        
        - Test was ran on models on a node that has 3 GHz Dual Core processor with 1 GB RAM with nVidia Tesla C870 GPGPU with 4 GB global memory and 128 cores.
        - The applications were first executed on CPU. These were compared with the timings from the parallel models executed on CPU + GPGPU.
        - Traffic used for performance analysis contains packets whose source or destination addresses are from different class of IP address. The modeled traffic also contains traffic from different services like FTP, SCP and TELNET.
        
        ### Data-parallel Firewall
        
        - LIMITATION: Accesses to memory are cached. But caching does not improve performance since the cached data is not used later. Overhead of copying the whole ruleset to shared memory eclipses the improvement achieved by reduction in global memory contention.
        
        ### Function-Parallel Firewall
        
        - LIMITATION: Because of the CUDA overhead in copying verdict array to/from the device memory and kernel launch for every packet makes function-parallel model inefficient.
        
        ### Hybrid Model
        
        - In hybrid model, a packet is assigned to a CUDA block processed by the threads within that block. Number of threads launched per block has significant effect on the processing time.
        
        ### Simulation
        
        Simulations are performed to test the latency of various models of Parallel Firewalls. Each is compared with the conventional sequential approach.
        
        ![https://velog.velcdn.com/images/brian11hwang/post/7e686373-ddbe-4e4d-80e4-4463e289de6d/image.png](https://velog.velcdn.com/images/brian11hwang/post/7e686373-ddbe-4e4d-80e4-4463e289de6d/image.png)
        
        ![https://velog.velcdn.com/images/brian11hwang/post/2511b10e-1d75-4996-8d39-387d24186ea6/image.png](https://velog.velcdn.com/images/brian11hwang/post/2511b10e-1d75-4996-8d39-387d24186ea6/image.png)
        
        ### Conclusion
        
        Data-Parallel design gives 3.4x and Hybrid model gives4.25x speedup compared to sequential approach. Because ofincreased throughput and low latency, parallel firewalls running on multi-cores will be more resilient to attack based on remote discovery of last matching rules.Yet, current parallel models are stateless. _**A stateful parallel firewall will pose new challenges to implement.**_
        
2. Graphics processing unit based next generation DDoS prevention system
    
    - Paper Review
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/2c77db72-d4ad-4676-8ec4-ac0e50cdaf36/Untitled.png)
        

### Experiments

1. Simple IPTables experiment I: ICMP BLock
    
    - Experiment I
        
        Ping Disable
        
        ICMP ping before
        
        ![https://velog.velcdn.com/images/brian11hwang/post/da930ac5-a220-4281-82f0-3138009e6e1f/image.png](https://velog.velcdn.com/images/brian11hwang/post/da930ac5-a220-4281-82f0-3138009e6e1f/image.png)
        
        ```
        IPTABLES -N ICMP
        IPTABLES -A INPUT -p icmp -j ICMP
        IPTALBLES -A ICMP -p icmp -icmp-type 8 -j DROP
        ```
        
        Add a policy to iptables that does not respond to ICMP messages.
        
        - IPTABLES -N ICMP: Adds a new chain called ICMP to the table.
        - IPTABLES-A INPUT-picmp-j ICMP: Adds a policy that forwards ICMP messages to the ICMP chain when they are received.
        - IPTALBLES-A ICMP-picmp-icmp-type 8-j DROP: Type 8 of ICMP messages (icmp echo request)Block messages.
        
        When ICMP File comes(INPUT), redirected to 'ICMP'
        
        ![https://velog.velcdn.com/images/brian11hwang/post/2e9150c5-1ab8-4d62-b86b-7453f8b24740/image.png](https://velog.velcdn.com/images/brian11hwang/post/2e9150c5-1ab8-4d62-b86b-7453f8b24740/image.png)
        
        With ICMP type 8 (icmp echo request), DROP Package
        
        ![https://velog.velcdn.com/images/brian11hwang/post/09e66a1b-9a40-4bed-ae9f-e1d3bc25e816/image.png](https://velog.velcdn.com/images/brian11hwang/post/09e66a1b-9a40-4bed-ae9f-e1d3bc25e816/image.png)
        
        We can see that no response to Ping happened.
        
2. Simple IPTables experiment II : UDP Flood Block
    
    - Experiment II
        
        ### Before:
        
        1. Iperf Log
        
        ```
        brian11hwang@jooyoung iperf -c 10.0.1.3 -B 10.0.1.4 -u -b 5G
        ------------------------------------------------------------
        Client connecting to 10.0.1.3, UDP port 5001
        Binding to local address 10.0.1.4
        Sending 1470 byte datagrams, IPG target: 2.19 us (kalman adjust)
        UDP buffer size:  208 KByte (default)
        ------------------------------------------------------------
        [  3] local 10.0.1.4 port 52229 connected with 10.0.1.3 port 80
        [ ID] Interval       Transfer     Bandwidth
        [  3]  0.0-10.0 sec  2.86 GBytes  2.46 Gbits/sec
        [  3] Sent 2087706 datagrams
        [  3] Server Report:
        [  3]  0.0-10.0 sec  2.80 GBytes  2.41 Gbits/sec   0.006 ms 41410/2087706 (2%)
        ```
        
        1. Tshark Log
        
        ```
        5584414 178.703669944     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
        5584415 178.703702400     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
        5584416 178.703702416     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
        5584417 178.703702431     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
        5584418 178.703702445     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
        5584419 178.703702459     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
        5584420 178.703702473     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
        5584421 178.703702488     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
        5584422 178.703702502     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
        5584423 178.703736543     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
        5584424 178.703736557     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
        5584425 178.703736572     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
        5584426 178.703736587     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
        5584427 178.721185099     10.0.1.3 → 10.0.1.4     UDP 1512 5001 → 58225 Len=1470
        ```
        
        Iperf Server Periodically sends ACK Although UDP Connection for status check.
        
        ### Iptables Setting:
        
        ```
        IPTABLES -N UDP
        IPTABLES -A INPUT -p udp -j UDP
        sudo iptables -I UDP -p udp -d 10.0.1.3 --dport 5001 -j DROP
        IPTABLES -A UDP -j LOG --log-prefix "UDP FLOOD"
        ```
        
        (Iperf uses 5001 Port)
        
        ### After:
        
        ```
        brian11hwang@jooyoung iperf -c 10.0.1.3 -B 10.0.1.4 -u -b 5G
        ------------------------------------------------------------
        Client connecting to 10.0.1.3, UDP port 5001
        Binding to local address 10.0.1.4
        Sending 1470 byte datagrams, IPG target: 2.19 us (kalman adjust)
        UDP buffer size:  208 KByte (default)
        ------------------------------------------------------------
        [  3] local 10.0.1.4 port 56652 connected with 10.0.1.3 port 5001
        [  3] WARNING: did not receive ack of last datagram after 10 tries.
        [ ID] Interval       Transfer     Bandwidth
        [  3]  0.0-10.0 sec  6.25 GBytes  5.37 Gbits/sec
        [  3] Sent 4565229 datagrams
        ```
        
        However, after Cannot Recieve Ack because Iptables drop all packet. Thus Iperf does not know the need to send ACK.
        

## ETC

### DPDK vs XDP (using Iperf)

Server Side:

```bash
brian11hwang@jooyoung iperf -s
------------------------------------------------------------
Server listening on TCP port 5001
TCP window size:  128 KByte (default)
------------------------------------------------------------
[  4] local 10.0.1.4 port 5001 connected with 10.0.1.3 port 54558
[ ID] Interval       Transfer     Bandwidth
[  4]  0.0-10.0 sec  11.0 GBytes  9.40 Gbits/sec
```

Client Side:

```bash
brian11hwang@b1 sudo iperf -c 10.0.1.4
------------------------------------------------------------
Client connecting to 10.0.1.4, TCP port 5001
TCP window size: 2.52 MByte (default)
------------------------------------------------------------
[  3] local 10.0.1.3 port 54558 connected with 10.0.1.4 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-10.0 sec  11.0 GBytes  9.42 Gbits/sec
```

_using UDP_

It is important for the Server side to also listen in UDP Mode!

Server Side:

```bash
brian11hwang@b1 iperf -s -u
------------------------------------------------------------
Server listening on UDP port 5001
Receiving 1470 byte datagrams
UDP buffer size:  208 KByte (default)
------------------------------------------------------------
[  3] local 10.0.1.3 port 5001 connected with 10.0.1.4 port 50905
[ ID] Interval       Transfer     Bandwidth        Jitter   Lost/Total Datagrams
[  3]  0.0-10.0 sec  1.25 MBytes  1.05 Mbits/sec   0.006 ms    0/  892 (0%)
[  4] local 10.0.1.3 port 5001 connected with 10.0.1.4 port 57524
[  4]  0.0-10.0 sec  1.25 MBytes  1.05 Mbits/sec   0.003 ms    0/  892 (0%)
```

Client Side: