irqaction을 등록하는 내부함수. 주로 아키텍쳐의 일부인 특별한 인터럽트들을 할당하는데 쓰인다.

> ==handler는 앞서 지정한 interrupt handler, thread_fn은 NULL인 상태임. nested의 뜻은, 인터럽트를 처리하는 중에 또다른 인터럽트를 처리할 수 있는지 유무다. 즉, 계속하여 인터럽트가 겹쳐져서 처리가 될 수 있다는 뜻이다. 그러나 nested가 불가능한 경우 이를 거부할 것이다.==

  

쓰레드화된 인터럽트가 만약 핸들러가 없다면, irq_default_primary_handler나 nested threaded의 경우 irq_nested_primary_handler, irq_forced_secondary_handler를 irq_handler로 직접 할당해주게 된다.

  

thread_fn이 NULL값으로 전달되었기 때문에, 아래의 과정은 실행되지 않는다. 그러나 software interrupt handler의 핵심 처리과정으로 보여 정리 하였다.

→setup_irq_thread() : *data로 들어온 irqaction 에 해당하는 irq_handler 함수를 처리하기 위한 커널 쓰레드를 생성하는 함수이다. **** 해당하는 irq_handler를 실행하는게 아닌 irq_thread()라는 함수를 실행함, 프로세스 이름은 “irq/{irq number}-{irq action의 이름}”으로 지정됨.

⇒irq_thread() ==(on new kernel thread)== : Interrupt handler thread라고 주석 되어 있음. handler_fn 함수 포인터를 선언하여 기다리고 있다가 만약 하드웨어 인터럽트가 끝났다면 irq_thread_fn을 실행함. 이때, irq_thread_fn() 함수는 thread_fn을 실행하게 됨. 여기서부터 software interrupt 처리 과정이 됨.

  

  

→mutex_lock() : __free_irq()와의 동시실행을 막기 위한 함수이다.

→chip_bus_lock() : irq_bus_lock()의 잘못된 사용을 막기 위한 함수이다.

만약 desc→action이 없다면 irq_request_resources()로 자원을 요청한다.

old 포인터를 통해 만약 해당하는 irq에 actioin이 있다면 이를 공유 할 수 있는지 확인하고, 위배하였다면 관련된 처리를 한다.

  

![[Untitled 12.png|Untitled 12.png]]

새로운 irq action을 해당하는 irq desc에 넣는 모습.

  

---

![[Untitled 1 5.png|Untitled 1 5.png]]

이는 thread_flags로 irqaction에 들어가는 unsigned long 변수이다.

  

→__irq_set_trigger()

→irq_activate() → irq_domain_activate_irq() ==(kernel/irq/chip.c) / (kernel/irq/internals.h) 혹은 (kernel/irq/irqdomain.c)에 있음.==

→irqd_set_activated() (include/linux/irq.h)

IRQD_ACTIVATED를 irq_data 구조체에 비트 세팅을 함.

→__irq_domain_activate_irq() : 재귀적으로 irq_data의 부모 데이터가 있다면 이를 활성화 시킴.

→irqd_set()

→irq_startup()