1. state가 TCP_ESTABLISHED 경우, tcp_rcv_established()를 호출하여 처리한다.
2. state가 TCP_LISTEN 경우, cookie_check 이 후 tcp_child_process()를 호출하여 처리한다.
3. 그 외에는 tcp_rcv_state_process()를 호출하여 처리한다.

tcp_v4_do_rcv 함수부터 실제 프로토콜을 수행한다. ESTABLISHED 상태는 tcp_rcv_esablished를 호출한다. ESTABLISHED 상태가 가장 흔하기 때문에 이 상태 처리를 별도로 떼어 내서 최적화한다. tcp_rcv_established는 header prediction 코드를 먼저 수행한다. Header prediction도 흔한 경우를 감지해서 빨리 처리한다. 여기서 흔한 경우는 보낼 데이터는 없고, 받은 데이터 패킷이 다음에 받아야 하는 패킷인 경우, 즉 시퀀스 번호가 수신 TCP가 기대하는 시퀀스 번호인 경우다. 데이터를 소켓 버퍼에 추가하고 ACK를 전송하면 끝이다.

 패킷 역전(out-of-order delivery) 현상
```c
/* The socket must have it's spinlock held when we get
 * here, unless it is a TCP_LISTEN socket.
 *
 * We have a potential double-lock case here, so even when
 * doing backlog processing we use the BH locking scheme.
 * This is because we cannot sleep with the original spinlock
 * held.
 */
int tcp_v4_do_rcv(struct sock *sk, struct sk_buff *skb)
{
	enum skb_drop_reason reason;
	struct sock *rsk;

	if (sk->sk_state == TCP_ESTABLISHED) { /* Fast path */
		struct dst_entry *dst;

		dst = rcu_dereference_protected(sk->sk_rx_dst,
						lockdep_sock_is_held(sk));

		sock_rps_save_rxhash(sk, skb);
		sk_mark_napi_id(sk, skb);
		if (dst) {
			if (sk->sk_rx_dst_ifindex != skb->skb_iif ||
			    !INDIRECT_CALL_1(dst->ops->check, ipv4_dst_check,
					     dst, 0)) {
				RCU_INIT_POINTER(sk->sk_rx_dst, NULL);
				dst_release(dst);
			}
		}
		tcp_rcv_established(sk, skb);
		return 0;
	}

	if (tcp_checksum_complete(skb))
		goto csum_err;

	if (sk->sk_state == TCP_LISTEN) {
		struct sock *nsk = tcp_v4_cookie_check(sk, skb);

		if (!nsk)
			return 0;
		if (nsk != sk) {
			reason = tcp_child_process(sk, nsk, skb);
			if (reason) {
				rsk = nsk;
				goto reset;
			}
			return 0;
		}
	} else
		sock_rps_save_rxhash(sk, skb);

	reason = tcp_rcv_state_process(sk, skb);
	if (reason) {
		rsk = sk;
		goto reset;
	}
	return 0;

reset:
	tcp_v4_send_reset(rsk, skb);
discard:
	kfree_skb_reason(skb, reason);
	/* Be careful here. If this function gets more complicated and
	 * gcc suffers from register pressure on the x86, sk (in %ebx)
	 * might be destroyed here. This current version compiles correctly,
	 * but you have been warned.
	 */
	return 0;

csum_err:
	reason = SKB_DROP_REASON_TCP_CSUM;
	trace_tcp_bad_csum(skb);
	TCP_INC_STATS(sock_net(sk), TCP_MIB_CSUMERRORS);
	TCP_INC_STATS(sock_net(sk), TCP_MIB_INERRS);
	goto discard;
}
```

[[황재훈/Research Intern/082924/tcp_rcv_established()|tcp_rcv_established()]]
[[황재훈/Research Intern/082924/tcp_child_process()|tcp_child_process()]]
[[황재훈/Research Intern/082924/tcp_rcv_state_process()]]
