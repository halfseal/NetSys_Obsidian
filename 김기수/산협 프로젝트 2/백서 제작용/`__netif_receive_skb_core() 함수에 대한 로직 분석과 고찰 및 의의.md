
#### 0. 함수 부분 별 분석
---
```c
  
	orig_dev = skb->dev;
	  
	skb_reset_network_header(skb);
	if (!skb_transport_header_was_set(skb))
		skb_reset_transport_header(skb);
	skb_reset_mac_len(skb);
	  
	pt_prev = NULL;

another_round:
	skb->skb_iif = skb->dev->ifindex;
	  
	__this_cpu_inc(softnet_data.processed);
	  
	if (static_branch_unlikely(&generic_xdp_needed_key)) {
		int ret2;
	  
		migrate_disable();
		ret2 = do_xdp_generic(rcu_dereference(skb->dev->xdp_prog),
						&skb);
		migrate_enable();
		  
		if (ret2 != XDP_PASS) {
			ret = NET_RX_DROP;
			goto out;
		}
	}

```

> 본 함수가 실행되기 전에 필요한 작업들을 해주는 부분이다.
> 
> `skb->skb_iif`는 `skb`의 `dev` 의 interface index를 가져온 것이다.
> 
 특히 `skb_reset_~` 네이밍의 함수들은 포인터들을 정렬시키는 역할을 한다. 이는 `skb->data - skb->head`의 로직을 통해 이루어진다.
>
> 그다음에 만약 xdp native와 독립적으로, xdp generic을 지원한다면, 여기서 XDP generic을 한번 더 하게 된다.

```c  
	if (eth_type_vlan(skb->protocol)) {
		skb = skb_vlan_untag(skb);
		if (unlikely(!skb))
			goto out;
	}
	  
	if (skb_skip_tc_classify(skb))
		goto skip_classify;
	  
	if (pfmemalloc)
		goto skip_taps;
	  
	list_for_each_entry_rcu(ptype, &net_hotdata.ptype_all, list) {
		if (pt_prev)
			ret = deliver_skb(skb, pt_prev, orig_dev);
		pt_prev = ptype;
	}
	  
	list_for_each_entry_rcu(ptype, &skb->dev->ptype_all, list) {
		if (pt_prev)
			ret = deliver_skb(skb, pt_prev, orig_dev);
		pt_prev = ptype;
	}
```

>그다음으로 vlan과 tc_classify skip 여부를 확인하고, 주어진 ptype list에서 맞는 ptype으로 deliver_skb가 실행되게 된다.
>
>함수 분석하면서 헷갈렸던 건데, 저기 for_each문에서 보면, `list_for_each_entry_rcu(ptype, &net_hotdata.ptype_all, list)`로 되어 있다.
>이는 `net_hotdata.ptype_all` 이라는 변수에 있는 list head에서 목록을 찾아다니는데, ptype으로 iteration에서 변수를 받아오게 되고, 해당 구조체에서 이 리스트에 접근하는 변수가 `list`인 것이다.
>[[주요 개념 구조도]] 
>
>여기서는 두 개의 for_each 문을 돌리게 되는데, 첫 번째는 `net_hotdata`에서 가져오게 되고, 두번째는 `skb->dev`에서 가져오게 된다. 이때 만약 `ptype`이 등록 될 때, `dev`가 `NULL`값이 아니라면 해당 `dev`에 넣게 되고, 아니라면 `net_hotdata`에 넣게 된다.
>
>그리고 패킷타입이 ETH_P_ALL 인 경우에만 해당한다. 이말 인 즉슨 모든 패킷에 대하여 처리를 하는 과정이라는 뜻이므로, `packet_type` 구조체를 등록하여 프로토콜에 상관없이 모든 패킷들에 대하여 다루게 할 수 있는 handler를 설정해 줄 수 있다는 뜻이다. 그러나 위의 로직에서는 해당 `net_device`에 대한 마지막 ptype이 처리되지 않는다는 점이 걸린다. 이후 코드에서 확인해 봐야 할 것이다.

```c
skip_taps:
#ifdef CONFIG_NET_INGRESS
	if (static_branch_unlikely(&ingress_needed_key)) {
		bool another = false;
		  
		nf_skip_egress(skb, true);
		skb = sch_handle_ingress(skb, &pt_prev, &ret, orig_dev,
						&another);
		if (another)
			goto another_round;
		if (!skb)
			goto out;
		  
		
		nf_skip_egress(skb, false);
		if (nf_ingress(skb, &pt_prev, &ret, orig_dev) < 0)
			goto out;
	}
#endif
	skb_reset_redirect(skb);
```

>여기서 집중적으로 본 부분은 바로 skip_taps: 라는 네이밍이였다. 여기서 말하는 tap이란 네트워크 패킷을 가로채거나 복사하여 다른 곳으로 전달하는 것을 뜻하며, 이 코드 바로 위에 모든 패킷에 대한 핸들러인 함수를 실행하는 코드를 건너뛰게 되는 것 이기에 의미가 상통하였다.

https://velog.io/@gadian88/Ingress-와-egress-차이
>>Ingress는 외부에서 내부로 유입되는 트래픽, egress는 내부에서 외부로 나가는 트래픽을 의미한다.
>여기에서는 traffic control에 관한 부분이기에 깊이있게 탐색하지 않으려고 한다.
>ingress / egress 경로가 무엇인지 정확하게 모르겠어서 우선 넘어가기로 하였다.
>내부 함수들을 보니 traffic control과 관련한 함수들을 호출하고 있었다. 따라서 바로 아래의 라벨 이름이 `skip_classify`로 추정된다.
>
>그렇게 `skip_classify:`라벨을 지나게 된다.

```c
skip_classify:
	if (pfmemalloc && !skb_pfmemalloc_protocol(skb))
		goto drop;
	  
	if (skb_vlan_tag_present(skb)) {
		if (pt_prev) {
			ret = deliver_skb(skb, pt_prev, orig_dev);
			pt_prev = NULL;
		}
		if (vlan_do_receive(&skb))
			goto another_round;
		else if (unlikely(!skb))
			goto out;
	}
  
	rx_handler = rcu_dereference(skb->dev->rx_handler);
	if (rx_handler) {
		if (pt_prev) {
				ret = deliver_skb(skb, pt_prev, orig_dev);
				pt_prev = NULL;
			}
			switch (rx_handler(&skb)) {
			case RX_HANDLER_CONSUMED:
				ret = NET_RX_SUCCESS;
				goto out;
			case RX_HANDLER_ANOTHER:
				goto another_round;
			case RX_HANDLER_EXACT:
				deliver_exact = true;
				break;
			case RX_HANDLER_PASS:
				break;
			default:
			BUG();
		}
	}
  ```

>만약 pfmemalloc이라면 해당 패킷들은 모두 드랍하게 된다.
>
>만약 vlan tag가 있다면, `pt_prev`를 통해 마지막 all type 패킷 처리를 하게 되고, 만약 `vlan_do_receive()` 함수가 `true`라면 새로운 패킷을 처리하러 `another_round:`라벨로 가게 된다.
>여기서 `vlan_do_receive()`함수는 가상랜을 지원하기 위한 함수로, 가상 랜 타입의 패킷이 들어왔을 때, 이를 적절하게 처리해 주기 위해 사용한다.
>주요 역할은 해당하는 가상 랜 장치를 찾아서 그 장치가 유효하고 활성화되어 있는지 확인한 후 패킷의 장치를 해당  가상 랜으로 설정하게 된다.
>
>그 다음으로 rx_handler를 가져오게 되는데, 우선 ice에서는 따로 관련 코드가 없어 넘어가도 되는 것으로 판단하였다. 등록된 rx_handler가 없기 때문에 아에 코드가 수행되지 않는 것이다. 그러나 이 장치에서 들어오는 패킷에 대하여 수행하고자 하는 handler function이 있다면, `netdev_rx_handler_register()`함수를 통해 이를 등록하면 된다. 보통 하나만 등록이 가능하므로, 여러개를 등록하면 안 될 것이다.

```c
	if (unlikely(skb_vlan_tag_present(skb)) && !netdev_uses_dsa(skb->dev)) {
check_vlan_id:
		if (skb_vlan_tag_get_id(skb)) {
			/* Vlan id is non 0 and vlan_do_receive() above couldn't
			* find vlan device.
			*/
			skb->pkt_type = PACKET_OTHERHOST;
		} else if (eth_type_vlan(skb->protocol)) {
			/* Outer header is 802.1P with vlan 0, inner header is
			* 802.1Q or 802.1AD and vlan_do_receive() above could
			* not find vlan dev for vlan id 0.
			*/
			__vlan_hwaccel_clear_tag(skb);
			skb = skb_vlan_untag(skb);
			if (unlikely(!skb))
				goto out;
			if (vlan_do_receive(&skb))
				/* After stripping off 802.1P header with vlan 0
				* vlan dev is found for inner header.
				*/
				goto another_round;
			else if (unlikely(!skb))
				goto out;
			else
				/* We have stripped outer 802.1P vlan 0 header.
				* But could not find vlan dev.
				* check again for vlan id to set OTHERHOST.
				*/
				goto check_vlan_id;
		}
		/* Note: we might in the future use prio bits
		* and set skb->priority like in vlan_do_receive()
		* For the time being, just ignore Priority Code Point
		*/
		__vlan_hwaccel_clear_tag(skb);
	}
	  
	type = skb->protocol;
```

>이 코드는 vlan을 처리하는 코드이다. 바로 윗 덩어리 코드와 상당부분 비슷하다.
>그리고 추가적으로 해당 `net_device`가 dsa를 사용하지 않아야 한다. dsa는 distributed switch architecture로, 이 것 자체로도 설명해야할 것이 많아 일단 넘어가기로 했다.
>
>vlan 아이디를 체크하고, skb의 vlan을 untag한뒤 `vlan_do_receive()`를 호출하게 된다. 만약 정상적으로 호출 되었다면, `another_round`라벨로 가게 되고, 비정상적으로 끝났고 `skb`가 없다면 `out`으로 가게 된다. 그러나 `skb`는 정상이라면 다시 `check_vlan_id`라벨로 가게 된다.

```c
	/* deliver only exact match when indicated */
	if (likely(!deliver_exact)) {
		deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type,
						&ptype_base[ntohs(type) &
							PTYPE_HASH_MASK]);
	}
	  
	deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type,
					&orig_dev->ptype_specific);
	  
	if (unlikely(skb->dev != orig_dev)) {
		deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type,
						&skb->dev->ptype_specific);
	}
	  
	if (pt_prev) {
		if (unlikely(skb_orphan_frags_rx(skb, GFP_ATOMIC)))
			goto drop;
		*ppt_prev = pt_prev;
	} else {
```

>앞서 `rx_handler`가 등록되어 있고, 여기서 `RX_HANDLER_EXACT`를 반환 받았다면 `deliver_exact`가 true가 된다. 이 exact라는 의미는 해당 handler가 등록되어있는 device에서 패킷 처리가 이루어져야 함을 의미한다. 따라서 조건문을 해석해보면, 이러한 반환을 받지 않은 경우에, 그 아래의 `deliver_ptype_list_skb()`를 수행하게된다.
>이 때 `ptype_base`는 본 /net/core/dev.c의 164번 라인에 global variable로 선언되어 있다.
>
>이는 `ptype_head()`라는 함수와 깊게 연관되어있다. 이 함수를 호출하는 함수인 `dev_add_pack()`이라는 함수는 위의 `ptype_head()`라는 함수가 찾아 준 list_head에 해당 `packet_type` 구조체를 넣게 된다.
>
>순서대로 해당 패킷과 패킷 핸들러가 `device`가 매치되지 않는 경우를 제외하고 특정 타입을 위한 패킷 핸들러들이 들어있는 전역 변수인 `ptype_base`에서 iteration을 통해 패킷들이 핸들링 되고,
>
>두 번째로 해당 패킷의 original device에 등록되어 있는 패킷 핸들러들이 들어있는 `dev->ptype_specific`에서 iteration을 통해 패킷들이 핸들링 되고,
>
>마지막으로 만약 `skb->dev`가 `orig_dev`와 다르다면 `skb->dev->ptype_specific`에서 iteration을 통해 패킷들이 핸들링 되게 된다. 여기서 `orig_dev`는 위의 vlan과 traffic control을 거치기 전에 원래의 `skb->dev`를 가르키고 있다. 중간에 변경될 여지가 있는가 싶다. 참고로 IP 프로토콜과 관련하여 추가되는 `ip_packet_type` 패킷 핸들러는 `dev`가 정의되어 있지 않으므로 global table인 `ptype_base`에 들어가게 된다. 따라서 위에서 해당 핸들러의 `func`인 `ip_rcv`가 실행되게 된다.

# 여기 중요한 포인트

>따라서 `ptype_head()`라는 함수를 살펴보면, 아래와 같다

```c
static inline struct list_head *ptype_head(const struct packet_type *pt)
{
	if (pt->type == htons(ETH_P_ALL))
		return pt->dev ? &pt->dev->ptype_all : &net_hotdata.ptype_all;
	else
		return pt->dev ? &pt->dev->ptype_specific :
				&ptype_base[ntohs(pt->type) & PTYPE_HASH_MASK];
}
```

>우선 해당 패킷 핸들러가 다루는 타입이 모든 패킷에 대한 것이라면, 첫 번째 if문을 타게 된다.
>
>>여기서 만약 `net_device`가 특정되어 있다면 해당 디바이스의 `ptype_all`리스트에 넣어주고, 아니라면 전역변수인 `net_notdata`에 넣게 된다.
>
>해당 패킷 핸들러가 다루는 타입이 특정 패킷 타입에 대한 것이라면, 두 번째 else문을 타게 된다.
>
>>여기서 만약 `net_device`가 특정되어 있다면 해당 디바이스의 `ptype_specific`리스트에 넣어주고, 아니라면 /net/core/dev.c의 ptype_base의 특정 index에 넣게 된다.
>
>저기 상수값인 `PTYPE_HASH_MASK`의 경우 `PTYPE_HASH_SIZE`에서 1을 뺀 값인데, 즉 0xFFFF를 만들기 위함으로 보인다. 왜냐면 패킷 타입은 16비트를 사용하기 때문이다.
>
>따라서 총 4개의 ptype list 가 존재하게 된다.

>다시 위에서 부터 이어가자면, 첫 번째 `deliver_ptype_list_skb()`의 경우 해당 패킷이 `deliver_exact`가 아니여서 반드시 특정 device에서 실행되지 않아도 되는 경우 이를 위해 전역 변수인 ptype_base를 가지고 `deliver_skb()`를 실행하게 된다.
>
>두 번째는 그냥 바로 `orig_dev`의 ptype list를 가지고 실행하게 된다.
>
>세 번째는 현재 skb가 가르키고 있는 `net_device`의 ptype list를 가지고 실행하게 된다.
>
>즉 전역변수에서, 그리고 본 함수가 실행될 때 들어온 첫 번째 패킷의 `net_dev`에서, 그리고 현재 패킷의 `net_dev`에서 `ptype_specific`을 가져와서 실행을 시키게 되는 것이다.

```c
drop:
			if (!deliver_exact)
				dev_core_stats_rx_dropped_inc(skb->dev);
			else
				dev_core_stats_rx_nohandler_inc(skb->dev);
			kfree_skb_reason(skb, SKB_DROP_REASON_UNHANDLED_PROTO);
			/* Jamal, now you will not able to escape explaining
			* me how you were going to use this. :-)
			*/
			ret = NET_RX_DROP;
	}
	  
out:
	/* The invariant here is that if *ppt_prev is not NULL
	* then skb should also be non-NULL.
	*
	* Apparently *ppt_prev assignment above holds this invariant due to
	* skb dereferencing near it.
	*/
	*pskb = skb;
	return ret;
}
```

>나머지는 `drop:`과 `out:` 라벨이다.


