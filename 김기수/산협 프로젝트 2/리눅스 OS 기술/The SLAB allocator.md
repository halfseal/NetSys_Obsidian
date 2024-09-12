https://jiravvit.tistory.com/entry/linux-kernel-4-슬랩할당자
https://lwn.net/Articles/229984/
#### 개요
----
자주 사용하는 데이터 컨테이너를 미리 할당해두고, 해당 타입의 할당을 요청 받았을 때 미리 할당해둔 영역에서 반환하여 할당시에 발생하는 오버헤드를 최소화하는 기술이다.
최소 할당 단위는 slab object이며, 이 것들이 모여 하나 이상의 연속된 페이지를 하나의 slab으로 구성하게 된다. 이러한 slab cache를 관리하기 위한 구조체 kmem_cache_t에 full list, partial list, empty list로 구분해서 관리하였는데, 이러한 관리 방법에 오버헤드가 많아 slub이 등장하게 되었다.

#### slub allocator
---
 우선 본 문서는 2007년에 작성된 문서이다. 주요 골자는 slab allocator가 scale up이 어려운 부분이 있다는 것이다. slab allocator는 object의 큐들을 유지하게 되는데, 이 큐들은 빠르게 할당을 할 수 있지만 꾀 복잡함을 야기하고 있다. 심지어 저장소 오버헤드가 시스템이 커질수록 같이 증가하는 경향을 보이고 있다.

각각의 slab(한 개 이상의 objects가 할당되어 있는 연속적인 페이지들의 모임)은 그 시작점에 메타데이터 덩어리를 담고 있다. 이것은 object의 정렬을 어렵게 한다. 캐시를 정리하는 코드는 또 다른 복잡성을 추가하게 된다.

slub allocator는 이러한 한계점에 대한 돌파구로, slab code의 drop-in replacement이다. slab allocator interface를 유지하면서 오버헤드와 대부분의 큐들을 드랍함으로써 더 나은 성능을 보장한다.

