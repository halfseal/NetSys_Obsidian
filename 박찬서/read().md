---
Parameter:
  - fd
  - buf
  - count
Return: ssize_t
Location: /fs/read_write.c
---
## read()

```c
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)

{

return ksys_read(fd, buf, count);

}
```

리눅스 커널에서 `read` 시스템 호출을 정의하는 매크로 파트이다. `SYSCALL_DEFINE3`는 세 가지 인수를 받는 `read` 시스템 호출을 설정한다. 즉, SYSCALL_DEFINE`n`에서 `n`의 의미는 Parameter의 개수를 의미한다.

- `fd`: 파일 디스크립터
- `buf`: 데이터를 읽어올 사용자 메모리 버퍼
- `count`: 읽고자 하는 데이터의 크기

이 시스템 호출이 실행되면 내부적으로 `ksys_read()` 함수가 호출되어 실제 파일로부터 데이터를 읽는 작업이 수행된다. `ksys_read()`는 파일 디스크립터, 버퍼, 크기를 받아 데이터를 읽고 결과를 반환한다.

## ksys_read()

```c title=ksys_read()
ssize_t ksys_read(unsigned int fd, char __user *buf, size_t count)

{

struct fd f = fdget_pos(fd);

ssize_t ret = -EBADF;

  

if (f.file) {

loff_t pos, *ppos = file_ppos(f.file);

if (ppos) {

pos = *ppos;

ppos = &pos;

}

ret = vfs_read(f.file, buf, count, ppos);

if (ret >= 0 && ppos)

f.file->f_pos = pos;

fdput_pos(f);

}

return ret;

}
```

`ksys_read`는 리눅스 커널에서 `read()` 시스템 콜의 메인 함수를 처리하는 함수이다. 이 함수의 동작을 자세히 설명하면 다음과 같다.

1. **fdget_pos(fd)**: 파일 디스크립터 `fd`에 대응하는 파일 객체를 가져온다. 이때, `fd`가 유효하지 않으면 `-EBADF`를 반환한다.
    
2. **pos와 ppos 설정**: 파일 위치(offset)를 관리한다. 파일 객체가 포지션을 관리하면 이를 `pos`와 `ppos`에 설정하여 처리한다.
    
3. **vfs_read() 호출**: `vfs_read()` 함수는 파일 시스템 레벨에서 데이터를 읽는다. 이 함수는 파일 객체와 읽을 버퍼, 크기, 오프셋을 인자로 받는다.
    
4. **파일 포인터 업데이트**: 읽기 작업이 성공적으로 끝나면 파일 객체의 위치를 `f_pos`로 업데이트한다.
    
5. **fdput_pos(f)**: 파일 디스크립터 객체의 참조를 해제한다.
    
6. **결과 반환**: 읽은 바이트 수나 에러 코드를 반환한다.
## vfs_read()

```c title=vfs_read()
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)

{

ssize_t ret;

  

if (!(file->f_mode & FMODE_READ))

return -EBADF;

if (!(file->f_mode & FMODE_CAN_READ))

return -EINVAL;

if (unlikely(!access_ok(buf, count)))

return -EFAULT;

  

ret = rw_verify_area(READ, file, pos, count);

if (ret)

return ret;

if (count > MAX_RW_COUNT)

count = MAX_RW_COUNT;

  

if (file->f_op->read)

ret = file->f_op->read(file, buf, count, pos);

else if (file->f_op->read_iter)

ret = new_sync_read(file, buf, count, pos);

else

ret = -EINVAL;

if (ret > 0) {

fsnotify_access(file);

add_rchar(current, ret);

}

inc_syscr(current);

return ret;

}

```

`vfs_read`는 파일 시스템 수준에서 데이터를 읽는 함수로, 주어진 파일에 대해 읽기 작업을 처리한다. 

1. **권한 검사**: 파일 모드에서 읽기 권한(`FMODE_READ`) 및 읽기 가능 상태(`FMODE_CAN_READ`)를 확인한다. `buf`가 사용자 메모리에 안전하게 접근할 수 있는지 확인한다 (`access_ok`).
    
2. **영역 검증**: `rw_verify_area()`를 호출해, 읽기 작업에 필요한 영역이 유효한지 확인한다. 오류가 있으면 반환한다.
   
3. **읽기 작업 수행**: 파일 객체의 `f_op`에 정의된 `read()` 함수가 있으면 이를 호출하여 데이터를 읽는다. 없다면 `read_iter()`가 있는 경우 `new_sync_read()`를 호출하여 데이터를 처리한다. 둘 다 없으면 `-EINVAL` 오류를 반환한다.
   
4. **추가 처리**: 읽기가 성공적으로 완료되면 파일 접근에 대한 알림(`fsnotify_access`)을 생성하고, 읽은 데이터 크기를 프로세스의 계정(`rchar`)에 기록한다.
   
5. **시스템 호출 횟수 증가**: 읽기 시스템 호출 횟수를 증가시킨다(`inc_syscr()`).
   
6. **결과 반환**: 읽은 바이트 수를 반환하거나 에러 코드가 발생했으면 그 에러 코드를 반환한다.
## new_sync_read()

```c title=new_sync_read()
static ssize_t new_sync_read(struct file *filp, char __user *buf, size_t len, loff_t *ppos)

{

struct kiocb kiocb;

struct iov_iter iter;

ssize_t ret;

  

init_sync_kiocb(&kiocb, filp);

kiocb.ki_pos = (ppos ? *ppos : 0);

iov_iter_ubuf(&iter, ITER_DEST, buf, len);

  

ret = call_read_iter(filp, &kiocb, &iter);

BUG_ON(ret == -EIOCBQUEUED);

if (ppos)

*ppos = kiocb.ki_pos;

return ret;

}
```

`new_sync_read` 함수는 파일에서 데이터를 사용자 제공 버퍼로 동기적으로 읽는 작업을 수행하는 함수이다.

1. **I/O 제어 블록 초기화**: `init_sync_kiocb`는 파일과 위치 정보를 담은 `kiocb` 구조체를 초기화한다.
2. **읽기 위치 설정**: 읽기 위치는 `kiocb.ki_pos`에 설정된다.
3. **반복자 설정**: `iov_iter_ubuf`로 사용자의 버퍼와 길이를 지정하는 반복자를 설정한다.
4. **읽기 작업 수행**: `call_read_iter`로 실제 읽기 작업을 처리한다.
5. **위치 업데이트**: 읽은 후 파일 포인터 위치가 업데이트된다.
6. **결과 반환**: 읽은 바이트 수 또는 오류 코드를 반환한다.
## call_read_iter()
/include/linux/fs.h

```c title=call_read_iter()
static inline ssize_t call_read_iter(struct file *file, struct kiocb *kio,

struct iov_iter *iter)

{

return file->f_op->read_iter(kio, iter);

}
```

`call_read_iter` 함수는 파일의 `read_iter` 함수 포인터를 호출하는 역할을 한다. 이 함수는 파일의 읽기 작업을 처리하는 데 사용되며, 매개변수로 전달된 `kiocb`와 `iov_iter`를 활용하여 비동기 또는 동기적 방식으로 데이터를 읽는다.

- `file->f_op->read_iter`: 파일 연산 구조체에서 읽기 작업을 담당하는 함수 포인터.
- `kio`: I/O 제어 블록을 나타낸다.
- `iter`: I/O 반복자 구조체로, 사용자 버퍼와 데이터를 처리하는 데 사용된다.

이 함수는 `read_iter`를 호출하고, 결과로 읽은 바이트 수 또는 오류 코드를 반환한다.