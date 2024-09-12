---
creator: 최민규
type: Glossary
created: 2022-05-03
---
<<[[NUMA cntd.|next]]>>

SMP (↔ AMP)

(=Symmetric Multiprocessing)

→ 다수의 CPU가 동일한 버스 및 메인메모리 공유
![[Untitled(22).png]]


## Non-Uniform Memory Access

> 메모리에 접근하는 시간이 CPU와 메모리의 상대적인 위치에 따라 달라지는 컴퓨터 메모리 설계 방법

- 메모리 버스 동시 접근으로 인한 대기 문제 발생

local memory로 갖고 있음. “CPU 전용 메모리”

실제 물리적인 거리 영향 (15cm 떨어진 메모리 → 1 싸이클 추가 소모)

중간 단계의 공유메모리(L3) 사용을 통해 주버스 사용 줄이는 것이 목표!

구글에서는 자체적으로 tcmalloc이라는 고성능 메모리 관리자를 만들어 배포

멀티쓰레드 최적화 힙 메모리 할당기 TCMalloc

Thread Caching

쓰레드 마다 캐시 둠.

- small (~32K)
    - 사이즈별 class 구분 (170 allocatable size-classes)
    - 각 클래스마다 free object list 만들어두고 반환 (singly linked list)
    - 부족하면 central free list에서 가져옴
- large (4K 단위)

[TCMalloc : Thread-Caching Malloc](http://goog-perftools.sourceforge.net/doc/tcmalloc.html)

tcmalloc vs ptmalloc

numa node 문제

command queue

malloc은 어떻길래?

TCmalloc과 numa의 연관성?

NUMA 코드레벨

[[Linux Kernel 5] NUMA (Non-Uniform Memory Access)](https://pr0gr4m.tistory.com/entry/Linux-Kernel-5-NUMA-Non-Uniform-Memory-Access)

NUMA 구조의 문제 해결

[Bandwidth-Aware Page Placement in NUMA](https://ieeexplore.ieee.org/abstract/document/9139869)