---
Parameter:
  - sk_buff
Return: int
Location: /net/ipv4/tcp_ipv4.c
---
```c title=tcp_v4_rcv코드
int tcp_v4_rcv(struct sk_buff *skb)
{
	struct net *net = dev_net(skb->dev);
	enum skb_drop_reason drop_reason;
	int sdif = inet_sdif(skb);
	int dif = inet_iif(skb);
	const struct iphdr *iph;
	const struct tcphdr *th;
	bool refcounted;
	struct sock *sk;
	int ret;
	  
	drop_reason = SKB_DROP_REASON_NOT_SPECIFIED;
	if (skb->pkt_type != PACKET_HOST)
		goto discard_it;
		
	/* Count it even if it's bad */
	__TCP_INC_STATS(net, TCP_MIB_INSEGS);

	// skb의 크기가 tcp hdr보다 작으면 discard
	if (!pskb_may_pull(skb, sizeof(struct tcphdr)))
		goto discard_it;
		 
	th = (const struct tcphdr *)skb->data;

	// header 값들 확인
	if (unlikely(th->doff < sizeof(struct tcphdr) / 4)) {
		drop_reason = SKB_DROP_REASON_PKT_TOO_SMALL;
		goto bad_packet;
	}
	if (!pskb_may_pull(skb, th->doff * 4))
		goto discard_it;
		
	/* An explanation is required here, I think.
	* Packet length and doff are validated by header prediction,
	* provided case of th->doff==0 is eliminated.
	* So, we defer the checks. */
	  
	if (skb_checksum_init(skb, IPPROTO_TCP, inet_compute_pseudo))
		goto csum_error;
	  
	th = (const struct tcphdr *)skb->data;
	iph = ip_hdr(skb);
lookup:
	sk = __inet_lookup_skb(net->ipv4.tcp_death_row.hashinfo,
					skb, __tcp_hdrlen(th), th->source,
					th->dest, sdif, &refcounted);
	if (!sk)
		goto no_tcp_socket;
  
process:
	if (sk->sk_state == TCP_TIME_WAIT)
		goto do_time_wait;
	  
	if (sk->sk_state == TCP_NEW_SYN_RECV) {
		struct request_sock *req = inet_reqsk(sk);
		bool req_stolen = false;
		struct sock *nsk;
	  
		sk = req->rsk_listener;
		if (!xfrm4_policy_check(sk, XFRM_POLICY_IN, skb))
			drop_reason = SKB_DROP_REASON_XFRM_POLICY;
		else
			drop_reason = tcp_inbound_hash(sk, req, skb,
							&iph->saddr, &iph->daddr,
							AF_INET, dif, sdif);
		if (unlikely(drop_reason)) {
			sk_drops_add(sk, skb);
			reqsk_put(req);
			goto discard_it;
		}
		if (tcp_checksum_complete(skb)) {
			reqsk_put(req);
			goto csum_error;
		}
		if (unlikely(sk->sk_state != TCP_LISTEN)) {
			nsk = reuseport_migrate_sock(sk, req_to_sk(req), skb);
			if (!nsk) {
				inet_csk_reqsk_queue_drop_and_put(sk, req);
				goto lookup;
			}
			sk = nsk;
			/* reuseport_migrate_sock() has already held one sk_refcnt
			* before returning.
			*/
		} else {
			/* We own a reference on the listener, increase it again
			* as we might lose it too soon.
			*/
			sock_hold(sk);
		}
		refcounted = true;
		nsk = NULL;
		if (!tcp_filter(sk, skb)) {
			th = (const struct tcphdr *)skb->data;
			iph = ip_hdr(skb);
			tcp_v4_fill_cb(skb, iph, th);
			nsk = tcp_check_req(sk, skb, req, false, &req_stolen);
		} else {
			drop_reason = SKB_DROP_REASON_SOCKET_FILTER;
		}
		if (!nsk) {
			reqsk_put(req);
			if (req_stolen) {
				/* Another cpu got exclusive access to req
				* and created a full blown socket.
				* Try to feed this packet to this socket
				* instead of discarding it.
				*/
				tcp_v4_restore_cb(skb);
				sock_put(sk);
				goto lookup;
			}
			goto discard_and_relse;
		}
		nf_reset_ct(skb);
		if (nsk == sk) {
			reqsk_put(req);
			tcp_v4_restore_cb(skb);
		} else {
			drop_reason = tcp_child_process(sk, nsk, skb);
			if (drop_reason) {
				tcp_v4_send_reset(nsk, skb);
				goto discard_and_relse;
			}
			sock_put(sk);
			return 0;
		}
	}
	  
	if (static_branch_unlikely(&ip4_min_ttl)) {
		/* min_ttl can be changed concurrently from do_ip_setsockopt() */
		if (unlikely(iph->ttl < READ_ONCE(inet_sk(sk)->min_ttl))) {
			__NET_INC_STATS(net, LINUX_MIB_TCPMINTTLDROP);
			drop_reason = SKB_DROP_REASON_TCP_MINTTL;
			goto discard_and_relse;
		}
	}
	  
	if (!xfrm4_policy_check(sk, XFRM_POLICY_IN, skb)) {
		drop_reason = SKB_DROP_REASON_XFRM_POLICY;
		goto discard_and_relse;
	}
	  
	drop_reason = tcp_inbound_hash(sk, NULL, skb, &iph->saddr, &iph->daddr,
						AF_INET, dif, sdif);
	if (drop_reason)
		goto discard_and_relse;
	  
	nf_reset_ct(skb);
	  
	if (tcp_filter(sk, skb)) {
		drop_reason = SKB_DROP_REASON_SOCKET_FILTER;
		goto discard_and_relse;
	}
	th = (const struct tcphdr *)skb->data;
	iph = ip_hdr(skb);
	tcp_v4_fill_cb(skb, iph, th);
	  
	skb->dev = NULL;
	  
	if (sk->sk_state == TCP_LISTEN) {
		ret = tcp_v4_do_rcv(sk, skb);
		goto put_and_return;
	}
	  
	sk_incoming_cpu_update(sk);
	  
	bh_lock_sock_nested(sk);
	tcp_segs_in(tcp_sk(sk), skb);
	ret = 0;
	if (!sock_owned_by_user(sk)) {
		ret = tcp_v4_do_rcv(sk, skb);
	} else {
		if (tcp_add_backlog(sk, skb, &drop_reason))
			goto discard_and_relse;
	}
	bh_unlock_sock(sk);
	  
put_and_return:
	if (refcounted)
		sock_put(sk);
	  
	return ret;
  
no_tcp_socket:
	drop_reason = SKB_DROP_REASON_NO_SOCKET;
	if (!xfrm4_policy_check(NULL, XFRM_POLICY_IN, skb))
		goto discard_it;
	
	tcp_v4_fill_cb(skb, iph, th);
	  
	if (tcp_checksum_complete(skb)) {
csum_error:
	drop_reason = SKB_DROP_REASON_TCP_CSUM;
	trace_tcp_bad_csum(skb);
	__TCP_INC_STATS(net, TCP_MIB_CSUMERRORS);
bad_packet:
	__TCP_INC_STATS(net, TCP_MIB_INERRS);
	} else {
	tcp_v4_send_reset(NULL, skb);
	}
  
discard_it:
	SKB_DR_OR(drop_reason, NOT_SPECIFIED);
	/* Discard frame. */
	kfree_skb_reason(skb, drop_reason);
	return 0;
  
discard_and_relse:
	sk_drops_add(sk, skb);
	if (refcounted)
		sock_put(sk);
	goto discard_it;
  
do_time_wait:
	if (!xfrm4_policy_check(NULL, XFRM_POLICY_IN, skb)) {
		drop_reason = SKB_DROP_REASON_XFRM_POLICY;
		inet_twsk_put(inet_twsk(sk));
		goto discard_it;
	}
  
	tcp_v4_fill_cb(skb, iph, th);
  
	if (tcp_checksum_complete(skb)) {
		inet_twsk_put(inet_twsk(sk));
		goto csum_error;
	}
	switch (tcp_timewait_state_process(inet_twsk(sk), skb, th)) {
	case TCP_TW_SYN: {
		struct sock *sk2 = inet_lookup_listener(net,
						net->ipv4.tcp_death_row.hashinfo,
						skb, __tcp_hdrlen(th),
						iph->saddr, th->source,
						iph->daddr, th->dest,
						inet_iif(skb),
						sdif);
		if (sk2) {
			inet_twsk_deschedule_put(inet_twsk(sk));
			sk = sk2;
			tcp_v4_restore_cb(skb);
			refcounted = false;
			goto process;
		}
	}
		/* to ACK */
		fallthrough;
	case TCP_TW_ACK:
		tcp_v4_timewait_ack(sk, skb);
		break;
	case TCP_TW_RST:
		tcp_v4_send_reset(sk, skb);
		inet_twsk_deschedule_put(inet_twsk(sk));
		goto discard_it;
	case TCP_TW_SUCCESS:;
	}
	goto discard_it;
}
```

>우선 `skb->data`를 통해 tcp header를 가르키는 포인터를 가져오게 된다.
>그 후, `__inet_lookup_skb()`함수를 통해 소켓을 가져오게 되고, 만약 해당하는 TCP 소켓이 없다면 `no_tcp_socket`라벨로 건너뛰게 된다.
>
>TCP 소켓을 확인하였다면 `process` 라벨 아래 코드들이 실행되는데, `sk_state`값을 확인하여 작업을 이어나간다.
>
 `sk_state == TCP_NEW_SYN_RECV` 조건문으로 3-way handshake 작업이 수행되게 되는데, `request_sock` 타입의 구조체로 `sock`을 캐스팅하여 `req` 변수로 다루게 된다. `xfrm4_policy_check()` 함수를 통해 패킷의 드랍 여부를 판단하고, 만약 드랍 이유가 발생하였다면 드랍하게 된다.
>
>그후 마저 체크썸을 확인하고 만약 소켓의 상태가 `TCP_LISTEN`이 아니라면 포트를 그대로 사용하면서 소켓을 migrate하여 새로운 소켓으로 대체한다. `TCP_LISTEN`이라면 `sock_hold()`함수를 사용하게 된다.
>
>그후 `tcp_filter()`함수를 호출하여 드랍여부를 조사하게 된다. 이것을 바탕으로 컨트롤 블럭에 메타데이터를 세팅해주고, `tcp_check_req()`함수를 호출하여 새로운 flow를 만들게 되는 것이다.
>여기서는 `sock_put()`함수를 통해 작업이 수행되게 된다. 그후 0을 return하게 된다.


>여기서부터는 새로운 flow가 아닌 부분들이다. 만약 `SYN`패킷이라면 위에서 처리가 끝났다.
>
>이후 정책값 확인, ttl 확인 및 `tcp_filter()`등을 통해 패킷을 검사하고, `tcp_v4_fill_cb()`함수를 통해 메타데이터를 채운다음, 만약 소켓의 상태가 `TCP_LISTEN`이라면 `tcp_v4_do_rcv()`함수를 호출하고, `ret`에 결과를 저장하여 `put_and_return`라벨로 건너뛰게 된다.

> 그게 아니라면 `sk_incoming_cpu_update()`를 실행한후 한번 더 조건을 확인하고 `tcp_v4_do_rcv()`함수를 실행하게 된다.

>`put_and_return`라벨은 보통의 정상적인 수신이 되었을 때 나타나는 루틴이며, 여기서도 `sock_out()`함수를 실행하고 `ret`을 리턴한다.

---
packet의 type이 PACKET_HOST인 경우에만 진행한다.
pskb_may_pull() 함수가 tcp header option 이 sk_buff의 kmalloc된 부분에 존재하도록 보장한다.
\_\_inet_lookup_skb()함수로 packet이 가야할 socket을 찾아준다.
위 함수에 찾아진 socket state 따라서 처리 방식이 달라진다.
1. socket이 TCP_TIME_WAIT인 경우
	 tcp connection 이 close 된 이후에도 delay packet들을 받기 위해서 잠시 유지하는 상태이다.
2. TCP_NEW_SYN_RECV인 경우
	 ![[Pasted image 20240901180451.png]]
	 xfrm policy check, checksum validation한다. 새로운 socket을 생성할 수도 있다.
3. TCP_LISTEN인 경우
	 tcp_v4_do_rcv() 함수 실행한다. 

sock_owned_by_user() 함수는 현재 소켓이 어플리케이션에서 사용하고 있는 상황이면 true를 리턴
소켓이 사용되지 않는 경우에는 tcp_v4_do_rcv()를 실행하고
소켓이 사용되는 경우에는 tcp_add_backlog()를 실행한다.

[[Encyclopedia of NetworkSystem/Function/net-ipv4/`__inet_lookup_skb()]]
[[tcp_filter()]]
[[tcp_check_req()]]
[[Encyclopedia of NetworkSystem/Function/net-ipv4/tcp_v4_do_rcv()|tcp_v4_do_rcv()]]
[[sock_owned_by_user()]]
[[tcp_add_backlog()]]

As usual, the pskb_may_pull() function is responsible for ensuring that the TCP header, and in the next call, the header options are in the kmalloc'ed portion of the sk_buff.
