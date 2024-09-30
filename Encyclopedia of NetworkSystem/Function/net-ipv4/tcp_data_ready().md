---
Parameter:
  - sock
Return: void
Location: /net/ipv4/tcp_input.c
---
```c
void tcp_data_ready(struct sock *sk)
{
	if (tcp_epollin_ready(sk, sk->sk_rcvlowat) || sock_flag(sk, SOCK_DONE))
		sk->sk_data_ready(sk);
}
```

