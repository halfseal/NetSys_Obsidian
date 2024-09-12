---
locate: include/linux/interrupt.h
---
### Field

---

```C
	irq_handler_t		handler;
	void			*dev_id;
	void __percpu		*percpu_dev_id;
	struct irqaction	*next;
	irq_handler_t		thread_fn;
	struct task_struct	*thread;
	struct irqaction	*secondary;
	unsigned int		irq;
	unsigned int		flags;
	unsigned long		thread_flags;
	unsigned long		thread_mask;
	const char		*name;
	struct proc_dir_entry	*dir;
```

  

### Description

---

이 구조체는 각각의 interrupt에 대하여 action에 대한 descriptor이다.

handler는 실제 interrupt handler function이고,

thread_fn은 interrupt handler function이긴 하지만, threaded interrupt로 넘어갔을 때 쓰이는 함수이다.

즉, handler는 hardware interrupt handler 이고, thread_fn은 soft interrupt handler 이다.

next 포인터는 공유된 인터럽트를 위해 다음 irq action을 가르키고 있다.