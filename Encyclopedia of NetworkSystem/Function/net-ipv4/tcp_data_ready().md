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
`sk_data_ready()` 함수는 socket내부에 존재하는 함수이다. 
`tcp_epollin_ready()` 함수로 epoll이 가능한지 확인해본다.
socket안에 처리해야할 데이터가 있음을 알려주는 callback 함수다. 

The `sk_data_ready` function points to the `sock_def_readable` function, which calls `wake_up_interruptible_sync_poll` to wake up the process that is waiting in `ep_wait`. The `ep_wait` then delivers epoll events and returns to userspace, which can then call `recv` system call, for example, to obtain new data that was packed in socket buffer
@sk_data_ready: callback to indicate there is data to be processed_|
Check if we need to signal EPOLLIN right now */

[[sock_def_readable()]]