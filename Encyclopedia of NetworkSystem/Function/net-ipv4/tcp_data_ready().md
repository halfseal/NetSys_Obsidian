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

>`sk->sk_data_ready`는 함수포인터이다. 대부분의 소켓들이 어떻게 초기화 되는지 확인해보았을 때, `net/core/sock.c`에서 `sock_init_data_uid()`라는 함수에서 `sock_def_readable()`로 초기화 되고 있는 것을 볼 수 있었다. 이 함수는 똑같은 파일에 있으며, rcu lock을 획득하고, 

