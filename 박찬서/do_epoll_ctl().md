---
Parameter:
  - int
  - epoll_event
  - bool
Return: int
Location: /fs/eventpoll.c
---

```c title=do_epoll_ctl()
```c
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
```

[[Encyclopedia of NetworkSystem/Function/net-ipv4/tcp_gro_complete().md|tcp_gro_complete()]]
