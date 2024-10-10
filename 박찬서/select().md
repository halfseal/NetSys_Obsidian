---
Parameter:
  - int
  - fd_set
  - ___kernel_old_timeval
Return: int
Location: /fs/select.c
---
## select()

```c
SYSCALL_DEFINE5(select, int, n, fd_set __user *, inp, fd_set __user *, outp,

fd_set __user *, exp, struct __kernel_old_timeval __user *, tvp)

{

return kern_select(n, inp, outp, exp, tvp);

}
```

`select` 시스템 호출은 프로세스가 여러 파일 디스크립터(FD)를 동시에 감시하여 읽기, 쓰기, 또는 예외 발생 여부를 확인할 수 있게 한다. 위 코드에서 `SYSCALL_DEFINE5`를 통해 5개의 매개변수를 사용하는 `select` 시스템 호출이 정의된다.

- `n`: 감시할 파일 디스크립터의 최대 개수
- `inp`: 읽기용 FD 세트
- `outp`: 쓰기용 FD 세트
- `exp`: 예외 처리용 FD 세트
- `tvp`: 타임아웃 구조체의 포인터

내부적으로는 `kern_select` 함수를 호출하여 실제 처리를 수행한다.

## kern_select()

```c title=kern_select()
static int kern_select(int n, fd_set __user *inp, fd_set __user *outp,

fd_set __user *exp, struct __kernel_old_timeval __user *tvp)

{

struct timespec64 end_time, *to = NULL;

struct __kernel_old_timeval tv;

int ret;

  

if (tvp) {

if (copy_from_user(&tv, tvp, sizeof(tv)))

return -EFAULT;

  

to = &end_time;

if (poll_select_set_timeout(to,

tv.tv_sec + (tv.tv_usec / USEC_PER_SEC),

(tv.tv_usec % USEC_PER_SEC) * NSEC_PER_USEC))

return -EINVAL;

}

  

ret = core_sys_select(n, inp, outp, exp, to);

return poll_select_finish(&end_time, tvp, PT_TIMEVAL, ret);

}
```

`kern_select` 함수는 `select` 시스템 호출의 내부 함수로, 파일 디스크립터를 기반으로 입력, 출력, 또는 예외를 감시한다.

1. `tvp`(타임아웃 값)가 존재하면 `copy_from_user`를 사용해 유저 공간에서 커널로 `tv` 값을 복사하고, 타임아웃을 설정한다.
2. `poll_select_set_timeout` 함수로 타임아웃이 유효한지 확인하고, 시간 변환 후 `to` 변수에 저장한다.
3. `core_sys_select` 함수는 실제로 파일 디스크립터의 상태를 확인하는 핵심 기능을 담당한다.
4. 마지막으로 `poll_select_finish`는 타임아웃 계산 후 결과를 반환한다.


## core_sys_select()

```c title=core_sys_select()
/*

* We can actually return ERESTARTSYS instead of EINTR, but I'd

* like to be certain this leads to no problems. So I return

* EINTR just for safety.

*

* Update: ERESTARTSYS breaks at least the xview clock binary, so

* I'm trying ERESTARTNOHAND which restart only when you want to.

*/

int core_sys_select(int n, fd_set __user *inp, fd_set __user *outp,

fd_set __user *exp, struct timespec64 *end_time)

{

fd_set_bits fds;

void *bits;

int ret, max_fds;

size_t size, alloc_size;

struct fdtable *fdt;

/* Allocate small arguments on the stack to save memory and be faster */

long stack_fds[SELECT_STACK_ALLOC/sizeof(long)];

  

ret = -EINVAL;

if (n < 0)

goto out_nofds;

  

/* max_fds can increase, so grab it once to avoid race */

rcu_read_lock();

fdt = files_fdtable(current->files);

max_fds = fdt->max_fds;

rcu_read_unlock();

if (n > max_fds)

n = max_fds;

  

/*

* We need 6 bitmaps (in/out/ex for both incoming and outgoing),

* since we used fdset we need to allocate memory in units of

* long-words.

*/

size = FDS_BYTES(n);

bits = stack_fds;

if (size > sizeof(stack_fds) / 6) {

/* Not enough space in on-stack array; must use kmalloc */

ret = -ENOMEM;

if (size > (SIZE_MAX / 6))

goto out_nofds;

  

alloc_size = 6 * size;

bits = kvmalloc(alloc_size, GFP_KERNEL);

if (!bits)

goto out_nofds;

}

fds.in = bits;

fds.out = bits + size;

fds.ex = bits + 2*size;

fds.res_in = bits + 3*size;

fds.res_out = bits + 4*size;

fds.res_ex = bits + 5*size;

  

if ((ret = get_fd_set(n, inp, fds.in)) ||

(ret = get_fd_set(n, outp, fds.out)) ||

(ret = get_fd_set(n, exp, fds.ex)))

goto out;

zero_fd_set(n, fds.res_in);

zero_fd_set(n, fds.res_out);

zero_fd_set(n, fds.res_ex);

  

ret = do_select(n, &fds, end_time);

  

if (ret < 0)

goto out;

if (!ret) {

ret = -ERESTARTNOHAND;

if (signal_pending(current))

goto out;

ret = 0;

}

  

if (set_fd_set(n, inp, fds.res_in) ||

set_fd_set(n, outp, fds.res_out) ||

set_fd_set(n, exp, fds.res_ex))

ret = -EFAULT;

  

out:

if (bits != stack_fds)

kvfree(bits);

out_nofds:

return ret;

}
```

 `core_sys_select` 함수는  `select` 시스템 호출의 핵심 기능을 담당한다. 이 함수는 주어진 파일 디스크립터 집합(inp, outp, exp)에서 입력, 출력, 예외 발생 여부를 감시한다.

1. 먼저, 파일 디스크립터의 개수를 확인한다.
2. fd_set 구조를 통해 메모리 할당을 하고, `get_fd_set`을 사용하여 유저 공간에서 전달된 fd_set을 커널 공간으로 복사한다.
3. 그런 다음 `do_select` 함수가 호출되어 파일 디스크립터에서 이벤트가 발생하는지 확인한다.
4. 작업이 완료되면, `set_fd_set`을 통해 결과를 유저 공간으로 다시 전달한다.

이 과정에서 `ERESTARTNOHAND`는 시그널이 처리되지 않은 경우에 재시작을 방지하는 역할을 한다.\
## do_select()

```c title=do_select()
static noinline_for_stack int do_select(int n, fd_set_bits *fds, struct timespec64 *end_time)

{

ktime_t expire, *to = NULL;

struct poll_wqueues table;

poll_table *wait;

int retval, i, timed_out = 0;

u64 slack = 0;

__poll_t busy_flag = net_busy_loop_on() ? POLL_BUSY_LOOP : 0;

unsigned long busy_start = 0;

  

rcu_read_lock();

retval = max_select_fd(n, fds);

rcu_read_unlock();

  

if (retval < 0)

return retval;

n = retval;

  

poll_initwait(&table);

wait = &table.pt;

if (end_time && !end_time->tv_sec && !end_time->tv_nsec) {

wait->_qproc = NULL;

timed_out = 1;

}

  

if (end_time && !timed_out)

slack = select_estimate_accuracy(end_time);

  

retval = 0;

for (;;) {

unsigned long *rinp, *routp, *rexp, *inp, *outp, *exp;

bool can_busy_loop = false;

  

inp = fds->in; outp = fds->out; exp = fds->ex;

rinp = fds->res_in; routp = fds->res_out; rexp = fds->res_ex;

  

for (i = 0; i < n; ++rinp, ++routp, ++rexp) {

unsigned long in, out, ex, all_bits, bit = 1, j;

unsigned long res_in = 0, res_out = 0, res_ex = 0;

__poll_t mask;

  

in = *inp++; out = *outp++; ex = *exp++;

all_bits = in | out | ex;

if (all_bits == 0) {

i += BITS_PER_LONG;

continue;

}

  

for (j = 0; j < BITS_PER_LONG; ++j, ++i, bit <<= 1) {

struct fd f;

if (i >= n)

break;

if (!(bit & all_bits))

continue;

mask = EPOLLNVAL;

f = fdget(i);

if (f.file) {

wait_key_set(wait, in, out, bit,

busy_flag);

mask = vfs_poll(f.file, wait);

  

fdput(f);

}

if ((mask & POLLIN_SET) && (in & bit)) {

res_in |= bit;

retval++;

wait->_qproc = NULL;

}

if ((mask & POLLOUT_SET) && (out & bit)) {

res_out |= bit;

retval++;

wait->_qproc = NULL;

}

if ((mask & POLLEX_SET) && (ex & bit)) {

res_ex |= bit;

retval++;

wait->_qproc = NULL;

}

/* got something, stop busy polling */

if (retval) {

can_busy_loop = false;

busy_flag = 0;

  

/*

* only remember a returned

* POLL_BUSY_LOOP if we asked for it

*/

} else if (busy_flag & mask)

can_busy_loop = true;

  

}

if (res_in)

*rinp = res_in;

if (res_out)

*routp = res_out;

if (res_ex)

*rexp = res_ex;

cond_resched();

}

wait->_qproc = NULL;

if (retval || timed_out || signal_pending(current))

break;

if (table.error) {

retval = table.error;

break;

}

  

/* only if found POLL_BUSY_LOOP sockets && not out of time */

if (can_busy_loop && !need_resched()) {

if (!busy_start) {

busy_start = busy_loop_current_time();

continue;

}

if (!busy_loop_timeout(busy_start))

continue;

}

busy_flag = 0;

  

/*

* If this is the first loop and we have a timeout

* given, then we convert to ktime_t and set the to

* pointer to the expiry value.

*/

if (end_time && !to) {

expire = timespec64_to_ktime(*end_time);

to = &expire;

}

  

if (!poll_schedule_timeout(&table, TASK_INTERRUPTIBLE,

to, slack))

timed_out = 1;

}

  

poll_freewait(&table);

  

return retval;

}
```

`do_select`는 `select` 시스템 호출에서 다루는 파일 디스크립터들을 감시하는 핵심 역할을 한다. 주어진 파일 디스크립터들을 검사하여, 이벤트(읽기, 쓰기, 예외 등)가 발생했는지 확인하고, 발생한 이벤트에 따라 결과를 반환한다.

1. **타임아웃 설정:** 타임아웃을 처리하기 위해 `end_time`을 확인하고, 남은 시간을 계산하여 대기 시간이 설정된다.
    
2. **파일 디스크립터 검사:** `max_select_fd`로 파일 디스크립터를 가져와 `vfs_poll`로 각 디스크립터의 상태를 확인한다.
    
3. **이벤트 감지:** `POLLIN`, `POLLOUT`, `POLLEX` 플래그를 통해 파일 디스크립터에서 발생한 이벤트(읽기 가능, 쓰기 가능, 예외 발생 등)를 검사하여 결과를 기록한다. 이벤트가 발생하면 `retval` 값을 증가시킨다.
    
4. **비동기 대기:** 이벤트가 감지되지 않으면 `poll_schedule_timeout`을 호출하여 커널에서 타임아웃 대기를 설정한다.
    
5. **busy polling:** 네트워크 소켓과 같은 경우에는 `POLL_BUSY_LOOP`을 사용하여 CPU를 지속적으로 사용해 빠르게 이벤트를 감지할 수 있는 바쁜 대기 (busy polling)를 허용한다. 이 방식은 CPU 자원을 사용하여 대기 시간을 줄일 수 있지만, 대기 시간이 지나거나 이벤트가 발생하지 않으면 타임아웃으로 전환된다.
    
6. **종료:** 모든 파일 디스크립터가 처리되었거나, 타임아웃이 발생하거나, 시그널이 감지되면 루프가 종료되고 대기 테이블을 정리한 후 결과를 반환한다.
    

이 함수는 `select` 시스템 호출이 완료될 때까지 주어진 파일 디스크립터들에서 이벤트가 발생하는지 감시하고 그 결과를 사용자에게 반환하는 역할을 한다.