[https://codebrowser.dev/linux/linux/net/core/dev.c.html](https://codebrowser.dev/linux/linux/net/core/dev.c.html)

```c
/*
 * enqueue_to_backlog is called to queue an skb to a per CPU backlog
 * queue (may be a remote CPU queue).
 */
static int enqueue_to_backlog(struct sk_buff *skb, int cpu,
			      unsigned int *qtail)
{
	enum skb_drop_reason reason;
	struct softnet_data *sd;
	unsigned long flags;
	unsigned int qlen;

	**reason** = SKB_DROP_REASON_NOT_SPECIFIED;
	sd = &per_cpu(softnet_data, cpu);

	rps_lock_irqsave(sd, &flags);
	if (!netif_running(skb->dev))
		goto drop;
	qlen = skb_queue_len(&sd->input_pkt_queue);
	if (qlen <= READ_ONCE(net_hotdata.max_backlog) &&
	    !skb_flow_limit(skb, qlen)) {
		if (qlen) {
enqueue:
			__skb_queue_tail(&sd->input_pkt_queue, skb);
			input_queue_tail_incr_save(sd, qtail);
			rps_unlock_irq_restore(sd, &flags);
			return NET_RX_SUCCESS;
		}

		/* Schedule NAPI for backlog device
		 * We can use non atomic operation since we own the queue lock
		 */
		if (!__test_and_set_bit(NAPI_STATE_SCHED, &sd->backlog.state))
			napi_schedule_rps(sd);
		goto enqueue;
	}
	reason = SKB_DROP_REASON_CPU_BACKLOG;

drop:
	sd->dropped++;
	rps_unlock_irq_restore(sd, &flags);

	dev_core_stats_rx_dropped_inc(skb->dev);
	kfree_skb_reason(skb, reason);
	return NET_RX_DROP;
}
```
