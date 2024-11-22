``` c
void tcp_init_xmit_timers(struct sock *sk)
{
	inet_csk_init_xmit_timers(sk, &tcp_write_timer, &tcp_delack_timer,
				  &tcp_keepalive_timer);
	...
```
이렇게 각 소켓마다 타이머 초기화
``` c
// net/ipv4/tcp_timer.c
static void tcp_write_timer(struct timer_list *t)
{
	struct inet_connection_sock *icsk =
			from_timer(icsk, t, icsk_retransmit_timer);
	struct sock *sk = &icsk->icsk_inet.sk;
	...
}
```
따라서 타이머에서 소켓 정보 쓰기 위해 이렇게 역으로 불러와야

``` c
// net/input/tcp_timer.c
/**
 *  tcp_retransmit_timer() - The TCP retransmit timeout handler
 *  @sk:  Pointer to the current socket.
 *
 *  This function gets called when the kernel timer for a TCP packet
 *  of this socket expires.
 *
 *  It handles retransmission, timer adjustment and other necesarry measures.
 *
 *  Returns: Nothing (void)
 */
void tcp_retransmit_timer(struct sock *sk)
{
	struct tcp_sock *tp = tcp_sk(sk);
	struct net *net = sock_net(sk);
	struct inet_connection_sock *icsk = inet_csk(sk);
	struct request_sock *req;
	struct sk_buff *skb;
	...
	
out_reset_timer:
	if (sk->sk_state == TCP_ESTABLISHED &&
	    (tp->thin_lto || net->ipv4.sysctl_tcp_thin_linear_timeouts) &&
	    tcp_stream_is_thin(tp) &&
	    icsk->icsk_retransmits <= TCP_THIN_LINEAR_RETRIES) {
		icsk->icsk_backoff = 0;

	} else {
		/* Use normal (exponential) backoff */
		icsk->icsk_rto = min(icsk->icsk_rto << 1, TCP_RTO_MAX);
	}
	inet_csk_reset_xmit_timer(sk, ICSK_TIME_RETRANS,
				  tcp_clamp_rto_to_user_timeout(sk), TCP_RTO_MAX);
	if (retransmits_timed_out(sk, net->ipv4.sysctl_tcp_retries1 + 1, 0))
		__sk_dst_reset(sk);

out:;
}
```

- `TCP thin`일 때는 `icsk->icsk_rto`를 `__tcp_set_rto(tp)`로 설정
``` c
// include/net/tcp.h
static inline u32 __tcp_set_rto(const struct tcp_sock *tp)
{
	return usecs_to_jiffies((tp->srtt_us >> 3) + tp->rttvar_us);
}
```
		(smoothe rtt의 1/8배에다가 rtt변동성을 더해준 값)
- 일반적인 TCP일때는 이전 `icsk->icsk_rto`의 두배로 설정

그 후
``` c
inet_csk_reset_xmit_timer(sk, ICSK_TIME_RETRANS,
			  tcp_clamp_rto_to_user_timeout(sk), TCP_RTO_MAX);

// include/net/inet_connection_sock.h
static inline void inet_csk_reset_xmit_timer(struct sock *sk, const int what,
					     unsigned long when,
					     const unsigned long max_when)
{
	...
		icsk->icsk_pending = what;
		icsk->icsk_timeout = jiffies + when;
		sk_reset_timer(sk, &icsk->icsk_retransmit_timer, icsk->icsk_timeout);
	...
}

// net/core/sock.c
void sk_reset_timer(struct sock *sk, struct timer_list* timer,
		    unsigned long expires)
{
	...
	mod_timer(timer, expires)
	...
}
```

``` c
// net/input/tcp_timer.c
static u32 tcp_clamp_rto_to_user_timeout(const struct sock *sk)
{
	struct inet_connection_sock *icsk = inet_csk(sk);
	u32 elapsed, start_ts;
	s32 remaining;

	start_ts = tcp_sk(sk)->retrans_stamp;
	if (!icsk->icsk_user_timeout)
		return icsk->icsk_rto;
	elapsed = tcp_time_stamp(tcp_sk(sk)) - start_ts;
	remaining = icsk->icsk_user_timeout - elapsed;
	if (remaining <= 0)
		return 1; /* user timeout has passed; fire ASAP */

	return min_t(u32, icsk->icsk_rto, msecs_to_jiffies(remaining));
}
```

즉, `icsk_retransmit_timer의` `timeout`을 현재시간(`jiffies`)에다가 아까 수정한 `icsk->icsk_rto`을 더해서 업데이트

아무 문제 없는 상황에서는
``` c
// net/input/tcp_timer.c
/* Called with bottom-half processing disabled.
   Called by tcp_write_timer() */
void tcp_write_timer_handler(struct sock *sk)
{
	...
	if (time_after(icsk->icsk_timeout, jiffies)) {
		sk_reset_timer(sk, &icsk->icsk_retransmit_timer, icsk->icsk_timeout);
		goto out;
	}
	
	switch (event) {
	...	
	case ICSK_TIME_RETRANS:
		icsk->icsk_pending = 0;
		tcp_retransmit_timer(sk);
		break;
	...
out:
	sk_mem_reclaim(sk);
}
```

늘어난 타임아웃은
``` c
// net/ipv4/tcp_input.c
static void tcp_set_rto(struct sock *sk)
{
	const struct tcp_sock *tp = tcp_sk(sk);
	inet_csk(sk)->icsk_rto = __tcp_set_rto(tp);
	tcp_bound_rto(sk);
}
```
여기서 처리