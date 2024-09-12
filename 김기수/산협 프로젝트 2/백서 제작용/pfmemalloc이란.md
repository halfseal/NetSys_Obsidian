https://docs.kernel.org/staging/static-keys.html
정적 키에 대한 리눅스 커널 공식 문서

>정적 키를 얘기 해보자면, 우선 이를 만들게 된 동기는 tracepoint의 overhead 때문이다.
주로 tracepoint는 conditional branch에 의해 작동하게 된다. 이러한 tracepoint들은 주로 global variable을 각각의 포인트들에서 확인하는데, 이러한 검사로 인한 오버헤드 자체는 작지만, 만약 메모리 캐시가 부족하여 압력을 받고 있을 때는 오버헤드가 커지게 된다. 또한 tracepoint가 꺼져있을 경우에 그 branch 코드들은 커널들에게 아무런 기능성을 제공하지 않으므로 이러한 영향들을 줄이기 위해 정적 키가 탄생하였다.

>우선 정적 키를 통해 `static_branch_(un)likely()` 함수를 if 문 안에 넣음으로써 이를 사용할 수 있다.
>지원하는 컴파일러는 gcc(v4.5)이상이다.
>위의 코드는 라인을 벗어나게 되는 jump 코드를 대체하여 single atomic 'no-op'코드를 그대로 삽입하게 된다.(x86 기준으로 5 byte)
>
>그리고, 만약 해당 키가 바뀌게 되면,(true->false or false->true) 해당 코드 영역이 jump 코드로 "패치"가 되는 것이다. 심지어 이는 메모리를 참조할 필요가 없으므로 메모리 캐시에 대한 오버헤드가 없다.
>
>즉, 해당 jump 코드 영역을 커널이 실시간으로 바이너리 코드를 변경하고 있는 것이다. 이것을 "패치"라고 일컫는다.
>
>이는 해당 코드가 실행되면서 일어나게 된다. 사용하는 방법은` DEFINE_STATIC_KEY_(TRUE/FALSE)(&KEY)`로 키를 만들게 되고, 이를 이용해서 conditional branch의 조건으로 사용하면 된다.
>만약 조건을 true 혹은 false로 바꾸고 싶다면, `static_branch_inc(&KEY)` / `static_branch_dec(&KEY)` 함수들을 사용하면 된다.

>즉 기존의 코드들은 바이너리가 변경되지는 않고 매번 branch check를 통해 jump의 여부를 결정하는 반면 위의 방법은 코드 바이너리 자체를 매번 바꾸게 된다. 그러면 이러한 branch 부분을 처리하는데 어떠한 추가적인 cost도 들지 않는다.
>단, 큰 단점이 존재하는데, 이러한 바이너리 코드를 변경하는데 엄청나게 cost가 높다. 따라서 많은 conditional branch에서 공통적으로 사용되며, 그 조건이 드물게 변경되는 경우에 사용하면 오버헤드를 크게 줄 일 수 있을 것이다.

아직까지 pfmemalloc을 잘 모르겠다.
```c
/**
* skb_pfmemalloc - Test if the skb was allocated from PFMEMALLOC reserves
* @skb: buffer
*/
```
slab / slub allocater를 보면 메모리 풀이 부족해도 할당이 가능하게 하는 메모리 할당이라고 하는데 자세하게 이해하지는 못하였다.

따라서 skb->pfmemalloc의 경우 해당 skb가 pfmemalloc을 통해 할당받은 메모리에 있는지 여부를 나타내고 있다.
