```c
/* The per-socket spinlock must be held here. */
static inline __must_check int sk_add_backlog(struct sock *sk, struct sk_buff *skb,
					      unsigned int limit)
{
	if (sk_rcvqueues_full(sk, limit))
		return -ENOBUFS;

	/*
	 * If the skb was allocated from pfmemalloc reserves, only
	 * allow SOCK_MEMALLOC sockets to use it as this socket is
	 * helping free memory
	 */
	if (skb_pfmemalloc(skb) && !sock_flag(sk, SOCK_MEMALLOC))
		return -ENOMEM;

	__sk_add_backlog(sk, skb);
	sk->sk_backlog.len += skb->truesize;
	return 0;
}
```



```c
/* OOB backlog add */
static inline void __sk_add_backlog(struct sock *sk, struct sk_buff *skb)
{
	/* dont let skb dst not refcounted, we are going to leave rcu lock */
	skb_dst_force(skb);

	if (!sk->sk_backlog.tail)
		WRITE_ONCE(sk->sk_backlog.head, skb);
	else
		sk->sk_backlog.tail->next = skb;

	WRITE_ONCE(sk->sk_backlog.tail, skb);
	skb->next = NULL;
}
```