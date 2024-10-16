
/net/socket.c
```c
/*
* Receive a datagram from a socket.
*/
  
SYSCALL_DEFINE4(recv, int, fd, void __user *, ubuf, size_t, size,
unsigned int, flags)
{
return __sys_recvfrom(fd, ubuf, size, flags, NULL, NULL);
}
```

>recv(), recvfrom() 두 시스템 콜 모두 `__sys_recvfrom()`함수를 호출하고 있었다.

```c
/*
 *  Receive a frame from the socket and optionally record the address of the
 *  sender. We verify the buffers are writable and if needed move the
 *  sender address from kernel to user space.
 */
int __sys_recvfrom(int fd, void __user *ubuf, size_t size, unsigned int flags,
           struct sockaddr __user *addr, int __user *addr_len)
{
    struct sockaddr_storage address;
    struct msghdr msg = {
        /* Save some cycles and don't copy the address if not needed */
        .msg_name = addr ? (struct sockaddr *)&address : NULL,
    };
    struct socket *sock;
    int err, err2;
    int fput_needed;
  
    err = import_ubuf(ITER_DEST, ubuf, size, &msg.msg_iter);
    if (unlikely(err))
        return err;
    sock = sockfd_lookup_light(fd, &err, &fput_needed);
    if (!sock)
        goto out;
  
    if (sock->file->f_flags & O_NONBLOCK)
        flags |= MSG_DONTWAIT;
    err = sock_recvmsg(sock, &msg, flags);
  
    if (err >= 0 && addr != NULL) {
        err2 = move_addr_to_user(&address,
                     msg.msg_namelen, addr, addr_len);
        if (err2 < 0)
            err = err2;
    }
    
    fput_light(sock->file, fput_needed);
out:
    return err;
}
```

>5, 6번째 parameter의 경우 `recvfrom`으로 호출되었을 때 사용한다. 이는 UDP와 같은 비 연결지향성 통신에서 어디서 왔는지를 확인하기 위해 사용된다.
>
>중간에`import_ubuf()`를 호출하여 `len`이 overflow값을 넘기지 않게 정리하고 `iov_iter_ubuf()`를 내부적으로 호출하여 `iov_iter` 구조체에 `ubuf`를 세팅하게 된다.
>
>이후 `sockfd_lookup_light()`함수를 통해 인자로 받은 `fd`를 소켓으로 바꾸어 반환하고, 만약 유효하지 않은 `fd`라면 함수가 err를 반환하고 종료된다.
>
>이후 반환 받은 sock을 바탕으로 `sock_recvmsg()`함수를 호출하게 된다.
>만약 에러가 없고 5번째 인자값이 존재하는 경우(`recvfrom()` 시스템콜로 호출 된 경우일 것이다.) `move_addr_to_user()`를 호출하여 송신자의 주소를 받아와서 해당 `addr` 공간에 넣어주게 된다.
>여기서 지켜봐야할 것은 sock_recvmsg 함수안에서 msghdr의 name을 설정해주는지 여부이다.
>`address`로 선언한 송신주소 저장소는 `msghdr->msg_name`에 포인터로 저장되어 있는데, `udp_recvmsg()`에서 이 address field를 세팅하는 부분이 있다. 따라서 이 값을 반환하게 되는 것이다.
>
>이후 file handler를 닫고 함수가 종료된다.

### sockfd_lookup / sockfd_lookup_light
```c
 /**

 *  sockfd_lookup - Go from a file number to its socket slot

 *  @fd: file handle

 *  @err: pointer to an error code return

 *

 *  The file handle passed in is locked and the socket it is bound

 *  to is returned. If an error occurs the err pointer is overwritten

 *  with a negative errno code and NULL is returned. The function checks

 *  for both invalid handles and passing a handle which is not a socket.

 *

 *  On a success the socket object pointer is returned.

 */

  

struct socket *sockfd_lookup(int fd, int *err)

{

    struct file *file;

    struct socket *sock;

  

    file = fget(fd);

    if (!file) {

        *err = -EBADF;

        return NULL;

    }

  

    sock = sock_from_file(file);

    if (!sock) {

        *err = -ENOTSOCK;

        fput(file);

    }

    return sock;

}

EXPORT_SYMBOL(sockfd_lookup);

  

static struct socket *sockfd_lookup_light(int fd, int *err, int *fput_needed)

{

    struct fd f = fdget(fd);

    struct socket *sock;

  

    *err = -EBADF;

    if (f.file) {

        sock = sock_from_file(f.file);

        if (likely(sock)) {

            *fput_needed = f.flags & FDPUT_FPUT;

            return sock;

        }

        *err = -ENOTSOCK;

        fdput(f);

    }

    return NULL;

}
```

> 둘이 무슨 차이인지는 모르겠다. 근데 socket.c에서 보면 다 light만 쓴다.