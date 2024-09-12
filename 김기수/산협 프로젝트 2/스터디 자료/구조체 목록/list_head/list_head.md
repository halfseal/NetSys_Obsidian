---
locate: include/linux/type.h
---
### Field

---

```C
struct list_head {
	struct list_head *next, *prev
}
```

### Description

---

리눅스 커널에서 doubly linked list를 구현하는데 사용 됨.

기존 링크드 리스트의 한계점을 극복하고자 만들어짐.

linked list를 만들고자 하는 구조체에 넣어서 사용하고, 중복하여 다른 linked list에도 연결 할 수 있음.

![[Untitled 10.png|Untitled 10.png]]

따로 list_head를 통해 접근 가능. 해당하는 구조체에 접근하기 위해서 container_of라는 함수를 이용함. 해당하는 구조체에이 list_head라는 member의 offset과 그 주소를 역산하면 간단하게 해당하는 구조체의 주소를 얻어 접근할 수 있음.

---

```C
container_of(ptr, struct target_struct, list) ==>
(struct target_struct *)((char *)ptr) - ((size_t) &((struct target_struct *)0)->list)
```