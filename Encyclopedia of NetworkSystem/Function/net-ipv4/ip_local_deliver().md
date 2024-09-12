---
Parameter:
  - sk_buff
Return: int
Location: /net/ipv4/ip_input.c
---
```c title=ip_local_deliver코드
int ip_local_deliver(struct sk_buff *skb)
{
	/*
	* Reassemble IP fragments.
	*/
	struct net *net = dev_net(skb->dev);
	  
	if (ip_is_fragment(ip_hdr(skb))) {
		if (ip_defrag(net, skb, IP_DEFRAG_LOCAL_DELIVER))
			return 0;
	}
	  
	return NF_HOOK(NFPROTO_IPV4, NF_INET_LOCAL_IN,
				net, NULL, skb, skb->dev, NULL,
				ip_local_deliver_finish);
}
```

>L4에 패킷을 전달하는 함수이다. 만약 fragment 되어있는 패킷이라면 이를 재조립하는 `ip_defrag()`호출하고, 0을 반환한다.
>아니라면, `ip_local_deliver_finish()`함수를 호출하게 된다.

[[ip_defrag()]]
[[ip_local_deliver_finish()]]
