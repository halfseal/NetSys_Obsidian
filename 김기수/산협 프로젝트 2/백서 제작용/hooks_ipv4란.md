> 우선 해당 변수는 `netns_nf` 라는 구조체의 변수로 존재한다. 타입은 `nf_hook_entries`이며, `NF_INET_NUMHOOKS`갯수만큼 pointer array로 선언되어 있다.
> 
> `netns_nf`구조체는 `net`구조체에 선언되어 있다. 다만 `#ifdef`로 `CONFIG_NETFILTER`가 `true`일 때만 선언된다.

>`nf_hookfn`이라는 함수 포인터는 `nf_hook_ops`안에 멤버 변수로 선언되어 이를 등록할 때, `nf_hook_ops` 통째로 함수 인자로 들어가게 된다.
>
>새로운 hookfn을 등록하는 과정을 찾는데 많은 어려움이 있었는데, `rcu_assign_pointer`를 통해 등록이 이루어지기 때문이다.
>따라서 `nf_hook_entry_head()`함수를 통해 찾은 entry_head를 바탕으로 여기에다가 RCU protected로 해당 `new_hooks`값을 저장하도록 하게 되어 있다.
>
>이 등록 함수는 `__nf_register_net_hook()`이다. 이 함수 안에서는 `nf_hook_entries_grow()`함수를 사용하고 있는데, 말 그대로 기존의 hook_entries를 가져와서 새로운 hook을 덧 붙여서 이를 반환하는 함수이다.
>
>어쨌든 새로운 hook을 가져오는 것이다.
>
>등록되는 지점들을 역으로 추적하여 hook 함수들을 아래에 모아 보았다.
>`nf_hook_run_bpf`
>나머지는 다른 함수로부터 인자로 받아왔다.
>`nf_table`이라는 방대한 구조체로부터 내려오는 것 같은데, 나중에 천천히 살펴보기로 하였다.

![[Pasted image 20240909150721.png]]

https://velog.io/@koo8624/Linux-A-Deep-dive-into-iptables-and-Netfilter
참고자료

>`nf_register_net_hooks()` : 단순히 여러 개인 경우 for문을 통해 각각 실행시킴
>`nft_netdev_register_hooks()` : 
>`nf_tables_register_hook()` :
>`nft_register_flowtable_net_hooks()` :
>`bpf_nf_link_attach()` : 
->`nf_register_net_hook()` -> `__nf_register_net_hook()`

>현재 처음으로 찾은 부분은 바로 PRE_ROUTING 부분이다. 이는 `ip_rcv`함수의 `NF_HOOK()`호출에서 보면, 2번 째 인자로 `NF_INET_PRE_ROUTING` enum과 함께 호출하고  있음을 알 수 있다.
>이와 관련되어, `nf_inet_hooks`타입의 enum은, `NF_INET_PRE_ROUTING`, `NF_INET_LOCAL_IN`, `NF_INET_FORWARD`, `NF_INET_LOCAL_OUT`, `NF_INET_POST_ROUTING` 등이 있으며, 각각 상황에 맞춰서 사용되게 된다.
>그 외에도 `NF_INET_NUMHOOKS`, `NF_INET_INGRESS` 등이 있다.
>이러한 훅에 따라 실행되는 체인이 다르게 되는데, 우선순위를 가지고 테이블들이 적용되게 된다.

>이해하기가 힘들어서 nf_hook_ops로 미리 선언된 변수들을 전부 찾아보기로 하였다.
>`ipv4_conntrack_ops[]`-> `ipv4_conntrack_in`, `ipv4_conntrack_local`, `nf_confirm`
>`ipv4_defrag_ops[]` -> `ipv4_conntrack_defrag`, `ipv4_conntrack_defrag`
>`nf_nat_ipv4_ops[]` -> `ipt_do_table`
>`ip_vs_ops4[]` -> IP virtual server와 관련 된 넷 필터. 고성능 리눅스 서버를 구축할 때 사용한다.
>기타 전부 `/net/netfilter`에 혹은 `/net/~~~/netfilter`에 implement 되어 있다.

>

