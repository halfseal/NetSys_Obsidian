gro_normal_one 에서 실행된 napi_skb_finish(), napi_gro_complete()에서 실행
네트워크에서 가져온 rx buffer를 처리한다
buffer가 congestion control 이나 다른 layer에 의해 드랍될 수 있다.
softirq context 에서만 콜 될 수 있다. interrupt이 허용된다.

trace_netif_receive_skb_entry, exit은 디버깅을 위한 코드이다?
NET_RX_SUCCESS : no congestion
NET_RX_DROP : packet dropped

[[netif_receive_skb_internal()]]

```c
int netif_receive_skb(struct sk_buff *skb)
{
	int ret;

	trace_netif_receive_skb_entry(skb);

	ret = netif_receive_skb_internal(skb);
	trace_netif_receive_skb_exit(ret);

	return ret;
}
EXPORT_SYMBOL(netif_receive_skb);
```


```c
void netif_receive_skb_list(struct list_head *head)
{
	struct sk_buff *skb;

	if (list_empty(head))
		return;
	if (trace_netif_receive_skb_list_entry_enabled()) {
		list_for_each_entry(skb, head, list)
			trace_netif_receive_skb_list_entry(skb);
	}
	netif_receive_skb_list_internal(head);
	trace_netif_receive_skb_list_exit(0);
}
```
