### 0. 신규 구조체들에 대한 선언 확인 및 분석
---

```c
/*
* The rps_dev_flow structure contains the mapping of a flow to a CPU, the
* tail pointer for that CPU's input queue at the time of last enqueue, and
* a hardware filter index.
*/
struct rps_dev_flow {
	u16 cpu;
	u16 filter;
	unsigned int last_qtail;
};
#define RPS_NO_FILTER 0xffff
```

```c
/*
* The rps_sock_flow_table contains mappings of flows to the last CPU
* on which they were processed by the application (set in recvmsg).
* Each entry is a 32bit value. Upper part is the high-order bits
* of flow hash, lower part is CPU number.
* rps_cpu_mask is used to partition the space, depending on number of
* possible CPUs : rps_cpu_mask = roundup_pow_of_two(nr_cpu_ids) - 1
* For example, if 64 CPUs are possible, rps_cpu_mask = 0x3f,
* meaning we use 32-6=26 bits for the hash.
*/
struct rps_sock_flow_table {
	u32 mask;
	  
	u32 ents[] ____cacheline_aligned_in_smp;
};
```

`mask`가 `ents`의 영역을 구분하게 됨. 예를 들어 가용 코어가 128개라면, `mask`는 `0x7f`가 되고, ents의 32-7 = 25개의 bit가 hash를 위한 영역이라는 것이다.

```c
/* This structure contains an instance of an RX queue. */
 
struct netdev_rx_queue {
	struct xdp_rxq_info xdp_rxq;
#ifdef CONFIG_RPS
	struct rps_map __rcu *rps_map;
	struct rps_dev_flow_table __rcu *rps_flow_table;
#endif
	struct kobject kobj;
	struct net_device *dev;
	netdevice_tracker dev_tracker;
  
#ifdef CONFIG_XDP_SOCKETS
	struct xsk_buff_pool *pool;
#endif
	/* NAPI instance for the queue
	* Readers and writers must hold RTNL
	*/
	struct napi_struct *napi;
} ____cacheline_aligned_in_smp;
```

```c
/*
* The rps_dev_flow_table structure contains a table of flow mappings.
*/
struct rps_dev_flow_table {
	unsigned int mask;
	struct rcu_head rcu;
	struct rps_dev_flow flows[];
};
```

```c
/*
* This structure holds an RPS map which can be of variable length. The
* map is an array of CPUs.
*/
struct rps_map {
	unsigned int len;
	struct rcu_head rcu;
	u16 cpus[];
};
```

rcu_dereference() -> atomic을 지키면서 해당 자료를 읽어오는 함수.

```c
/* Read mostly data used in network fast paths. */
struct net_hotdata {
#if IS_ENABLED(CONFIG_INET)
	struct packet_offload ip_packet_offload;
	struct net_offload tcpv4_offload;
	struct net_protocol tcp_protocol;
	struct net_offload udpv4_offload;
	struct net_protocol udp_protocol;
	struct packet_offload ipv6_packet_offload;
	struct net_offload tcpv6_offload;
#if IS_ENABLED(CONFIG_IPV6)
	struct inet6_protocol tcpv6_protocol;
	struct inet6_protocol udpv6_protocol;
#endif
	struct net_offload udpv6_offload;
#endif
	struct list_head offload_base;
	struct list_head ptype_all;
	struct kmem_cache *skbuff_cache;
	struct kmem_cache *skbuff_fclone_cache;
	struct kmem_cache *skb_small_head_cache;
#ifdef CONFIG_RPS
	struct rps_sock_flow_table __rcu *rps_sock_flow_table;
	u32 rps_cpu_mask;
#endif
	int gro_normal_batch;
	int netdev_budget;
	int netdev_budget_usecs;
	int tstamp_prequeue;
	int max_backlog;
	int dev_tx_weight;
	int dev_rx_weight;
};
```

> `/include/net/hotdata.h`에서 구조체를 정의하고, 여기서 하나 변수를 선언하게 된다.
> `extern struct net_hotdata net_hotdata;` 구문인데, `extern`키워드는 다른 파일에서도 이 구조체를 참조할 수 있도록 하는 키워드이다.
> 이후, `/net/core/hotdata.c`에서 이 `net_hotdata`를 초기화하게 된다. 일부 초기값들이 설정되어 있다.
> `net_hotdata`는 네트워크 스택이 진행 될 때, 정말 자주 참조하는 변수들을 한데 모아서 오버헤드를 줄이고 성능향상을 꾀하고 있다.

>`net_hotdata.rps_cpu_mask`를 설정하는 부분은 `/net/core/sysctl_net_core.c`파일에 `rps_sock_flow_sysctl()`함수에 있으며, 해당 파일의 165번 라인에 있다. 여기서는 코어의 갯수를 세서 2의 제곱 수중 가장 가까운 수로 설정해준다.

구조체들 간의 상관관계 도식화
[[RPS-IPI 관계도]]

### 1. RPS를 다루는 함수들 확인 및 분석
---

```c
static inline void rps_record_sock_flow(struct rps_sock_flow_table *table,
					u32 hash)
{
	unsigned int index = hash & table->mask;
	u32 val = hash & ~net_hotdata.rps_cpu_mask;
  
	/* We only give a hint, preemption can change CPU under us */
	val |= raw_smp_processor_id();
	  
	/* The following WRITE_ONCE() is paired with the READ_ONCE()
	* here, and another one in get_rps_cpu().
	*/
	if (READ_ONCE(table->ents[index]) != val)
		WRITE_ONCE(table->ents[index], val);
}
```

>해당 `hash`를 가진 패킷에 대하여 테이블에 해당 정보를 저장하는 함수이다. 여기서는 테이블의 마스크와 해시를 and 연산하여 인덱스를 추출하고, 저장 할 값으로는 hash와 `rps_cpu_mask`의 비트 반전 간의 and 연산 결과에 현재 이 함수를 실행 중인 코어의 ID를 or연산으로 가져오게 된다.

>우선 `get_rps_cpu()`함수 내부에서 `ident`로 선언 된 변수는 `sock_flow_table`에서`hash`에 마스크를 씌운 값을 인덱스로 하여 가져오게 된다. `ents`에 들어있는 `unsigned 32bit` 타입은 상위 비트와 하위비트를 같은 `sock_flow_table`에서 가지고 있는 `mask`로 구분하게 된다.
>
>이 구분을 통해 상위 비트는 `flow_hash`값을 나타내게 되고, 하위 비트는 `CPU number`를 나타내게 된다.
>
>`RPS_NO_CPU`라는 값은 `0xffff`이다.
>
>`table`은 `rps_sock_flow_table`타입의 구조체로, 여기서 `mask`를 꺼내와서 index를 찾게 된다. 이 index에 저장할 `val`은 원래 `hash`를 아래 CPU 비트만큼을 잘라내고 현재 본 코드가 동작중인 CPU의 Id값을 집어 넣게 된다.

### 2. 코드 분석
---
```c
/* First check into global flow table if there is a match.
		 * This READ_ONCE() pairs with WRITE_ONCE() from rps_record_sock_flow().
		 */
		ident = READ_ONCE(sock_flow_table->ents[hash & sock_flow_table->mask]);
		if ((ident ^ hash) & ~net_hotdata.rps_cpu_mask)
			goto try_rps;

		next_cpu = ident & net_hotdata.rps_cpu_mask;

		/* OK, now we know there is a match,
		 * we can look at the local (per receive queue) flow table
		 */
		rflow = &flow_table->flows[hash & flow_table->mask];
		tcpu = rflow->cpu;

```

>`get_rps_cpu()`함수의 중간 부분을 발췌해왔다. 이 코드의 앞부분은 주요 변수들이 모두 존재하는지 확인하고, 아니라면 `goto done;`등의 작업을 통해 `rps`가 끝나도록 구성되어 있다.
>
>우선 `ident` 변수부터 살펴보자면, 바로 `sock_flow_table`의 한 `ents`의 내용물을 가지고 있음을 알 수 있다. 여기서 내용물은 특정 flow hash의 상위 비트들과 CPU 아이디로 이루어진 값이다.
>
>`if`문 안에서는 ident에 저장된 flow hash와 현재 비교하고자 하는 hash 값을 비교하게 되는데, 이 둘이 다르다면 `try_rps`라벨로 가게 된다. 즉, 여기서 조건문이 의미하는 것은 현재 작업중인 패킷의 해시값이 `rps table`에 저장된 `hash`값과 다를 때라는 것이다.
>만약 같다면 계속해서 코드가 수행 될 것이다. 그렇다면, `next_cpu`는 `sock_flow_table`에 저장 된 해당 `hash`에 해당하는 `cpu id`로 설정되게 될 것이다.
>또한, 현재 flow에 해당하는 `rflow`값을 가져오게 되는데, 이는 `local flow table`에서 가져오게 된다.
>
>> 주석에 의하면, `sock_flow_table`은 global flow table이 되고,
>> `flow_table`은 local(per receive queue) flow table이 되게 된다.
>
>여기서 `tcpu`값을 가져오게 된다.


```c
/*
		 * If the desired CPU (where last recvmsg was done) is
		 * different from current CPU (one in the rx-queue flow
		 * table entry), switch if one of the following holds:
		 *   - Current CPU is unset (>= nr_cpu_ids).
		 *   - Current CPU is offline.
		 *   - The current CPU's queue tail has advanced beyond the
		 *     last packet that was enqueued using this table entry.
		 *     This guarantees that all previous packets for the flow
		 *     have been dequeued, thus preserving in order delivery.
		 */
		if (unlikely(tcpu != next_cpu) &&
		    (tcpu >= nr_cpu_ids || !cpu_online(tcpu) ||
		     ((int)(per_cpu(softnet_data, tcpu).input_queue_head -
		      rflow->last_qtail)) >= 0)) {
			tcpu = next_cpu;
			rflow = set_rps_cpu(dev, skb, rflow, next_cpu);
		}

		if (tcpu < nr_cpu_ids && cpu_online(tcpu)) {
			*rflowp = rflow;
			cpu = tcpu;
			goto done;
		}
	}
```

> ![[Pasted image 20240816103722.png]]
> 참조한 공식문서에서는 분명히 RPS는 application specific한 packet steering을 하지 않는다고 하였지만, global flow table에서는 해당 hash값의 패킷을 다룬 CPU의 정보를 매핑하여 저장하고 있었다. 이는  `recvmsg()`안에서 이루어진다고 한다.
> ![[Pasted image 20240826133557.png]]
> 두 개의 flow table을 유지하는 이유는 out of order를 방지하기 위함이다. 현재 처리 중인 패킷이 다 처리되었는지 확인하는 flow table이 있다.
> 
>주석의 내용을 분석해보자면, 우선 `desired CPU`란 마지막으로 `userspace`로 패킷을 올려보낸 CPU를 의미하며, `current CPU`란 local flow table에 저장된 CPU를 의미한다. 만약 둘이 다르고, 다음 셋 중 하나를 만족하게 된다면, CPU 전환을 수행하게 된다.
>
> - Current CPU가 설정되지 않은 경우 -> `>=nr_cpu_ids` -> `RPS_NO_CPU`값이 `0xffff`이기 때문이다.
> - Current CPU가 offline 인 경우
> - 이 table entry를 사용한 큐잉된 마지막 패킷을 current CPU의 queue tail을 넘어 선 경우. -> 이 경우는 해당 flow의 모든 패킷이 디큐잉되었다는 것을 보장하므로, 순서가 보존된 채로 전달이 가능함을 보장할 수 있다.
>
>위와 같은 상황인 경우 `tcpu`를 `next_cpu`값으로 설정하게 된다. 즉, `tcpu` 값을 application이 작동 중인 것으로 보이는 CPU로 설정하는 것이다.
>
>또한, `set_rps_cpu()`함수를 호출하여 이를 `rflow`에 저장하게 된다. [[set_rps_cpu()]]
>
>공식문서의 설명에서 볼 수 있듯이, 바로 위의 코드 중 첫번째 `if`문은 RFS를 사용하기 위한 코드들로 보인다.
>그 다음은 `tcpu`가 유효한지 확인하는 것인데, `tcpu`가 유효 범위 내부이면서, 해당 코어가 켜져있을 경우, 이`skb`가 가지는 `rflow`를 해당 `*rflowp` 포인터에 할당함으로써 `get_rps_cpu()`함수가 종료되더라도 이 정보를 가지고 갈 수 있게끔 하였다.
>그후 `goto done;`을 통해 해당 `tcpu`가 반환 될 수 있도록 하였다.

```c
try_rps:

	if (map) {
		tcpu = map->cpus[reciprocal_scale(hash, map->len)];
		if (cpu_online(tcpu)) {
			cpu = tcpu;
			goto done;
		}
	}

done:
	return cpu;
}
```

>마지막 코드 조각이다.
>앞서 `flow_table`이 있을 때 여러 과정들을 거치게 되고, 여기서는 `map`변수가 유효한지 먼저 검사를 하게 된다. 만약 `NULL` 혹은 유효하지 않은 값이라면, `cpu`는 -1로 그대로 반환되는데, 이는 rps를 통해 매핑 할 cpu가 없음을 의미한다.
>
>이어서 보자면, `reciprocal_scale()`함수가 사용되는데 이는 `hash`값을 0보다 크거나 같고, `map->len`보다 작은 범위내로 스케일링 해주는 함수이다. 따라서 모든 해시값에 대하여 대응하는`cpus` array의 index를 획득할 수 있게 된다.
>`reciprocal_scale(u32 val, u32 ep_ro)`는 다음 값을 반환한다. `return (u32)(((u64) val * ep_ro) << 32)` 즉, 두 값을 곱해서 2^32 값으로 나누어준 몫이다. 
>shift 연산과 곱셈 연산은 같은 연산 우위이므로 이를 바꾸어보면 좀더 이해하기 쉽다. val값을 32만큼 left shift하면 0밖에 안남을 것이다. 그러나 이는 컴퓨터 공학적 관점이고, 그냥 수학적으로 보면 0과 1사이의 소수 값일 것이다. u32 전체 범위를 0부터 1까지 범위로 압축한 것이다. 여기다가 범위인 `ep_ro`를 곱하면 주어진 범위로 스케일링이 되는 원리이다.
>
>
>따라서 hash값을 바탕으로 tcpu를 획득할 수 있게 되고, 만약 해당 코어가 온라인이라면 이를 `cpu` 변수에 할당하고 `goto done;`을 하게 된다. >>>> 근데 굳이 `goto done;`을 해야할까? 끝나면 그대로 순차적으로 실행되는 것 아닌가?

### 3. rps_map 세팅
---
>`rps_map`은 위의 코드에서 단순히 사용만되었다. 따라서 언제, 어떻게 초기화 되고 있는지 확인해보았다

```c
static int netdev_rx_queue_set_rps_mask(struct netdev_rx_queue *queue,
					cpumask_var_t mask)
{
	static DEFINE_MUTEX(rps_map_mutex);
	struct rps_map *old_map, *map;
	int cpu, i;
	  
	map = kzalloc(max_t(unsigned int,
				RPS_MAP_SIZE(cpumask_weight(mask)), L1_CACHE_BYTES),
			  GFP_KERNEL);
	if (!map)
		return -ENOMEM;
	  
	i = 0;
	for_each_cpu_and(cpu, mask, cpu_online_mask)
		map->cpus[i++] = cpu;
	  
	if (i) {
		map->len = i;
	} else {
		kfree(map);
		map = NULL;
	}
	  
	mutex_lock(&rps_map_mutex);
	old_map = rcu_dereference_protected(queue->rps_map,
						mutex_is_locked(&rps_map_mutex));
	rcu_assign_pointer(queue->rps_map, map);
	  
	if (map)
		static_branch_inc(&rps_needed);
	if (old_map)
		static_branch_dec(&rps_needed);
	  
	mutex_unlock(&rps_map_mutex);
	  
	if (old_map)
		kfree_rcu(old_map, rcu);
	return 0;
}
```

>코드를 보면, 중간에 15번 라인에서 볼 수 있듯이, RPS가 가능한 CPU들을 가르키고 있는 `mask`와, 현재 online 상태인 CPU들을 가르키고 있는 `cpu_online_mask` 두 마스크를 가지고 RPS가 가능한 CPU들의 array를 만들고 있다. index의 경우 1씩 늘어나고 있고, 여기에 저장되는 값은 RPS 가능한 CPU의 id이다.
>
>해당 마스크는 특정 010010 과 같은 string에서 추출해서 만들어지게 된다. 이는 앞서 찾았던 `rps_cpus`파일을 읽어와서 만드는 것으로 보인다.



