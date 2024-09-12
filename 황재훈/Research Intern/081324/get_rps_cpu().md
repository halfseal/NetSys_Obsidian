---
Location: /net/core/dev.c
---
__netif_receive_skb__ 에서 호출된 함수로 특정 skb가 어느 cpu로 가야하는지 알려준다.

rps_sock_flow_table에는 desired cpu, 마지막으로 해당 flow를 처리한 cpu가 있다.
rps_dev_flow_table에는 current cpu, 지금 해당 flow를 처리하고 있는 cpu가 있다.
대부분의 경우 desired cpu, current cpu는 동일하다. 하지만 다른 경우 따로 처리를 한다.
만약 flow table에서 cpu가 조회되지 않는다면 rps map을 참조해서 cpu를 가져온다.
[[rps_sock_flow_table]]
[[rps_dev_flow_table]]
[[netdev_rx_queue]]
[[rps_map]]

packet들을 순서대로 처리하기 위해서 rps_sock_flow table과 rps_dev_flow table을 비교한다. 만약 desired cpu가 (rps_sock_flow에서 찾아진) 지금 current cpu(rps_dev_flow)와 일치한다면 packet은 cpu의 back log 에 enqueue 된다. 만약 두 cpu가 일치하지 않는다면 다음의 조건을 만족할 경우, current cpu를 desired cpu로 바꾼다. 
1. current cpu가 설정이 되어 있지 않은 경우,  tcpu >= nr_cpu_ids 
2. current cpu가 offline 인 경우,  !cpu_online(tcpu)
3. current cpu queue의 head counter >= rps_dev_flow[]의 tail counter보다 작은 경우

이후 packet은 current cpu로 전송이 된다. 3번 조건을 통해서 현재 cpu 내의 queue가 모두 처리되기 전에는 다음 cpu로 옮기지 않는다. 


```c
static int get_rps_cpu(struct net_device *dev, struct sk_buff *skb,
		       struct rps_dev_flow **rflowp)
{
	const struct rps_sock_flow_table *sock_flow_table;
	struct netdev_rx_queue *rxqueue = dev->_rx;
	struct rps_dev_flow_table *flow_table;
	struct rps_map *map;
	int cpu = -1;
	u32 tcpu;
	u32 hash;

	if (skb_rx_queue_recorded(skb)) {
		u16 index = skb_get_rx_queue(skb);

		if (unlikely(index >= dev->real_num_rx_queues)) {
			WARN_ONCE(dev->real_num_rx_queues > 1,
				  "%s received packet on queue %u, but number "
				  "of RX queues is %u\n",
				  dev->name, index, dev->real_num_rx_queues);
			goto done;
		}
		rxqueue += index;
	}

	/* Avoid computing hash if RFS/RPS is not active for this rxqueue */

	flow_table = rcu_dereference(rxqueue->rps_flow_table);
	map = rcu_dereference(rxqueue->rps_map);
	if (!flow_table && !map)
		goto done;

	skb_reset_network_header(skb);
	hash = skb_get_hash(skb);
	if (!hash)
		goto done;

	sock_flow_table = rcu_dereference(net_hotdata.rps_sock_flow_table);
	if (flow_table && sock_flow_table) {
		struct rps_dev_flow *rflow;
		u32 next_cpu;
		u32 ident;

		/* First check into global flow table if there is a match.
		 * This READ_ONCE() pairs with WRITE_ONCE() from rps_record_sock_flow().
		 */
		ident = READ_ONCE(sock_flow_table->ents[hash & sock_flow_table->mask]);
		if ((ident ^ hash) & ~net_hotdata.rps_cpu_mask)
			goto try_rps;

		next_cpu = ident & net_hotdata.rps_cpu_mask;

		/* OK, now we know there is a match,
		 * we can look at the local (per receive queue) flow table
		 */
		rflow = &flow_table->flows[hash & flow_table->mask];
		tcpu = rflow->cpu;

		/*
		 * If the desired CPU (where last recvmsg was done) is
		 * different from current CPU (one in the rx-queue flow
		 * table entry), switch if one of the following holds:
		 *   - Current CPU is unset (>= nr_cpu_ids).
		 *   - Current CPU is offline.
		 *   - The current CPU's queue tail has advanced beyond the
		 *     last packet that was enqueued using this table entry.
		 *     This guarantees that all previous packets for the flow
		 *     have been dequeued, thus preserving in order delivery.
		 */
		if (unlikely(tcpu != next_cpu) &&
		    (tcpu >= nr_cpu_ids || !cpu_online(tcpu) ||
		     ((int)(per_cpu(softnet_data, tcpu).input_queue_head -
		      rflow->last_qtail)) >= 0)) {
			tcpu = next_cpu;
			rflow = set_rps_cpu(dev, skb, rflow, next_cpu);
		}

		if (tcpu < nr_cpu_ids && cpu_online(tcpu)) {
			*rflowp = rflow;
			cpu = tcpu;
			goto done;
		}
	}

try_rps:

	if (map) {
		tcpu = map->cpus[reciprocal_scale(hash, map->len)];
		if (cpu_online(tcpu)) {
			cpu = tcpu;
			goto done;
		}
	}

done:
	return cpu;
}
```

