The counter in rps_dev_flow_table values records the length of the current CPU’s backlog when a packet in this flow was last enqueued. Each backlog queue has a head counter that is incremented on dequeue. A tail counter is computed as head counter + queue length. In other words, the counter in rps_dev_flow[i] records the last element in flow it that has been enqueued onto the currently designated CPU for flow i (of course, entry i is actually selected by hash and multiple flows may hash to the same entry i).
```c
 * The rps_dev_flow_table structure contains a table of flow mappings.
 */
struct rps_dev_flow_table {
	unsigned int		mask;
	struct rcu_head		rcu;
	struct rps_dev_flow	flows[];
};
```