---
creator: Maike
type: Paper Review
created: 2022-05-10
---
Reading material: Iron: Isolating Network-based CPU in Container Environments

The paper’s main focus/problem:

In containerized environments, the nature of network processing collides with the principle of isolation. Why? Because arriving packets are handled the moment they arrive using interrupts, which is not necessary during the execution period of the receiving container. Hence, another container is doing the work of somebody else. Can’t we just ignore packets until it's the container’s time to process them? Not really, since unnecessary buffering of packets can lead to lower performance and eventually packet loss.

How is this relevant to our current paper?

The current paper, too, runs multiple applications on one core. They are not containerized and are not trying to guarantee isolation and fairness so that their networking has an impact on each other goes without saying.

The real question is: Why are we seeing increased scheduling overhead and decreased performance when short and long flows are mixed?

1. Traffic Patterns Long flows consistently send packets to the host as an ongoing stream of data Short Flows work in a send/response kind of way and arrive sporadically →Short Flows might trigger a similar amount of interrupts for only a fraction of the data when compared to long flows. This explains why short flow processing alone has such high scheduling overhead, but why mixed? Is it really just because the short-flow applications go idle so fast?
2. Simple Head-Of-Line blocking (Not scheduling related) Short-flow performance drops because there is always other data from the long flow being processed. In a pure short-flow environment, there will always be packets equal to the number of short-flows running on the host at most, due to its request-response nature.
3. Flood of Control Packets (Not scheduling related) Short connections need to be established and torn down, which are all actions navigated by control packets. Data packets for established connections have a ‘fast path’. So maybe it’s the long flow packets being blocked by the control packets which decreases the long-flow performance. (Would be interesting to see if these observations would replicate using mTCP, where control packets and data packets have different processing pipelines)
![[Untitled(29).png]]    
