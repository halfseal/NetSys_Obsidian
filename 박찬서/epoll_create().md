---
Parameter:
  - int
Return: int
Location: /fs/eventpoll.c
---
## epoll_create()

```c
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,

struct epoll_event __user *, event)

{

struct epoll_event epds;

  

if (ep_op_has_event(op) &&

copy_from_user(&epds, event, sizeof(struct epoll_event)))

return -EFAULT;

  

return do_epoll_ctl(epfd, op, fd, &epds, false);

}
```

`epoll_create` 시스템 호출의 정의로, `size`라는 인자를 받아 `epoll` 인스턴스를 생성하는 작업을 수행한다. 만약 `size`가 0보다 작거나 같으면 잘못된 인자라는 의미로 `-EINVAL` 오류를 반환한다.

그 외의 경우에는 `do_epoll_create` 함수가 호출되어 `epoll` 인스턴스 생성 작업을 처리한다. `do_epoll_create(0)`은 내부적으로 `epoll` 자료구조를 생성하고 초기화하는 역할을 담당한다.

## do_epoll_create()

```c title=do_epoll_create()
/*

* Open an eventpoll file descriptor.

*/

static int do_epoll_create(int flags)

{

int error, fd;

struct eventpoll *ep = NULL;

struct file *file;

  

/* Check the EPOLL_* constant for consistency. */

BUILD_BUG_ON(EPOLL_CLOEXEC != O_CLOEXEC);

  

if (flags & ~EPOLL_CLOEXEC)

return -EINVAL;

/*

* Create the internal data structure ("struct eventpoll").

*/

error = ep_alloc(&ep);

if (error < 0)

return error;

/*

* Creates all the items needed to setup an eventpoll file. That is,

* a file structure and a free file descriptor.

*/

fd = get_unused_fd_flags(O_RDWR | (flags & O_CLOEXEC));

if (fd < 0) {

error = fd;

goto out_free_ep;

}

file = anon_inode_getfile("[eventpoll]", &eventpoll_fops, ep,

O_RDWR | (flags & O_CLOEXEC));

if (IS_ERR(file)) {

error = PTR_ERR(file);

goto out_free_fd;

}

#ifdef CONFIG_NET_RX_BUSY_POLL

ep->busy_poll_usecs = 0;

ep->busy_poll_budget = 0;

ep->prefer_busy_poll = false;

#endif

ep->file = file;

fd_install(fd, file);

return fd;

  

out_free_fd:

put_unused_fd(fd);

out_free_ep:

ep_clear_and_put(ep);

return error;

}
```

 `epoll_create` 시스템 호출이 내부적으로 어떻게 `epoll` 파일 디스크립터를 생성하는지 보여준다.

1. **ep_alloc(&ep)**: `epoll`의 내부 데이터 구조인 `eventpoll`을 할당한다.
2. **get_unused_fd_flags**: 새로운 파일 디스크립터를 확보한다.
3. **anon_inode_getfile**: 익명 파일을 생성하고 `epoll` 파일로 설정한다.
4. **fd_install**: 파일 디스크립터를 파일 시스템에 설치한다.
5. 만약 에러가 발생하면 적절한 클린업을 수행한 후, 에러 코드를 반환한다.

이 과정에서 `eventpoll` 구조체와 관련된 설정이 이루어진다.

## ep_alloc()

```c title=ep_alloc()
static int ep_alloc(struct eventpoll **pep)

{

struct eventpoll *ep;

  

ep = kzalloc(sizeof(*ep), GFP_KERNEL);

if (unlikely(!ep))

return -ENOMEM;

  

mutex_init(&ep->mtx);

rwlock_init(&ep->lock);

init_waitqueue_head(&ep->wq);

init_waitqueue_head(&ep->poll_wait);

INIT_LIST_HEAD(&ep->rdllist);

ep->rbr = RB_ROOT_CACHED;

ep->ovflist = EP_UNACTIVE_PTR;

ep->user = get_current_user();

refcount_set(&ep->refcount, 1);

  

*pep = ep;

  

return 0;

}
```

`ep_alloc` 함수는 `eventpoll` 구조체를 할당하고 초기화하는 함수이다. 이 함수는 `epoll`의 내부 자료 구조인 `eventpoll` 객체를 생성하고 필요한 필드를 초기화한다.

1. **메모리 할당**: `eventpoll` 크기만큼 메모리를 할당 (`kzalloc`)하고, 할당 실패 시 에러 코드 (`-ENOMEM`)를 반환합니다.
2. **락 및 대기열 초기화**: 구조체 내에 필요한 `mutex`, `rwlock`, `waitqueue`를 초기화한다.
3. **이벤트 리스트 초기화**: `rdllist`, `rbr`, `ovflist`를 초기화한다.
4. **사용자 정보 초기화**: 현재 프로세스의 사용자를 저장하고, 참조 카운트를 1로 설정한다.

이렇게 생성된 `eventpoll` 객체는 이후 `epoll` 관련 작업에 사용됩니다.