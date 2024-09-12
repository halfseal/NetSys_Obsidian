---
creator: 김명수
type: Hands-On
created: 2022-11-21
---
Pascal

[main.cu](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7f0fef26-8ff9-4869-a3c0-c8bac0d23f5d/main.cu)
![[Untitled(223).png]]


shared data / async / stream flag(non-blocking)
![[Untitled(224).png]]


1. Stream shared data & Memcopy sync & No stream flag
![[Untitled(225).png]]


2. Stream shared data & Memcopy Async & No stream flag
![[Untitled(226).png]]


   The results of computation are same && correct


3. Stream shared data & Memcopy sync & Stream flag non-blocking
![[Untitled(227).png]]

The results are wrong

![[Untitled(228).png]]


4. Stream shared data & Memcopy Async & stream flag non-blocking
![[Untitled(229).png]]
The results are correct


5. Stream private data & Memcopy sync & No stream flag
![[Untitled(230).png]]
The results are correct


6. Stream private data & Memcopy Async & No stream flag
![[Untitled(231).png]]

The results are correct


7. Stream private data & Memcopy sync & stream flag non-blocking
![[Untitled(232).png]]
 The results are wrong

8. Stream private data & Memcopy Async & stream flag non-blocking
![[Untitled(233).png]]

The results are correct


Avoid using non-blocking kernel execution without async memcopy

nsight 사용해보기