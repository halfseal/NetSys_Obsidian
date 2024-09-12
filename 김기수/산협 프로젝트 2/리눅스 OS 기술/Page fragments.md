---
sticker: lucide//clipboard-check
---
#### 개요
---
`page fragment`란, 임의의 길이의 임의의 오프셋의 0 혹은 더 높은 주소에 존재하는 복합 페이지 메모리 공간을 의미한다. page의 reference counter에서, page 속에 수 많은 fragment들은 각자 참조가 세어지고 있다. *(counted)*

#### 일단 문단
---
`page_frag`함수들은 page fragments를 위한 간단한 할당/해제 프레임워크를 제공하고 있다. 보통은 네트워크 스택과 네트워크 device driver에서 사용되는데, `sk_buff->head`, `skb_shared_info`의 멤버 변수인 `frags`를 지원하기 위한 메모리 영역으로 제공된다.

#### 이단 문단
---
일단 `page fragment API`를 사용하기 위해서는 이를 뒷 받쳐주는 page fragment cache가 필요하다. 이것은 fragment 할당을 위한 중심점으로 제공되고, 트랙은 cached page를 사용하게 하기 위해 다중 호출을 허용한다. 이러한 다중 호출을 허용함으로써 얻는 이점은 할당될 때에 더 비용이 들 수 있었을 것을 회피할 수 있다는 점이다. *다중 호출이 아닌 경우 해당 lock을 획득하기 위한 경쟁에 의한 성능 손실을 의미하는 것* 그러나 이 caching의 특징때문에, cache에 대한 어떠한 호출이던지 CPU당 제한 혹은 CPU당 제한과 fragment 할당 시에 꺼지도록 하는 interrupt 강제에 의해 보호받아야 한다.

#### 삼단 문단
---
네트워크 스택은 fragment allocation을 다루기 위해 CPU 당 2개의 구분된 cache를 가지고 있다. `netdev_alloc_cache`는  `netdev_alloc_frag` 와 `__netdev_alloc_skb`호출 중에 caller에 의해 사용된다. `napi_alloc_cache`는 `__napi_alloc_frag`와 `napi_alloc_skb` 호출 중에 caller에 의해 사용된다.
이 두 호출 사이에 가장 주요한 차이는 그 들이 호출되는 context가 어디냐 일 것이다. `netdev` 접두사 함수들은 그 함수가 interrupt를 끄기 때문에, 어떠한 context에서도 사용할 수 있다. 그러나, `napi` 접두사 함수들은 오직 softirq context 상에서만 사용 가능하다.

#### 사단 문단
---
많은 네트워크 device driver들은 page fragment 할당을 위해 비슷한 방법론을 사용하고 있다. 그러나 page fragments들은 ring이나 descriptor 레벨에서 cached 되어 있다. 이러한 상황에서도 작동하게 하기 위해서는 page cache를 분해하는 보편적인 방법이 제공되어야 할 것이다. 이러한 이유로 인해 `__page_frag_cache_drain` 이 작성되었다. 이렇게 하는 것의 장점은 할당마다 get_page를 호출하는 것을 피하기 위해 페이지에 추가된 여러 참조를 정리할 수 있다는 것이다.