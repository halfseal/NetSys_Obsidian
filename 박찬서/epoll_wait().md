---
Parameter:
  - epfd
  - epoll_event
  - int
Return: int
Location: /fs/eventpoll.c
---
## epoll_wait()

```c
SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events,

int, maxevents, int, timeout)

{

struct timespec64 to;

  

return do_epoll_wait(epfd, events, maxevents,

ep_timeout_to_timespec(&to, timeout));

}
```

이 시스템 호출은 epoll 감시 목록에 있는 파일 디스크립터 중에서 이벤트가 발생하는지를 확인하고, 발생한 이벤트를 반환한다.

1. 매개변수로 epoll 파일 디스크립터 `epfd`, 이벤트 구조체 배열 `events`, 최대 이벤트 개수 `maxevents`, 그리고 대기 시간인 `timeout`을 받는다.

2. `ep_timeout_to_timespec` 함수를 사용해 `timeout` 값을 `timespec64` 형식으로 변환하여 `do_epoll_wait` 함수로 넘긴다.
   
3.  `do_epoll_wait`를 호출하여 실질적인 이벤트 대기를 처리한다.

## do_epoll_wait()

```c title=do_epoll_wait
/*

* Implement the event wait interface for the eventpoll file. It is the kernel

* part of the user space epoll_wait(2).

*/

static int do_epoll_wait(int epfd, struct epoll_event __user *events,

int maxevents, struct timespec64 *to)

{

int error;

struct fd f;

struct eventpoll *ep;

  

/* The maximum number of event must be greater than zero */

if (maxevents <= 0 || maxevents > EP_MAX_EVENTS)

return -EINVAL;

  

/* Verify that the area passed by the user is writeable */

if (!access_ok(events, maxevents * sizeof(struct epoll_event)))

return -EFAULT;

  

/* Get the "struct file *" for the eventpoll file */

f = fdget(epfd);

if (!f.file)

return -EBADF;

  

/*

* We have to check that the file structure underneath the fd

* the user passed to us _is_ an eventpoll file.

*/

error = -EINVAL;

if (!is_file_epoll(f.file))

goto error_fput;

  

/*

* At this point it is safe to assume that the "private_data" contains

* our own data structure.

*/

ep = f.file->private_data;

  

/* Time to fish for events ... */

error = ep_poll(ep, events, maxevents, to);

  

error_fput:

fdput(f);

return error;

}
```

`do_epoll_wait` 함수는 `epoll_wait()` 시스템 호출의 커널 구현 부분이다. 주요 역할은 사용자 공간에서 제공된 `epfd`, `events`, `maxevents`, `timeout` 등을 바탕으로 이벤트를 처리하는 것이다.

1. **입력값 검증**: `maxevents`가 0보다 크고 최대 허용 이벤트 수 이하인지 확인한다. `events` 메모리 영역이 사용자에 의해 적절하게 쓰기가 가능한지 검사한다.
   
2. **파일 디스크립터 확인**: 전달된 `epfd`가 유효한 `epoll` 파일인지 확인한다.
   
3. **이벤트 처리**: 모든 검증을 통과하면 `ep_poll()` 함수 호출로 실제 이벤트 처리를 수행한다.
   
4. **결과 반환**: 결과 값을 반환하며, 에러가 발생할 경우 적절한 오류 코드를 반환한다.

## ep_poll()
 
```c title=ep_poll()
/**

* ep_poll - Retrieves ready events, and delivers them to the caller-supplied

* event buffer.

*

* @ep: Pointer to the eventpoll context.

* @events: Pointer to the userspace buffer where the ready events should be

* stored.

* @maxevents: Size (in terms of number of events) of the caller event buffer.

* @timeout: Maximum timeout for the ready events fetch operation, in

* timespec. If the timeout is zero, the function will not block,

* while if the @timeout ptr is NULL, the function will block

* until at least one event has been retrieved (or an error

* occurred).

*

* Return: the number of ready events which have been fetched, or an

* error code, in case of error.

*/

static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,

int maxevents, struct timespec64 *timeout)

{

int res, eavail, timed_out = 0;

u64 slack = 0;

wait_queue_entry_t wait;

ktime_t expires, *to = NULL;

  

lockdep_assert_irqs_enabled();

  

if (timeout && (timeout->tv_sec | timeout->tv_nsec)) {

slack = select_estimate_accuracy(timeout);

to = &expires;

*to = timespec64_to_ktime(*timeout);

} else if (timeout) {

/*

* Avoid the unnecessary trip to the wait queue loop, if the

* caller specified a non blocking operation.

*/

timed_out = 1;

}

  

/*

* This call is racy: We may or may not see events that are being added

* to the ready list under the lock (e.g., in IRQ callbacks). For cases

* with a non-zero timeout, this thread will check the ready list under

* lock and will add to the wait queue. For cases with a zero

* timeout, the user by definition should not care and will have to

* recheck again.

*/

eavail = ep_events_available(ep);

  

while (1) {

if (eavail) {

/*

* Try to transfer events to user space. In case we get

* 0 events and there's still timeout left over, we go

* trying again in search of more luck.

*/

res = ep_send_events(ep, events, maxevents);

if (res)

return res;

}

  

if (timed_out)

return 0;

  

eavail = ep_busy_loop(ep, timed_out);

if (eavail)

continue;

  

if (signal_pending(current))

return -EINTR;

  

/*

* Internally init_wait() uses autoremove_wake_function(),

* thus wait entry is removed from the wait queue on each

* wakeup. Why it is important? In case of several waiters

* each new wakeup will hit the next waiter, giving it the

* chance to harvest new event. Otherwise wakeup can be

* lost. This is also good performance-wise, because on

* normal wakeup path no need to call __remove_wait_queue()

* explicitly, thus ep->lock is not taken, which halts the

* event delivery.

*

* In fact, we now use an even more aggressive function that

* unconditionally removes, because we don't reuse the wait

* entry between loop iterations. This lets us also avoid the

* performance issue if a process is killed, causing all of its

* threads to wake up without being removed normally.

*/

init_wait(&wait);

wait.func = ep_autoremove_wake_function;

  

write_lock_irq(&ep->lock);

/*

* Barrierless variant, waitqueue_active() is called under

* the same lock on wakeup ep_poll_callback() side, so it

* is safe to avoid an explicit barrier.

*/

__set_current_state(TASK_INTERRUPTIBLE);

  

/*

* Do the final check under the lock. ep_start/done_scan()

* plays with two lists (->rdllist and ->ovflist) and there

* is always a race when both lists are empty for short

* period of time although events are pending, so lock is

* important.

*/

eavail = ep_events_available(ep);

if (!eavail)

__add_wait_queue_exclusive(&ep->wq, &wait);

  

write_unlock_irq(&ep->lock);

  

if (!eavail)

timed_out = !schedule_hrtimeout_range(to, slack,

HRTIMER_MODE_ABS);

__set_current_state(TASK_RUNNING);

  

/*

* We were woken up, thus go and try to harvest some events.

* If timed out and still on the wait queue, recheck eavail

* carefully under lock, below.

*/

eavail = 1;

  

if (!list_empty_careful(&wait.entry)) {

write_lock_irq(&ep->lock);

/*

* If the thread timed out and is not on the wait queue,

* it means that the thread was woken up after its

* timeout expired before it could reacquire the lock.

* Thus, when wait.entry is empty, it needs to harvest

* events.

*/

if (timed_out)

eavail = list_empty(&wait.entry);

__remove_wait_queue(&ep->wq, &wait);

write_unlock_irq(&ep->lock);

}

}

}
```

`ep_poll` 함수는 `epoll_wait`의 실제 이벤트 처리 부분이다. 이 함수는 `eventpoll` 구조체에서 준비된 이벤트들을 가져와 사용자 공간의 이벤트 버퍼에 전달하는 역할을 한다. 이 함수의 동작 과정은 다음과 같다.

1. **타임아웃 설정**: `timeout` 값이 주어졌다면, 이 값을 기준으로 타이머 설정 한다.
    
2. **이벤트 체크**: 현재 등록된 파일들에서 대기 중인 이벤트가 있는지 확인한다(`ep_events_available`).
    
3. **이벤트 전송**: 준비된 이벤트가 있으면 사용자 공간으로 이벤트를 전달한다(`ep_send_events`).
    
4. **대기 큐 처리**: 만약 이벤트가 없으면 `wait_queue`에 프로세스를 추가하여 이벤트가 발생할 때까지 대기한다. 이 때, 대기 중에 타임아웃이 발생하거나 시그널이 들어오면 대기를 종료하고, 이벤트가 있으면 다시 확인한다.
    
5. **반복**: 이벤트가 발생할 때까지 계속해서 위 과정을 반복하며, 중간에 타임아웃이 발생하거나 시그널이 들어오면 즉시 반환한다.
    

이 과정은 이벤트가 발생할 때까지 대기하는 비동기식 작업을 효율적으로 처리하기 위한 메커니즘으로 사용된다.

[[ep_events_available()]]
[[ep_send_events()]]
## ep_send_events()

```c title=ep_send_events()
static int ep_send_events(struct eventpoll *ep,
			  struct epoll_event __user *events, int maxevents)
{
	struct epitem *epi, *tmp;
	LIST_HEAD(txlist);
	poll_table pt;
	int res = 0;

	/*
	 * Always short-circuit for fatal signals to allow threads to make a
	 * timely exit without the chance of finding more events available and
	 * fetching repeatedly.
	 */
	if (fatal_signal_pending(current))
		return -EINTR;

	init_poll_funcptr(&pt, NULL);

	mutex_lock(&ep->mtx);
	ep_start_scan(ep, &txlist);

	/*
	 * We can loop without lock because we are passed a task private list.
	 * Items cannot vanish during the loop we are holding ep->mtx.
	 */
	list_for_each_entry_safe(epi, tmp, &txlist, rdllink) {
		struct wakeup_source *ws;
		__poll_t revents;

		if (res >= maxevents)
			break;

		/*
		 * Activate ep->ws before deactivating epi->ws to prevent
		 * triggering auto-suspend here (in case we reactive epi->ws
		 * below).
		 *
		 * This could be rearranged to delay the deactivation of epi->ws
		 * instead, but then epi->ws would temporarily be out of sync
		 * with ep_is_linked().
		 */
		ws = ep_wakeup_source(epi);
		if (ws) {
			if (ws->active)
				__pm_stay_awake(ep->ws);
			__pm_relax(ws);
		}

		list_del_init(&epi->rdllink);

		/*
		 * If the event mask intersect the caller-requested one,
		 * deliver the event to userspace. Again, we are holding ep->mtx,
		 * so no operations coming from userspace can change the item.
		 */
		revents = ep_item_poll(epi, &pt, 1);
		if (!revents)
			continue;

		events = epoll_put_uevent(revents, epi->event.data, events);
		if (!events) {
			list_add(&epi->rdllink, &txlist);
			ep_pm_stay_awake(epi);
			if (!res)
				res = -EFAULT;
			break;
		}
		res++;
		if (epi->event.events & EPOLLONESHOT)
			epi->event.events &= EP_PRIVATE_BITS;
		else if (!(epi->event.events & EPOLLET)) {
			/*
			 * If this file has been added with Level
			 * Trigger mode, we need to insert back inside
			 * the ready list, so that the next call to
			 * epoll_wait() will check again the events
			 * availability. At this point, no one can insert
			 * into ep->rdllist besides us. The epoll_ctl()
			 * callers are locked out by
			 * ep_send_events() holding "mtx" and the
			 * poll callback will queue them in ep->ovflist.
			 */
			list_add_tail(&epi->rdllink, &ep->rdllist);
			ep_pm_stay_awake(epi);
		}
	}
	ep_done_scan(ep, &txlist);
	mutex_unlock(&ep->mtx);

	return res;
}
```

`ep_send_events` 함수는 `epoll_wait`에서 준비된 이벤트들을 사용자 공간에 전달하는 역할을 한다. 아래와 같은 과정으로 동작한다.

1. **시그널 확인**: 치명적인 시그널이 있으면 바로 종료한다.
2. **폴링 준비**: 사용자 공간에 전달할 이벤트를 저장할 리스트(`txlist`)를 준비하고, 준비된 항목을 확인한다.
3. **이벤트 처리**: 각 이벤트 항목을 확인하여, 사용자가 요청한 이벤트가 발생했으면 사용자 공간으로 전달한다.
4. **레벨 트리거(EPOLLET) 및 원샷 처리**: 이벤트가 처리된 후, 원샷 모드에서는 이벤트를 삭제하고, 레벨 트리거 모드에서는 다시 대기열에 넣는다.

이 방식으로 `epoll_wait`이 효율적으로 이벤트를 처리할 수 있게 도와준다.

