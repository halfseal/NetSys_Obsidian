```c
static inline void sock_rps_save_rxhash(struct sock *sk,
					const struct sk_buff *skb)
{
#ifdef CONFIG_RPS
	/* The following WRITE_ONCE() is paired with the READ_ONCE()
	 * here, and another one in sock_rps_record_flow().
	 */
	if (unlikely(READ_ONCE(sk->sk_rxhash) != skb->hash))
		WRITE_ONCE(sk->sk_rxhash, skb->hash);
#endif
}
```

sk_rxhash 변수를 skb->hash의 값으로 수정해준다. rps flow table과 연관되어있는 줄 알았는데...
`#ifdef CONFIG_RPS` 가 설정되어있는 거 보면 rps에 쓰이긴하는듯 아님말고

