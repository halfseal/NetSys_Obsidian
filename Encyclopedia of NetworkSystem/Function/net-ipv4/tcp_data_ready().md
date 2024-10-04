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

>`sk->sk_data_ready`는 함수포인터이다. 대부분의 소켓들이 어떻게 초기화 되는지 확인해보았을 때, `net/core/sock.c`에서 `sock_init_data_uid()`라는 함수에서 `sock_def_readable()`로 초기화 되고 있는 것을 볼 수 있었다. 
>	이 함수는 똑같은 파일에 있으며, rcu lock을 획득하고, 소켓의 `sk_wq`를 가져오게 된다.
>	이를 바탕으로 `skwq_has_sleeper()`함수를 실행하게 되는데, 이 함수는 만약 기다리고 있는 프로세스가 있는지 확인하는 함수이다. 만약 받은 `socket_wq`가 waiting processes를 가지고 있다면 true를 반환하게 된다. 이후 wake_up_interruptibal_sync_poll()함수를 실행하게 된다.
>	 그리고 난 다음에 `sk_wake_async()`함수를 실행하게 되고, rcu lock을 해제하게 된다.
>

sk_data_ready() -> [[sock_def_readable()]]


