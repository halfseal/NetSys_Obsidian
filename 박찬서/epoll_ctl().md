---
Parameter:
  - epfd
  - op
  - fd
  - epoll_event
Return: int
Location: /fs/eventpoll.c
---
## epoll_ctl()

```c
/*

* The following function implements the controller interface for

* the eventpoll file that enables the insertion/removal/change of

* file descriptors inside the interest set.

*/

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

`epoll_ctl` 함수는 `epoll` 인터페이스에서 파일 디스크립터를 등록하거나 제거하거나 변경할 때 사용하는 시스템 호출이. 이 함수는 총 4개의 매개변수를 받으며, 각각 다음을 의미한다.

1. `epfd`: `epoll` 파일 디스크립터
2. `op`: 실행할 작업(등록, 수정, 제거)
3. `fd`: 대상 파일 디스크립터
4. `event`: 이벤트 구조체 (등록 및 변경 시 사용)

이 함수는 사용자 공간에서 전달된 이벤트 데이터를 커널로 복사한 뒤, `do_epoll_ctl()` 함수에서 실제 작업을 처리하게 한다.
## do_epoll_ctl()

```c title=do_epoll_ctl()

int do_epoll_ctl(int epfd, int op, int fd, struct epoll_event *epds,

bool nonblock)

{

int error;

int full_check = 0;

struct fd f, tf;

struct eventpoll *ep;

struct epitem *epi;

struct eventpoll *tep = NULL;

  

error = -EBADF;

f = fdget(epfd);

if (!f.file)

goto error_return;

  

/* Get the "struct file *" for the target file */

tf = fdget(fd);

if (!tf.file)

goto error_fput;

  

/* The target file descriptor must support poll */

error = -EPERM;

if (!file_can_poll(tf.file))

goto error_tgt_fput;

  

/* Check if EPOLLWAKEUP is allowed */

if (ep_op_has_event(op))

ep_take_care_of_epollwakeup(epds);

  

/*

* We have to check that the file structure underneath the file descriptor

* the user passed to us _is_ an eventpoll file. And also we do not permit

* adding an epoll file descriptor inside itself.

*/

error = -EINVAL;

if (f.file == tf.file || !is_file_epoll(f.file))

goto error_tgt_fput;

  

/*

* epoll adds to the wakeup queue at EPOLL_CTL_ADD time only,

* so EPOLLEXCLUSIVE is not allowed for a EPOLL_CTL_MOD operation.

* Also, we do not currently supported nested exclusive wakeups.

*/

if (ep_op_has_event(op) && (epds->events & EPOLLEXCLUSIVE)) {

if (op == EPOLL_CTL_MOD)

goto error_tgt_fput;

if (op == EPOLL_CTL_ADD && (is_file_epoll(tf.file) ||

(epds->events & ~EPOLLEXCLUSIVE_OK_BITS)))

goto error_tgt_fput;

}

  

/*

* At this point it is safe to assume that the "private_data" contains

* our own data structure.

*/

ep = f.file->private_data;

  

/*

* When we insert an epoll file descriptor inside another epoll file

* descriptor, there is the chance of creating closed loops, which are

* better be handled here, than in more critical paths. While we are

* checking for loops we also determine the list of files reachable

* and hang them on the tfile_check_list, so we can check that we

* haven't created too many possible wakeup paths.

*

* We do not need to take the global 'epumutex' on EPOLL_CTL_ADD when

* the epoll file descriptor is attaching directly to a wakeup source,

* unless the epoll file descriptor is nested. The purpose of taking the

* 'epnested_mutex' on add is to prevent complex toplogies such as loops and

* deep wakeup paths from forming in parallel through multiple

* EPOLL_CTL_ADD operations.

*/

error = epoll_mutex_lock(&ep->mtx, 0, nonblock);

if (error)

goto error_tgt_fput;

if (op == EPOLL_CTL_ADD) {

if (READ_ONCE(f.file->f_ep) || ep->gen == loop_check_gen ||

is_file_epoll(tf.file)) {

mutex_unlock(&ep->mtx);

error = epoll_mutex_lock(&epnested_mutex, 0, nonblock);

if (error)

goto error_tgt_fput;

loop_check_gen++;

full_check = 1;

if (is_file_epoll(tf.file)) {

tep = tf.file->private_data;

error = -ELOOP;

if (ep_loop_check(ep, tep) != 0)

goto error_tgt_fput;

}

error = epoll_mutex_lock(&ep->mtx, 0, nonblock);

if (error)

goto error_tgt_fput;

}

}

  

/*

* Try to lookup the file inside our RB tree. Since we grabbed "mtx"

* above, we can be sure to be able to use the item looked up by

* ep_find() till we release the mutex.

*/

epi = ep_find(ep, tf.file, fd);

  

error = -EINVAL;

switch (op) {

case EPOLL_CTL_ADD:

if (!epi) {

epds->events |= EPOLLERR | EPOLLHUP;

error = ep_insert(ep, epds, tf.file, fd, full_check);

} else

error = -EEXIST;

break;

case EPOLL_CTL_DEL:

if (epi) {

/*

* The eventpoll itself is still alive: the refcount

* can't go to zero here.

*/

ep_remove_safe(ep, epi);

error = 0;

} else {

error = -ENOENT;

}

break;

case EPOLL_CTL_MOD:

if (epi) {

if (!(epi->event.events & EPOLLEXCLUSIVE)) {

epds->events |= EPOLLERR | EPOLLHUP;

error = ep_modify(ep, epi, epds);

}

} else

error = -ENOENT;

break;

}

mutex_unlock(&ep->mtx);

  

error_tgt_fput:

if (full_check) {

clear_tfile_check_list();

loop_check_gen++;

mutex_unlock(&epnested_mutex);

}

  

fdput(tf);

error_fput:

fdput(f);

error_return:

  

return error;

}
```

`do_epoll_ctl` 함수는 `epoll_ctl` 시스템 호출의 구체적인 구현으로, 파일 디스크립터(파일, 소켓 등)를 epoll 이벤트 감시 목록에 추가, 삭제, 수정하는 작업을 수행한다.

1. **파일 디스크립터 검증:** `epfd`와 `fd` 두 개의 파일 디스크립터를 가져오고, epoll에 적합한지 검사한다. `fdget` 함수를 통해 파일 구조체를 얻고, 해당 파일이 유효한지 확인한다.
    
2. **유효성 검사:** 대상 파일(`fd`)이 epoll 파일(`epfd`)인지, 적절한 이벤트를 지원하는지 검증한다. 추가하려는 파일이 epoll 자체이거나, 중복된 파일이거나, 이벤트가 적합하지 않으면 오류가 발생한다.
    
3. **동작 수행:** epoll이 실행할 작업을 결정하는 `op`에 따라, epoll에 파일을 추가(EPOLL_CTL_ADD), 제거(EPOLL_CTL_DEL), 수정(EPOLL_CTL_MOD)할 수 있다.
    
    - **EPOLL_CTL_ADD:** 대상 파일이 이미 epoll에 추가되지 않았으면 새로운 항목을 삽입한다.
    - **EPOLL_CTL_DEL:** epoll에서 해당 파일을 안전하게 제거한다.
    - **EPOLL_CTL_MOD:** 이미 추가된 파일에 대해 감시 이벤트를 수정한다.
      
4. **락 처리:** 에러 발생 시 잠금 해제, 파일 참조 해제 작업을 수행하여 리소스를 정리한다.
    

`do_epoll_ctl` 함수는 epoll이 파일 디스크립터의 이벤트를 감시하도록 설정하며, epoll의 복잡한 구조(예: 중첩된 epoll이나 루프)를 처리하는 추가적인 검사를 수행한다.