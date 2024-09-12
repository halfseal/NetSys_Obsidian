![[황재훈/Research Intern/pastNotion/intel ice/NAPI/img/Untitled.png|Untitled.png]]

### RX 의 과정 요약

1. pkt이 NIC에 도착
2. DMA (NIC이 kernel mem으로)
3. NIC이 cpu로 hw IRQ 전송 (CPU가 바로 처리)
4. driver가 NAPI 실행
5. ~~ksoftirqd 가 NAPI polling을 실행~~
    1. net rx action
    2. napi poll을 했는데, 일이 남으면 napi polling실행한다
    3. do softirqd function
6. mapping 된 ring buffer unmapp
7. mem으로 DMA된 pkt은 skb로 바뀌어서 처리된다
8. 반복

### NAPI

1. NAPI는 꺼져있는 상태로 있다
2. NIC이 pkt을 받으면 irq를 생성한다
3. driver가 irq를 받고 irq handler를 실행한다
4. irq handler가 softirq를 통해 NAPI를 실행한다
5. driver가 더이상의 irq를 보내지 못하게한다
6. NAPI polling에 의해 pkt을 처리한다
7. 더 이상 처리할 데이터가 없으면 NAPI가 꺼지고 irq를 허용한다
8. 반복

  

- ==`ice_vsi_alloc_q_vector()`==
    
    ```JavaScript
    int ice_vsi_alloc_q_vectors(struct ice_vsi *vsi)
    {
    	struct device *dev = ice_pf_to_dev(vsi->back);
    	u16 v_idx;
    	int err;
    
    	if (vsi->q_vectors[0]) {
    		dev_dbg(dev, "VSI %d has existing q_vectors\n", vsi->vsi_num);
    		return -EEXIST;
    	}
    
    	for (v_idx = 0; v_idx < vsi->num_q_vectors; v_idx++) {
    		err = ice_vsi_alloc_q_vector(vsi, v_idx);
    		if (err)
    			goto err_out;
    	}
    
    	return 0;
    
    err_out:
    	while (v_idx--)
    		ice_free_q_vector(vsi, v_idx);
    
    	dev_err(dev, "Failed to allocate %d q_vector for VSI %d, ret=%d\n",
    		vsi->num_q_vectors, vsi->vsi_num, err);
    	vsi->num_q_vectors = 0;
    	return err;
    }
    
    /**
     * ice_vsi_alloc_q_vector - Allocate memory for a single interrupt vector
     * @vsi: the VSI being configured
     * @v_idx: index of the vector in the VSI struct
     *
     * We allocate one q_vector and set default value for ITR setting associated
     * with this q_vector. If allocation fails we return -ENOMEM.
     */
    static int ice_vsi_alloc_q_vector(struct ice_vsi *vsi, u16 v_idx)
    {
    	struct ice_pf *pf = vsi->back;
    	struct ice_q_vector *q_vector;
    	int err;
    
    	/* allocate q_vector */
    	q_vector = kzalloc(sizeof(*q_vector), GFP_KERNEL);
    	if (!q_vector)
    		return -ENOMEM;
    
    	q_vector->vsi = vsi;
    	q_vector->v_idx = v_idx;
    	q_vector->tx.itr_setting = ICE_DFLT_TX_ITR;
    	q_vector->rx.itr_setting = ICE_DFLT_RX_ITR;
    	q_vector->tx.itr_mode = ITR_DYNAMIC;
    	q_vector->rx.itr_mode = ITR_DYNAMIC;
    	q_vector->tx.type = ICE_TX_CONTAINER;
    	q_vector->rx.type = ICE_RX_CONTAINER;
    	q_vector->irq.index = -ENOENT;
    
    	if (vsi->type == ICE_VSI_VF) {
    		q_vector->reg_idx = ice_calc_vf_reg_idx(vsi->vf, q_vector);
    		goto out;
    	} else if (vsi->type == ICE_VSI_CTRL && vsi->vf) {
    		struct ice_vsi *ctrl_vsi = ice_get_vf_ctrl_vsi(pf, vsi);
    
    		if (ctrl_vsi) {
    			if (unlikely(!ctrl_vsi->q_vectors)) {
    				err = -ENOENT;
    				goto err_free_q_vector;
    			}
    
    			q_vector->irq = ctrl_vsi->q_vectors[0]->irq;
    			goto skip_alloc;
    		}
    	}
    
    	q_vector->irq = ice_alloc_irq(pf, vsi->irq_dyn_alloc);
    	if (q_vector->irq.index < 0) {
    		err = -ENOMEM;
    		goto err_free_q_vector;
    	}
    
    skip_alloc:
    	q_vector->reg_idx = q_vector->irq.index;
    
    	/* only set affinity_mask if the CPU is online */
    	if (cpu_online(v_idx))
    		cpumask_set_cpu(v_idx, &q_vector->affinity_mask);
    
    	/* This will not be called in the driver load path because the netdev
    	 * will not be created yet. All other cases with register the NAPI
    	 * handler here (i.e. resume, reset/rebuild, etc.)
    	 */
    	if (vsi->netdev)
    		netif_napi_add(vsi->netdev, &q_vector->napi, ice_napi_poll);
    
    out:
    	/* tie q_vector and VSI together */
    	vsi->q_vectors[v_idx] = q_vector;
    
    	return 0;
    
    err_free_q_vector:
    	kfree(q_vector);
    
    	return err;
    }
    ```
    

1. ice_probe() → ice_init() → ice_vsi_init() → ice vsi alloc q vector()
2. q_vector을 위한 공간을 할당하는 함수이다
    
    1. q_vector 내부에 vsi, napi_struct, rx_queue, tx_queue 있다
    
    - `struct ice_q_vector`
        
        ```JavaScript
        struct ice_q_vector {
        	struct ice_vsi *vsi;
        
        	u16 v_idx;			/* index in the vsi->q_vector array. */
        	u16 reg_idx;
        	u8 num_ring_rx;			/* total number of Rx rings in vector */
        	u8 num_ring_tx;			/* total number of Tx rings in vector */
        	u8 wb_on_itr:1;			/* if true, WB on ITR is enabled */
        	/* in usecs, need to use ice_intrl_to_usecs_reg() before writing this
        	 * value to the device
        	 */
        	u8 intrl;
        
        	struct napi_struct napi;
        
        	struct ice_ring_container rx;
        	struct ice_ring_container tx;
        
        	cpumask_t affinity_mask;
        	struct irq_affinity_notify affinity_notify;
        
        	struct ice_channel *ch;
        
        	char name[ICE_INT_NAME_STR_LEN];
        
        	u16 total_events;	/* net_dim(): number of interrupts processed */
        	struct msi_map irq;
        } ____cacheline_internodealigned_in_smp;
        ```
        
3. 이미 vsi에 q_vector가 존재하면 넘어간다
4. `q_vector = kzalloc(sizeof(*q_vector), GFP_KERNEL)`
    1. q_vector를 kzalloc()을 통해 공간 할당한다
    2. kzalloc() = kmalloc() + memset(), 연속된 공간 할당 + 0으로 초기화
    3. GFP_KERNEL : get free page, kernel data struct, inode cache, DMAable mem 에 사용
        1. GFP_NOWAIT (atomic context에서 allocation 할때 사용)
        2. GFP_NOWARN (Allocations which have a reasonable fallback should be using)
        3. GFP_ATOMIC….
5. vf일 경우 따로 처리를 해준다
    1. ice_alloc_irq 안함, 이유가 뭘까?
        - ~~아마 vf일경우 vf를 실행하고 잇는 물리적인 device가 irq를 먼저 할당해 놔서?~~
6. `q_vector->irq = ice_alloc_irq(pf, vsi->irq_dyn_alloc)`
    
    1. q_vector 마다 irq를 할당한다
    
    - `struct msi_map ice_alloc_irq(struct ice_pf *pf, bool dyn_only)`
        
        ```JavaScript
        /**
         * ice_alloc_irq - Allocate new interrupt vector
         * @pf: board private structure
         * @dyn_only: force dynamic allocation of the interrupt
         *
         * Allocate new interrupt vector for a given owner id.
         * return struct msi_map with interrupt details and track
         * allocated interrupt appropriately.
         *
         * This function reserves new irq entry from the irq_tracker.
         * if according to the tracker information all interrupts that
         * were allocated with ice_pci_alloc_irq_vectors are already used
         * and dynamically allocated interrupts are supported then new
         * interrupt will be allocated with pci_msix_alloc_irq_at.
         *
         * Some callers may only support dynamically allocated interrupts.
         * This is indicated with dyn_only flag.
         *
         * On failure, return map with negative .index. The caller
         * is expected to check returned map index.
         *
         */
        struct msi_map ice_alloc_irq(struct ice_pf *pf, bool dyn_only)
        {
        	int sriov_base_vector = pf->sriov_base_vector;
        	struct msi_map map = { .index = -ENOENT };
        	struct device *dev = ice_pf_to_dev(pf);
        	struct ice_irq_entry *entry;
        
        	entry = ice_get_irq_res(pf, dyn_only);
        	if (!entry)
        		return map;
        
        	/* fail if we're about to violate SRIOV vectors space */
        	if (sriov_base_vector && entry->index >= sriov_base_vector)
        		goto exit_free_res;
        
        	if (pci_msix_can_alloc_dyn(pf->pdev) && entry->dynamic) {
        		map = pci_msix_alloc_irq_at(pf->pdev, entry->index, NULL);
        		if (map.index < 0)
        			goto exit_free_res;
        		dev_dbg(dev, "allocated new irq at index %d\n", map.index);
        	} else {
        		map.index = entry->index;
        		map.virq = pci_irq_vector(pf->pdev, map.index);
        	}
        
        	return map;
        
        exit_free_res:
        	dev_err(dev, "Could not allocate irq at idx %d\n", entry->index);
        	ice_free_irq_res(pf, entry->index);
        	return map;
        }
        ```
        
        - msix (messeage signaled interrupt - x)
            - 권장되는 interrrupt 방식, 특히 multi rx queue NIC에서
            - 더 flexible 하다고 한다??
            - rx queue가 그것만의 hw interrupt를 할당할 수 있다. 이는 특정 cpu에 의해 관리된다?
            - This is because each RX queue can ==**have its own hardware interrupt assigned**==, which can then be handled by a specific CPU (with `irqbalance` or by modifying `/proc/irq/IRQ_NUMBER/smp_affinity`) ???
    
    - interrupt vector 를 할당해준다
        - interrupt vector는 interrupt handler 으로의 주소가 적혀있다
        - Interrupt는 0에서 255까지 숫자와 연결되어 있고 interrupt vector와도 연결되어 있다.
        - interrupt vector을 모아 놓은게 interrupt vector table
        - interrupt이 발생할 경우, 이에 맞는 interrupt handler를 실행할 수 있다.
7. `netif_napi_add(vsi->netdev, &q_vector->napi, ice_napi_poll);`
    1. netdevice가 존재할 경우 실행, ~~net device가 없으면 napi를 실행못하니까?~~
    2. `ice_napi_poll()` 을 polling function으로 사용
        1. tx queue 초기화, budget 분배, rx queue 초기화

  

- ==`netif_napi_add()`== ==: initalize napi context==
    
    ```JavaScript
    /**
     * netif_napi_add() - initialize a NAPI context
     * @dev:  network device
     * @napi: NAPI context
     * @poll: polling function
     *
     * netif_napi_add() must be used to initialize a NAPI context prior to calling
     * *any* of the other NAPI-related functions.
     */
    static inline void
    netif_napi_add(struct net_device *dev, struct napi_struct *napi,
    	       int (*poll)(struct napi_struct *, int))
    {
    	netif_napi_add_weight(dev, napi, poll, NAPI_POLL_WEIGHT);
    }
    \#define NAPI_POLL_WEIGHT 64
    
    void netif_napi_add_weight(struct net_device *dev, struct napi_struct *napi,
    			   int (*poll)(struct napi_struct *, int), int weight)
    {
    	if (WARN_ON(test_and_set_bit(NAPI_STATE_LISTED, &napi->state)))
    		return;
    
    	INIT_LIST_HEAD(&napi->poll_list);
    	INIT_HLIST_NODE(&napi->napi_hash_node);
    	hrtimer_init(&napi->timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL_PINNED);
    	napi->timer.function = napi_watchdog;
    	init_gro_hash(napi);
    	napi->skb = NULL;
    	INIT_LIST_HEAD(&napi->rx_list);
    	napi->rx_count = 0;
    	napi->poll = poll;
    	if (weight > NAPI_POLL_WEIGHT)
    		netdev_err_once(dev, "%s() called with weight %d\n", __func__,
    				weight);
    	napi->weight = weight;
    	napi->dev = dev;
    \#ifdef CONFIG_NETPOLL
    	napi->poll_owner = -1;
    \#endif
    	napi->list_owner = -1;
    	set_bit(NAPI_STATE_SCHED, &napi->state);
    	set_bit(NAPI_STATE_NPSVC, &napi->state);
    	list_add_rcu(&napi->dev_list, &dev->napi_list);
    	napi_hash_add(napi);
    	napi_get_frags_check(napi);
    	/* Create kthread for this napi if dev->threaded is set.
    	 * Clear dev->threaded if kthread creation failed so that
    	 * threaded mode will not be enabled in napi_enable().
    	 */
    	if (dev->threaded && napi_kthread_create(napi))
    		dev->threaded = 0;
    	netif_napi_set_irq(napi, -1);
    }
    ```
    
- `napi_struct`
    
    ```JavaScript
    struct napi_struct {
    	/* The poll_list must only be managed by the entity which
    	 * changes the state of the NAPI_STATE_SCHED bit.  This means
    	 * whoever atomically sets that bit can add this napi_struct
    	 * to the per-CPU poll_list, and whoever clears that bit
    	 * can remove from the list right before clearing the bit.
    	 */
    	struct list_head	poll_list;
    
    	unsigned long		state;
    	int			weight;
    	int			defer_hard_irqs_count;
    	unsigned long		gro_bitmask;
    	int			(*poll)(struct napi_struct *, int);
    \#ifdef CONFIG_NETPOLL
    	/* CPU actively polling if netpoll is configured */
    	int			poll_owner;
    \#endif
    	/* CPU on which NAPI has been scheduled for processing */
    	int			list_owner;
    	struct net_device	*dev;
    	struct gro_list		gro_hash[GRO_HASH_BUCKETS];
    	struct sk_buff		*skb;
    	struct list_head	rx_list; /* Pending GRO_NORMAL skbs */
    	int			rx_count; /* length of rx_list */
    	unsigned int		napi_id;
    	struct hrtimer		timer;
    	struct task_struct	*thread;
    	/* control-path-only fields follow */
    	struct list_head	dev_list;
    	struct hlist_node	napi_hash_node;
    	int			irq;
    };
    ```
    

1. `netif_napi_add_weight(dev, napi, poll, NAPI_POLL_WEIGHT);` 가 안에 있다
    1. NAPI_POLL_WEIGHT == 64
2. `if (WARN_ON(test_and_set_bit(NAPI_STATE_LISTED, &napi->state)))`
    1. 이미 napi에 list가 있는 상태면 넘어간다
3. list의 초기값 설정, timer 초기값 설정
4. `init_gro_hash(napi);`
    
    - `static void init_gro_hash(struct napi_struct *napi)`
        
        ```JavaScript
        static void init_gro_hash(struct napi_struct *napi)
        {
        	int i;
        
        	for (i = 0; i < GRO_HASH_BUCKETS; i++) {
        		INIT_LIST_HEAD(&napi->gro_hash[i].list);
        		napi->gro_hash[i].count = 0;
        	}
        	napi->gro_bitmask = 0;
        }
        ```
        
    
    1. gro 준비
    2. gro_hash_bucket의 개수만큼 list를 초기화 해준다
5. napi 에 skb가 있는 이유
    1. napi_struct를 이어서 poll list를 만든다 ?
6. `napi_kthread_create(napi)`
    
    - code
        
        ```JavaScript
        static int napi_kthread_create(struct napi_struct *n)
        {
        	int err = 0;
        
        	/* Create and wake up the kthread once to put it in
        	 * TASK_INTERRUPTIBLE mode to avoid the blocked task
        	 * warning and work with loadavg.
        	 */
        	n->thread = kthread_run(napi_threaded_poll, n, "napi/%s-%d",
        				n->dev->name, n->napi_id);
        	if (IS_ERR(n->thread)) {
        		err = PTR_ERR(n->thread);
        		pr_err("kthread_run failed with err %d\n", err);
        		n->thread = NULL;
        	}
        
        	return err;
        }
        ```
        
    
    1. kthread를 만들면서 깨우고 interruptible mode로 설정한다
    2. blocked task warning을 피하고 loadavg를 사용하기 위해 한다 ????
7. `netif_napi_set_irq(napi, -1);`
    1. napi_struct에 있는 irq 변수를 -1로 설정한다
    2. enum 으로 설정되어 있을 것 같은데 -1로 설정한 것 보아 irq disable 한거랑 연관있을 듯?
    3. napi->irq = irq;

- Types of NAPI state
    
    ```JavaScript
    enum {
    	NAPI_STATE_SCHED,		/* Poll is scheduled */
    	NAPI_STATE_MISSED,		/* reschedule a napi */
    	NAPI_STATE_DISABLE,		/* Disable pending */
    	NAPI_STATE_NPSVC,		/* Netpoll - don't dequeue from poll_list */
    	NAPI_STATE_LISTED,		/* NAPI added to system lists */
    	NAPI_STATE_NO_BUSY_POLL,	/* Do not add in napi_hash, no busy polling */
    	NAPI_STATE_IN_BUSY_POLL,	/* sk_busy_loop() owns this NAPI */
    	NAPI_STATE_PREFER_BUSY_POLL,	/* prefer busy-polling over softirq processing*/
    	NAPI_STATE_THREADED,		/* The poll is performed inside its own thread*/
    	NAPI_STATE_SCHED_THREADED,	/* Napi is currently scheduled in threaded mode */
    };
    ```
    

  

- ==`static void ice_vsi_rx_napi_schedule(struct ice_vsi *vsi)`==
    
    ```JavaScript
    /**
    
    - ice_vsi_rx_napi_schedule - Schedule napi on RX queues from VSI
    - @vsi: VSI to schedule napi on
    */
    static void ice_vsi_rx_napi_schedule(struct ice_vsi *vsi)
    {
    int i;
        
        ice_for_each_rxq(vsi, i) {
        struct ice_rx_ring *rx_ring = vsi->rx_rings[i];
        ```
         if (rx_ring->xsk_pool)
         	napi_schedule(&rx_ring->q_vector->napi);
        ```
        }
        }
    ```
    

- ice_xdp_setup_prog → ice_xdp → 에서 끊김

1. napi schedule 키워드로 찾다가 나온 함수, 정확한 호출 과정을 잘 모르겠다?
2. `ice_for_each_rxq(vsi, i)` : 모든 존재하는 rx queue에서 napi schedule
    1. virtaul function 마다 rx queue가 존재
3. `ice_rx_ring` : 존재하는 rx_ring을 담는 변수
    1. `struct xsk_buff_pool` *xsk_pool 라는 변수
    2. ice_rx_ring struct 속에 있다
        1. descriptor ring 관련
    3. XDP (express data path) socket 기능과 연관되어있다? zero copy에 쓰인다?
4. rx_ring 에 xsk_pool이 존재하면 `napi_schedule()`

  

- ==`____napi_schedule()`==
    
    ```JavaScript
    /* Called with irq disabled */
    static inline void ____napi_schedule(struct softnet_data *sd,
    				     struct napi_struct *napi)
    {
    	struct task_struct *thread;
    
    	lockdep_assert_irqs_disabled();
    
    	if (test_bit(NAPI_STATE_THREADED, &napi->state)) {
    		/* Paired with smp_mb__before_atomic() in
    		 * napi_enable()/dev_set_threaded().
    		 * Use READ_ONCE() to guarantee a complete
    		 * read on napi->thread. Only call
    		 * wake_up_process() when it's not NULL.
    		 */
    		thread = READ_ONCE(napi->thread);
    		if (thread) {
    			/* Avoid doing set_bit() if the thread is in
    			 * INTERRUPTIBLE state, cause napi_thread_wait()
    			 * makes sure to proceed with napi polling
    			 * if the thread is explicitly woken from here.
    			 */
    			if (READ_ONCE(thread->__state) != TASK_INTERRUPTIBLE)
    				set_bit(NAPI_STATE_SCHED_THREADED, &napi->state);
    			wake_up_process(thread);
    			return;
    		}
    	}
    
    	list_add_tail(&napi->poll_list, &sd->poll_list);
    	WRITE_ONCE(napi->list_owner, smp_processor_id());
    	/* If not called from net_rx_action()
    	 * we have to raise NET_RX_SOFTIRQ.
    	 */
    	if (!sd->in_net_rx_action)
    		__raise_softirq_irqoff(NET_RX_SOFTIRQ);
    }
    
    \#define lockdep_assert_irqs_disabled()					\
    do {									\
    	WARN_ON_ONCE(__lockdep_enabled && this_cpu_read(hardirqs_enabled)); \
    } while (0)
    ```
    

1. `lockdep_assert_irqs_disabled()` : NAPI 이후에는 irq 금지
2. NAPI의 상태가 thread이면서, NAPI의 thread가 존재할 경우 진행
    1. 이 함수에서만 napi polling이 깨워지기 위해서 한다고함
    2. napi_thread_wait() 이 다른곳에서 깨워지는 것을 막는다
3. NAPI의 thread가 interruptable이 아닌 경우
    
    1. NAPI를 실행한다. poll을 schedule 상태로 바꾼다
    2. poll list는 NAPI_STATE_SCHED bit를 관리하는 존재에 의해서만 관리
    3. bit를 킨 존재는, bit를 끌 수도 있다
    4. 왜 interruptable이 아닌 경우에 실행하는지 모르겠다. 반대야할것 같은데?
    
      
    

- ==`int ice_napi_poll(struct napi_struct *napi, int budget)`==
    
    ```JavaScript
    /**
     * ice_napi_poll - NAPI polling Rx/Tx cleanup routine
     * @napi: napi struct with our devices info in it
     * @budget: amount of work driver is allowed to do this pass, in packets
     *
     * This function will clean all queues associated with a q_vector.
     *
     * Returns the amount of work done
     */
    int ice_napi_poll(struct napi_struct *napi, int budget)
    {
    	struct ice_q_vector *q_vector =
    				container_of(napi, struct ice_q_vector, napi);
    	struct ice_tx_ring *tx_ring;
    	struct ice_rx_ring *rx_ring;
    	bool clean_complete = true;
    	int budget_per_ring;
    	int work_done = 0;
    
    	/* Since the actual Tx work is minimal, we can give the Tx a larger
    	 * budget and be more aggressive about cleaning up the Tx descriptors.
    	 */
    	ice_for_each_tx_ring(tx_ring, q_vector->tx) {
    		bool wd;
    
    		if (tx_ring->xsk_pool)
    			wd = ice_xmit_zc(tx_ring);
    		else if (ice_ring_is_xdp(tx_ring))
    			wd = true;
    		else
    			wd = ice_clean_tx_irq(tx_ring, budget);
    
    		if (!wd)
    			clean_complete = false;
    	}
    
    	/* Handle case where we are called by netpoll with a budget of 0 */
    	if (unlikely(budget <= 0))
    		return budget;
    
    	/* normally we have 1 Rx ring per q_vector */
    	if (unlikely(q_vector->num_ring_rx > 1))
    		/* We attempt to distribute budget to each Rx queue fairly, but
    		 * don't allow the budget to go below 1 because that would exit
    		 * polling early.
    		 */
    		budget_per_ring = max_t(int, budget / q_vector->num_ring_rx, 1);
    	else
    		/* Max of 1 Rx ring in this q_vector so give it the budget */
    		budget_per_ring = budget;
    
    	ice_for_each_rx_ring(rx_ring, q_vector->rx) {
    		int cleaned;
    
    		/* A dedicated path for zero-copy allows making a single
    		 * comparison in the irq context instead of many inside the
    		 * ice_clean_rx_irq function and makes the codebase cleaner.
    		 */
    		cleaned = rx_ring->xsk_pool ?
    			  ice_clean_rx_irq_zc(rx_ring, budget_per_ring) :
    			  ice_clean_rx_irq(rx_ring, budget_per_ring);
    		work_done += cleaned;
    		/* if we clean as many as budgeted, we must not be done */
    		if (cleaned >= budget_per_ring)
    			clean_complete = false;
    	}
    
    	/* If work not completed, return budget and polling will return */
    	if (!clean_complete) {
    		/* Set the writeback on ITR so partial completions of
    		 * cache-lines will still continue even if we're polling.
    		 */
    		ice_set_wb_on_itr(q_vector);
    		return budget;
    	}
    
    	/* Exit the polling mode, but don't re-enable interrupts if stack might
    	 * poll us due to busy-polling
    	 */
    	if (napi_complete_done(napi, work_done)) {
    		ice_net_dim(q_vector);
    		ice_enable_interrupt(q_vector);
    	} else {
    		ice_set_wb_on_itr(q_vector);
    	}
    
    	return min_t(int, work_done, budget - 1);
    }
    ```
    

1. TX ring 초기화하기 (clean)
    1. `wd = ice_xmit_zc(tx_ring);`
        
        - `ice_xmit_zc`
            
            ```JavaScript
            /**
             * ice_xmit_zc - take entries from XSK Tx ring and place them onto HW Tx ring
             * @xdp_ring: XDP ring to produce the HW Tx descriptors on
             *
             * Returns true if there is no more work that needs to be done, false otherwise
             */
            bool ice_xmit_zc(struct ice_tx_ring *xdp_ring)
            {
            	struct xdp_desc *descs = xdp_ring->xsk_pool->tx_descs;
            	u32 nb_pkts, nb_processed = 0;
            	unsigned int total_bytes = 0;
            	int budget;
            
            	ice_clean_xdp_irq_zc(xdp_ring);
            
            	budget = ICE_DESC_UNUSED(xdp_ring);
            	budget = min_t(u16, budget, ICE_RING_QUARTER(xdp_ring));
            
            	nb_pkts = xsk_tx_peek_release_desc_batch(xdp_ring->xsk_pool, budget);
            	if (!nb_pkts)
            		return true;
            
            	if (xdp_ring->next_to_use + nb_pkts >= xdp_ring->count) {
            		nb_processed = xdp_ring->count - xdp_ring->next_to_use;
            		ice_fill_tx_hw_ring(xdp_ring, descs, nb_processed, &total_bytes);
            		xdp_ring->next_to_use = 0;
            	}
            
            	ice_fill_tx_hw_ring(xdp_ring, &descs[nb_processed], nb_pkts - nb_processed,
            			    &total_bytes);
            
            	ice_set_rs_bit(xdp_ring);
            	ice_xdp_ring_update_tail(xdp_ring);
            	ice_update_tx_ring_stats(xdp_ring, nb_pkts, total_bytes);
            
            	if (xsk_uses_need_wakeup(xdp_ring->xsk_pool))
            		xsk_set_tx_need_wakeup(xdp_ring->xsk_pool);
            
            	return nb_pkts < budget;
            }
            ```
            
        
        1. xsk_pool 이 있을 경우, zero copy pkt tx 할 경우, kernel bypass에 사용된다?
        2. wd에 tx가 성공적이었는지 저장
    2. `ice_ring_is_xdp(tx_ring)` 일 때는 항상 true로 설정한다
        
        - `ice_ring_is_xdp()`
            
            ```JavaScript
            static inline bool ice_ring_is_xdp(struct ice_tx_ring *ring)
            {
            return !!(ring->flags & ICE_TX_FLAGS_RING_XDP);
            }
            ```
            
        
        1. xdp (express data path) 사용할 때
        2. 왜 여기는 true로 설정할까?
    3. `wd = ice_clean_tx_irq(tx_ring, budget);`
        
        - `ice_clean_tx_irq()`
            
            ```JavaScript
            /**
             * ice_clean_tx_irq - Reclaim resources after transmit completes
             * @tx_ring: Tx ring to clean
             * @napi_budget: Used to determine if we are in netpoll
             *
             * Returns true if there's any budget left (e.g. the clean is finished)
             */
            static bool ice_clean_tx_irq(struct ice_tx_ring *tx_ring, int napi_budget)
            {
            	unsigned int total_bytes = 0, total_pkts = 0;
            	unsigned int budget = ICE_DFLT_IRQ_WORK;
            	struct ice_vsi *vsi = tx_ring->vsi;
            	s16 i = tx_ring->next_to_clean;
            	struct ice_tx_desc *tx_desc;
            	struct ice_tx_buf *tx_buf;
            
            	/* get the bql data ready */
            	netdev_txq_bql_complete_prefetchw(txring_txq(tx_ring));
            
            	tx_buf = &tx_ring->tx_buf[i];
            	tx_desc = ICE_TX_DESC(tx_ring, i);
            	i -= tx_ring->count;
            
            	prefetch(&vsi->state);
            
            	do {
            		struct ice_tx_desc *eop_desc = tx_buf->next_to_watch;
            
            		/* if next_to_watch is not set then there is no work pending */
            		if (!eop_desc)
            			break;
            
            		/* follow the guidelines of other drivers */
            		prefetchw(&tx_buf->skb->users);
            
            		smp_rmb();	/* prevent any other reads prior to eop_desc */
            
            		ice_trace(clean_tx_irq, tx_ring, tx_desc, tx_buf);
            		/* if the descriptor isn't done, no work yet to do */
            		if (!(eop_desc->cmd_type_offset_bsz &
            		      cpu_to_le64(ICE_TX_DESC_DTYPE_DESC_DONE)))
            			break;
            
            		/* clear next_to_watch to prevent false hangs */
            		tx_buf->next_to_watch = NULL;
            
            		/* update the statistics for this packet */
            		total_bytes += tx_buf->bytecount;
            		total_pkts += tx_buf->gso_segs;
            
            		/* free the skb */
            		napi_consume_skb(tx_buf->skb, napi_budget);
            
            		/* unmap skb header data */
            		dma_unmap_single(tx_ring->dev,
            				 dma_unmap_addr(tx_buf, dma),
            				 dma_unmap_len(tx_buf, len),
            				 DMA_TO_DEVICE);
            
            		/* clear tx_buf data */
            		tx_buf->type = ICE_TX_BUF_EMPTY;
            		dma_unmap_len_set(tx_buf, len, 0);
            
            		/* unmap remaining buffers */
            		while (tx_desc != eop_desc) {
            			ice_trace(clean_tx_irq_unmap, tx_ring, tx_desc, tx_buf);
            			tx_buf++;
            			tx_desc++;
            			i++;
            			if (unlikely(!i)) {
            				i -= tx_ring->count;
            				tx_buf = tx_ring->tx_buf;
            				tx_desc = ICE_TX_DESC(tx_ring, 0);
            			}
            
            			/* unmap any remaining paged data */
            			if (dma_unmap_len(tx_buf, len)) {
            				dma_unmap_page(tx_ring->dev,
            					       dma_unmap_addr(tx_buf, dma),
            					       dma_unmap_len(tx_buf, len),
            					       DMA_TO_DEVICE);
            				dma_unmap_len_set(tx_buf, len, 0);
            			}
            		}
            		ice_trace(clean_tx_irq_unmap_eop, tx_ring, tx_desc, tx_buf);
            
            		/* move us one more past the eop_desc for start of next pkt */
            		tx_buf++;
            		tx_desc++;
            		i++;
            		if (unlikely(!i)) {
            			i -= tx_ring->count;
            			tx_buf = tx_ring->tx_buf;
            			tx_desc = ICE_TX_DESC(tx_ring, 0);
            		}
            
            		prefetch(tx_desc);
            
            		/* update budget accounting */
            		budget--;
            	} while (likely(budget));
            
            	i += tx_ring->count;
            	tx_ring->next_to_clean = i;
            
            	ice_update_tx_ring_stats(tx_ring, total_pkts, total_bytes);
            	netdev_tx_completed_queue(txring_txq(tx_ring), total_pkts, total_bytes);
            
            \#define TX_WAKE_THRESHOLD ((s16)(DESC_NEEDED * 2))
            	if (unlikely(total_pkts && netif_carrier_ok(tx_ring->netdev) &&
            		     (ICE_DESC_UNUSED(tx_ring) >= TX_WAKE_THRESHOLD))) {
            		/* Make sure that anybody stopping the queue after this
            		 * sees the new next_to_clean.
            		 */
            		smp_mb();
            		if (netif_tx_queue_stopped(txring_txq(tx_ring)) &&
            		    !test_bit(ICE_VSI_DOWN, vsi->state)) {
            			netif_tx_wake_queue(txring_txq(tx_ring));
            			++tx_ring->ring_stats->tx_stats.restart_q;
            		}
            	}
            
            	return !!budget;
            }
            ```
            
        
        1. default tx 초기 과정이다
        2. descriptor를 free해준다
2. budget_per_ring = max_t(int, budget / q_vector->num_ring_rx, 1);
    1. budget은 rx_ring 개수로 나눠서 공평하게 배분한다
    2. 최소 1 은 받게 설정
3. clean rx ring
    1. clean 된 irq의 개수를 budget과 비교한다
    2. 만약 전자가 더 많으면 uncomplete된 상태이므로? 계속 polling을 한다
    3. clean 된 개수와 budget을 대소 비교로 complete 여부를 알수있나?
4. if not complete, return the budget
5. exit polling
    - `static void ice_net_dim(struct ice_q_vector *q_vector)`
        
        ```JavaScript
        /**
         * ice_net_dim - Update net DIM algorithm
         * @q_vector: the vector associated with the interrupt
         *
         * Create a DIM sample and notify net_dim() so that it can possibly decide
         * a new ITR value based on incoming packets, bytes, and interrupts.
         *
         * This function is a no-op if the ring is not configured to dynamic ITR.
         */
        static void ice_net_dim(struct ice_q_vector *q_vector)
        {
        	struct ice_ring_container *tx = &q_vector->tx;
        	struct ice_ring_container *rx = &q_vector->rx;
        
        	if (ITR_IS_DYNAMIC(tx)) {
        		struct dim_sample dim_sample;
        
        		__ice_update_sample(q_vector, tx, &dim_sample, true);
        		net_dim(&tx->dim, dim_sample);
        	}
        
        	if (ITR_IS_DYNAMIC(rx)) {
        		struct dim_sample dim_sample;
        
        		__ice_update_sample(q_vector, rx, &dim_sample, false);
        		net_dim(&rx->dim, dim_sample);
        	}
        }
        ```
        
    - `static void ice_enable_interrupt(struct ice_q_vector *q_vector)`
        
        ```JavaScript
        /**
         * ice_enable_interrupt - re-enable MSI-X interrupt
         * @q_vector: the vector associated with the interrupt to enable
         *
         * If the VSI is down, the interrupt will not be re-enabled. Also,
         * when enabling the interrupt always reset the wb_on_itr to false
         * and trigger a software interrupt to clean out internal state.
         */
        static void ice_enable_interrupt(struct ice_q_vector *q_vector)
        {
        	struct ice_vsi *vsi = q_vector->vsi;
        	bool wb_en = q_vector->wb_on_itr;
        	u32 itr_val;
        
        	if (test_bit(ICE_DOWN, vsi->state))
        		return;
        
        	/* trigger an ITR delayed software interrupt when exiting busy poll, to
        	 * make sure to catch any pending cleanups that might have been missed
        	 * due to interrupt state transition. If busy poll or poll isn't
        	 * enabled, then don't update ITR, and just enable the interrupt.
        	 */
        	if (!wb_en) {
        		itr_val = ice_buildreg_itr(ICE_ITR_NONE, 0);
        	} else {
        		q_vector->wb_on_itr = false;
        
        		/* do two things here with a single write. Set up the third ITR
        		 * index to be used for software interrupt moderation, and then
        		 * trigger a software interrupt with a rate limit of 20K on
        		 * software interrupts, this will help avoid high interrupt
        		 * loads due to frequently polling and exiting polling.
        		 */
        		itr_val = ice_buildreg_itr(ICE_IDX_ITR2, ICE_ITR_20K);
        		itr_val |= GLINT_DYN_CTL_SWINT_TRIG_M |
        			   ICE_IDX_ITR2 << GLINT_DYN_CTL_SW_ITR_INDX_S |
        			   GLINT_DYN_CTL_SW_ITR_INDX_ENA_M;
        	}
        	wr32(&vsi->back->hw, GLINT_DYN_CTL(q_vector->reg_idx), itr_val);
        }
        ```
        

  

- `static void ice_napi_enable_all(struct ice_vsi *vsi)`
    
    ```JavaScript
    /**
     * ice_napi_enable_all - Enable NAPI for all q_vectors in the VSI
     * @vsi: the VSI being configured
     */
    static void ice_napi_enable_all(struct ice_vsi *vsi)
    {
    	int q_idx;
    
    	if (!vsi->netdev)
    		return;
    
    	ice_for_each_q_vector(vsi, q_idx) {
    		struct ice_q_vector *q_vector = vsi->q_vectors[q_idx];
    
    		ice_init_moderation(q_vector);
    
    		if (q_vector->rx.rx_ring || q_vector->tx.tx_ring)
    			napi_enable(&q_vector->napi);
    	}
    }
    ```
    

- napi_struct 의 state를 바꿔준다

- `static void ice_napi_disable_all(struct ice_vsi *vsi)`
    
    ```JavaScript
    /**
     * ice_napi_disable_all - Disable NAPI for all q_vectors in the VSI
     * @vsi: VSI having NAPI disabled
     */
    static void ice_napi_disable_all(struct ice_vsi *vsi)
    {
    	int q_idx;
    
    	if (!vsi->netdev)
    		return;
    
    	ice_for_each_q_vector(vsi, q_idx) {
    		struct ice_q_vector *q_vector = vsi->q_vectors[q_idx];
    
    		if (q_vector->rx.rx_ring || q_vector->tx.tx_ring)
    			napi_disable(&q_vector->napi);
    
    		cancel_work_sync(&q_vector->tx.dim.work);
    		cancel_work_sync(&q_vector->rx.dim.work);
    	}
    }
    ```
    

- napi_struct 의 state 바꾸고 timer 초기화

- `static void ice_vsi_rx_napi_schedule(struct ice_vsi *vsi)`
    
    ```JavaScript
    /**
     * ice_vsi_rx_napi_schedule - Schedule napi on RX queues from VSI
     * @vsi: VSI to schedule napi on
     */
    static void ice_vsi_rx_napi_schedule(struct ice_vsi *vsi)
    {
    	int i;
    
    	ice_for_each_rxq(vsi, i) {
    		struct ice_rx_ring *rx_ring = vsi->rx_rings[i];
    
    		if (rx_ring->xsk_pool)
    			napi_schedule(&rx_ring->q_vector->napi);
    	}
    }
    ```
    

- vsi 를 for문으로 돌면서 전부 napi schedule 해준다

- `static void ice_napi_add(struct ice_vsi *vsi)`
    
    ```JavaScript
    /**
     * ice_napi_add - register NAPI handler for the VSI
     * @vsi: VSI for which NAPI handler is to be registered
     *
     * This function is only called in the driver's load path. Registering the NAPI
     * handler is done in ice_vsi_alloc_q_vector() for all other cases (i.e. resume,
     * reset/rebuild, etc.)
     */
    static void ice_napi_add(struct ice_vsi *vsi)
    {
    	int v_idx;
    
    	if (!vsi->netdev)
    		return;
    
    	ice_for_each_q_vector(vsi, v_idx) {
    		netif_napi_add(vsi->netdev, &vsi->q_vectors[v_idx]->napi,
    			       ice_napi_poll);
    		__ice_q_vector_set_napi_queues(vsi->q_vectors[v_idx], false);
    	}
    }
    ```
    

- napi handler를 등록한다 (vsi를 위해서)
- driver load path에서만 call 된다
    - resume, reset, rebuild의 경우에는 ice_vsi_alloc_q_vector() 사용???

  

  

IRQ

- 3가지 종류의 IRQ (msi-x, msi, legacy)

SoftIrq

- interrupt handler 바깥에서 interrupt를 실행시킬 수 있는 방법이 있다
    - system 상에서 hw interrupt이 아예 막히는 상황이 있다. 이런 경우를 위해 softirq가 사용된다??
- 무작위로 victim을 정해 그것의 context에서 실행된다
    - 다른 process에 할당된 resource를 사용한다는 내용인듯?
- priority가 높아서 거의 항상 먼저 실행된다 (hw interrupt 제외하고)
- 10가지 종류의 sw interrupt vector
    - 2 tasklet, 2 network, 2 block layer, 2 timer, 1 scheduler
    - 1 read-copy-update processing
- kernel은 cpu 마다 어떤 softirq를 처리해야하는지 표시된 bitmask가 존재
- 현재 실행되는 thread를 멈추고 실행되는 2가지 경우
    - hw interrupt이 끝난 후 (끝나고 sw irq를 실행하는 경우가 많다)
    - kernel이 swirq processing을 다시 실행할 경우 (이 때 victim이 발생)
- ksoftirqd는 10회 soft irq를 process하고도 남아있으면 wake 된다
    - cpu 마다 1개의 thread가 실행
    - wake 된 이후 scheduling 이 되면 실행된다
    - Ksoftirqd will also be **poked** if a softirq is raised outside of (hardware or software) interrupt context??
        - otherwise, an arbitrary amount of time might pass before softirqs are processed again??  
              
              
            

  

- `static int ice_req_irq_msix_misc(struct ice_pf *pf)`
    
    ```JavaScript
    /**
     * ice_req_irq_msix_misc - Setup the misc vector to handle non queue events
     * @pf: board private structure
     *
     * This sets up the handler for MSIX 0, which is used to manage the
     * non-queue interrupts, e.g. AdminQ and errors. This is not used
     * when in MSI or Legacy interrupt mode.
     */
    static int ice_req_irq_msix_misc(struct ice_pf *pf)
    {
    	struct device *dev = ice_pf_to_dev(pf);
    	struct ice_hw *hw = &pf->hw;
    	u32 pf_intr_start_offset;
    	struct msi_map irq;
    	int err = 0;
    
    	if (!pf->int_name[0])
    		snprintf(pf->int_name, sizeof(pf->int_name) - 1, "%s-%s:misc",
    			 dev_driver_string(dev), dev_name(dev));
    
    	if (!pf->int_name_ll_ts[0])
    		snprintf(pf->int_name_ll_ts, sizeof(pf->int_name_ll_ts) - 1,
    			 "%s-%s:ll_ts", dev_driver_string(dev), dev_name(dev));
    	/* Do not request IRQ but do enable OICR interrupt since settings are
    	 * lost during reset. Note that this function is called only during
    	 * rebuild path and not while reset is in progress.
    	 */
    	if (ice_is_reset_in_progress(pf->state))
    		goto skip_req_irq;
    
    	/* reserve one vector in irq_tracker for misc interrupts */
    	irq = ice_alloc_irq(pf, false);
    	if (irq.index < 0)
    		return irq.index;
    
    	pf->oicr_irq = irq;
    	err = devm_request_threaded_irq(dev, pf->oicr_irq.virq, ice_misc_intr,
    					ice_misc_intr_thread_fn, 0,
    					pf->int_name, pf);
    	if (err) {
    		dev_err(dev, "devm_request_threaded_irq for %s failed: %d\n",
    			pf->int_name, err);
    		ice_free_irq(pf, pf->oicr_irq);
    		return err;
    	}
    
    	/* reserve one vector in irq_tracker for ll_ts interrupt */
    	if (!pf->hw.dev_caps.ts_dev_info.ts_ll_int_read)
    		goto skip_req_irq;
    
    	irq = ice_alloc_irq(pf, false);
    	if (irq.index < 0)
    		return irq.index;
    
    	pf->ll_ts_irq = irq;
    	err = devm_request_irq(dev, pf->ll_ts_irq.virq, ice_ll_ts_intr, 0,
    			       pf->int_name_ll_ts, pf);
    	if (err) {
    		dev_err(dev, "devm_request_irq for %s failed: %d\n",
    			pf->int_name_ll_ts, err);
    		ice_free_irq(pf, pf->ll_ts_irq);
    		return err;
    	}
    
    skip_req_irq:
    	ice_ena_misc_vector(pf);
    
    	ice_ena_ctrlq_interrupts(hw, pf->oicr_irq.index);
    	/* This enables LL TS interrupt */
    	pf_intr_start_offset = rd32(hw, PFINT_ALLOC) & PFINT_ALLOC_FIRST;
    	if (pf->hw.dev_caps.ts_dev_info.ts_ll_int_read)
    		wr32(hw, PFINT_SB_CTL,
    		     ((pf->ll_ts_irq.index + pf_intr_start_offset) &
    		      PFINT_SB_CTL_MSIX_INDX_M) | PFINT_SB_CTL_CAUSE_ENA_M);
    	wr32(hw, GLINT_ITR(ICE_RX_ITR, pf->oicr_irq.index),
    	     ITR_REG_ALIGN(ICE_ITR_8K) >> ICE_ITR_GRAN_S);
    
    	ice_flush(hw);
    	ice_irq_dynamic_ena(hw, NULL, NULL);
    
    	return 0;
    }
    ```
    
- `struct msi_map ice_alloc_irq(struct ice_pf *pf, bool dyn_only)`
    
    ```JavaScript
    struct msi_map ice_alloc_irq(struct ice_pf *pf, bool dyn_only)
    {
    	int sriov_base_vector = pf->sriov_base_vector;
    	struct msi_map map = { .index = -ENOENT };
    	struct device *dev = ice_pf_to_dev(pf);
    	struct ice_irq_entry *entry;
    
    	entry = ice_get_irq_res(pf, dyn_only);
    	if (!entry)
    		return map;
    
    	/* fail if we're about to violate SRIOV vectors space */
    	if (sriov_base_vector && entry->index >= sriov_base_vector)
    		goto exit_free_res;
    
    	if (pci_msix_can_alloc_dyn(pf->pdev) && entry->dynamic) {
    		map = pci_msix_alloc_irq_at(pf->pdev, entry->index, NULL);
    		if (map.index < 0)
    			goto exit_free_res;
    		dev_dbg(dev, "allocated new irq at index %d\n", map.index);
    	} else {
    		map.index = entry->index;
    		map.virq = pci_irq_vector(pf->pdev, map.index);
    	}
    
    	return map;
    
    exit_free_res:
    	dev_err(dev, "Could not allocate irq at idx %d\n", entry->index);
    	ice_free_irq_res(pf, entry->index);
    	return map;
    }
    ```
    
- `struct ice_irq_entry *ice_get_irq_res(struct ice_pf *pf, bool dyn_only)`
    
    ```JavaScript
    /**
     * ice_get_irq_res - get an interrupt resource
     * @pf: board private structure
     * @dyn_only: force entry to be dynamically allocated
     *
     * Allocate new irq entry in the free slot of the tracker. Since xarray
     * is used, always allocate new entry at the lowest possible index. Set
     * proper allocation limit for maximum tracker entries.
     *
     * Returns allocated irq entry or NULL on failure.
     */
    static struct ice_irq_entry *ice_get_irq_res(struct ice_pf *pf, bool dyn_only)
    {
    	struct xa_limit limit = { .max = pf->irq_tracker.num_entries,
    				  .min = 0 };
    	unsigned int num_static = pf->irq_tracker.num_static;
    	struct ice_irq_entry *entry;
    	unsigned int index;
    	int ret;
    
    	entry = kzalloc(sizeof(*entry), GFP_KERNEL);
    	if (!entry)
    		return NULL;
    
    	/* skip preallocated entries if the caller says so */
    	if (dyn_only)
    		limit.min = num_static;
    
    	ret = xa_alloc(&pf->irq_tracker.entries, &index, entry, limit,
    		       GFP_KERNEL);
    
    	if (ret) {
    		kfree(entry);
    		entry = NULL;
    	} else {
    		entry->index = index;
    		entry->dynamic = index >= num_static;
    	}
    
    	return entry;
    }
    ```
    

  

  

  

  

![[황재훈/Research Intern/pastNotion/intel ice/NAPI/img/Untitled 1.png|Untitled 1.png]]

![[황재훈/Research Intern/pastNotion/intel ice/NAPI/img/Untitled 2.png|Untitled 2.png]]