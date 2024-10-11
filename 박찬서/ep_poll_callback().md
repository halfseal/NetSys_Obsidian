---
Parameter:
  - wait_queue_entry_t
  - unsigned
  - int
  - void
Return: int
Location: /fs/eventpoll.c
---


```c title-ep_poll_callback()
/*

* This is the callback that is passed to the wait queue wakeup

* mechanism. It is called by the stored file descriptors when they

* have events to report.

*

* This callback takes a read lock in order not to contend with concurrent

* events from another file descriptor, thus all modifications to ->rdllist

* or ->ovflist are lockless. Read lock is paired with the write lock from

* ep_start/done_scan(), which stops all list modifications and guarantees

* that lists state is seen correctly.

*

* Another thing worth to mention is that ep_poll_callback() can be called

* concurrently for the same @epi from different CPUs if poll table was inited

* with several wait queues entries. Plural wakeup from different CPUs of a

* single wait queue is serialized by wq.lock, but the case when multiple wait

* queues are used should be detected accordingly. This is detected using

* cmpxchg() operation.

*/

static int ep_poll_callback(wait_queue_entry_t *wait, unsigned mode, int sync, void *key)

{

int pwake = 0;

struct epitem *epi = ep_item_from_wait(wait);

struct eventpoll *ep = epi->ep;

__poll_t pollflags = key_to_poll(key);

unsigned long flags;

int ewake = 0;

  

read_lock_irqsave(&ep->lock, flags);

  

ep_set_busy_poll_napi_id(epi);

  

/*

* If the event mask does not contain any poll(2) event, we consider the

* descriptor to be disabled. This condition is likely the effect of the

* EPOLLONESHOT bit that disables the descriptor when an event is received,

* until the next EPOLL_CTL_MOD will be issued.

*/

if (!(epi->event.events & ~EP_PRIVATE_BITS))

goto out_unlock;

  

/*

* Check the events coming with the callback. At this stage, not

* every device reports the events in the "key" parameter of the

* callback. We need to be able to handle both cases here, hence the

* test for "key" != NULL before the event match test.

*/

if (pollflags && !(pollflags & epi->event.events))

goto out_unlock;

  

/*

* If we are transferring events to userspace, we can hold no locks

* (because we're accessing user memory, and because of linux f_op->poll()

* semantics). All the events that happen during that period of time are

* chained in ep->ovflist and requeued later on.

*/

if (READ_ONCE(ep->ovflist) != EP_UNACTIVE_PTR) {

if (chain_epi_lockless(epi))

ep_pm_stay_awake_rcu(epi);

} else if (!ep_is_linked(epi)) {

/* In the usual case, add event to ready list. */

if (list_add_tail_lockless(&epi->rdllink, &ep->rdllist))

ep_pm_stay_awake_rcu(epi);

}

  

/*

* Wake up ( if active ) both the eventpoll wait list and the ->poll()

* wait list.

*/

if (waitqueue_active(&ep->wq)) {

if ((epi->event.events & EPOLLEXCLUSIVE) &&

!(pollflags & POLLFREE)) {

switch (pollflags & EPOLLINOUT_BITS) {

case EPOLLIN:

if (epi->event.events & EPOLLIN)

ewake = 1;

break;

case EPOLLOUT:

if (epi->event.events & EPOLLOUT)

ewake = 1;

break;

case 0:

ewake = 1;

break;

}

}

wake_up(&ep->wq);

}

if (waitqueue_active(&ep->poll_wait))

pwake++;

  

out_unlock:

read_unlock_irqrestore(&ep->lock, flags);

  

/* We have to call this outside the lock */

if (pwake)

ep_poll_safewake(ep, epi, pollflags & EPOLL_URING_WAKE);

  

if (!(epi->event.events & EPOLLEXCLUSIVE))

ewake = 1;

  

if (pollflags & POLLFREE) {

/*

* If we race with ep_remove_wait_queue() it can miss

* ->whead = NULL and do another remove_wait_queue() after

* us, so we can't use __remove_wait_queue().

*/

list_del_init(&wait->entry);

/*

* ->whead != NULL protects us from the race with

* ep_clear_and_put() or ep_remove(), ep_remove_wait_queue()

* takes whead->lock held by the caller. Once we nullify it,

* nothing protects ep/epi or even wait.

*/

smp_store_release(&ep_pwq_from_wait(wait)->whead, NULL);

}

  

return ewake;

}
```

`ep_poll_callback` 함수는 `wait_queue`에서 이벤트가 발생했을 때 실행되는 콜백 함수이다. 이 함수의 목적은 파일 디스크립터에서 발생한 이벤트를 감지하고, 이를 `epoll` 시스템에서 처리하기 위한 준비를 하는 것이다. 

1. **int pwake = 0;**  
    `pwake`는 `poll_wait` 대기열을 깨울지 여부를 나타내는 플래그이다.
    
2. *_struct epitem _epi = ep_item_from_wait(wait);
    `wait_queue`에서 이벤트를 감지한 후, 이를 처리할 `epitem` 객체를 얻어온다. `epitem`은 `epoll`에 등록된 이벤트를 관리하는 구조체이다.
    
3. *_struct eventpoll _ep = epi->ep; 
    `epitem`에서 `eventpoll` 구조체를 가져온다. 이 구조체는 `epoll`에서 관리하는 전체 이벤트 목록을 포함한다.
    
4. **__poll_t pollflags = key_to_poll(key);**  
    이벤트를 감지한 후, 해당 이벤트의 종류를 `pollflags`에 저장한다.
    
5. **read_lock_irqsave(&ep->lock, flags);**  
    동시성 문제를 방지하기 위해 `epoll`의 읽기 잠금을 설정한다. 이 잠금은 다른 CPU나 스레드에서 이벤트 목록을 동시에 수정하는 것을 방지한다.
    
6. **if (!(epi->event.events & ~EP_PRIVATE_BITS)) goto out_unlock;**  
    등록된 이벤트가 없으면 콜백을 종료하고, 잠금을 해제한다.
    
7. **if (pollflags && !(pollflags & epi->event.events)) goto out_unlock;**  
    이벤트가 발생했는지 확인한다. 발생한 이벤트가 사용자 요청과 일치하지 않으면 콜백을 종료한다.
    
8. **if (READ_ONCE(ep->ovflist) != EP_UNACTIVE_PTR)**  
    이벤트가 사용자로 전달되는 동안, 잠금을 해제한 후 발생한 추가 이벤트는 `ovflist`에 기록된다. 이 목록에 이벤트가 있으면 적절히 처리한다.
    
9. **list_add_tail_lockless(&epi->rdllink, &ep->rdllist);**  
    이벤트가 발생하면 `rdllist`에 이벤트를 추가한다. 이 목록은 이후 `epoll_wait` 호출에서 이벤트를 확인하는 데 사용된다.
    
10. **wake_up(&ep->wq);**  
    이벤트가 발생했음을 알리기 위해 대기열에 있는 모든 스레드를 깨운다.
    
11. **read_unlock_irqrestore(&ep->lock, flags);**  
    이벤트 처리가 끝나면 잠금을 해제하고 인터럽트를 다시 활성화한다.
    
12. **if (pwake) ep_poll_safewake(ep, epi, pollflags & EPOLL_URING_WAKE);**  
    `poll_wait` 대기열이 활성화된 경우, 이벤트가 발생했음을 안전하게 알린다.
    

이 함수는 `wait_queue`에서 이벤트가 발생했을 때 자동으로 호출되며, 이벤트를 `epoll` 시스템에서 처리할 수 있도록 준비하는 역할을 한다.

`wait_queue`에서 이벤트가 발생하면 대기 중이던 프로세스나 스레드를 깨우는 동작이 일어나는데, 이때 등록된 콜백 함수인 `ep_poll_callback`이 호출된다. 이 함수는 이벤트를 처리하고, 준비된 이벤트가 있는지 확인한 후 이를 이벤트풀에 추가하여 사용자에게 이벤트가 발생했음을 알리는 역할을 한다.

`ep_poll_callback` 함수가 등록되는 과정은 `epoll_ctl` 시스템 호출에서 이루어진다. 파일 디스크립터를 `epoll`에 추가할 때, `epoll_ctl` 함수가 호출되고, 해당 파일 디스크립터에 대한 이벤트 발생 시 호출될 콜백 함수가 등록된다.

이 과정에서, 특정 파일 디스크립터에 대해 관심 있는 이벤트가 설정되면, 해당 파일 디스크립터가 `wait_ queue`에 등록되고, 그때 `ep_poll_callback`이 콜백 함수로 설정된다. 이후 파일 디스크립터에서 이벤트가 발생하면 `wait queue`에 등록된 대기 항목이 깨워지면서, `ep_poll_callback` 함수가 호출된다.

1. `epoll_ctl` 시스템 호출을 통해 파일 디스크립터가 `epoll`에 등록된다.
2. 파일 디스크립터에 대한 폴링(polling)이 이루어지며, `poll` 테이블에 `ep_poll_callback`이 콜백 함수로 등록된다.
3. 파일 디스크립터에 이벤트가 발생하면 `wait queue`에서 `ep_poll_callback`이 호출되어 이벤트가 처리된다.

이렇게 함으로써 `epoll`은 비동기적으로 파일 디스크립터의 상태 변화를 감지하고, 적절한 이벤트가 발생했을 때 콜백을 통해 이를 처리할 수 있게 된다.

