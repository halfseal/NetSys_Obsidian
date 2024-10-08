---
Parameter:
  - eventpoll
  - epoll_event
  - int
  - timespec64
Return: int
Location: /fs/eventpoll.c
---

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
