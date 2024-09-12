소켓 버퍼는 kernel 상에 존재함.

write 함수가 사용되면, user space의 해당 application에서 write하여 user space 버퍼에 있는 것을 kernel space로 복사함.

이 때 메모리를 할당/해제 하며, 만약 버퍼가 다 찬 경우 Lock을 통해 overflow를 방지한다.

send buffer와 receive buffer를 따로 가지고 있어 duplex communication이 가능하다.