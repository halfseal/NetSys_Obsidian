[https://d2.naver.com/helloworld/47667](https://d2.naver.com/helloworld/47667)

[[struct device]]

~/linux-6.9/drivers/net/ethernet/intel/ice/ice_main.c

[[ice_probe() ⇒ 실질적으로 처음 카드를 인식하는 단계]]

ice_open() ⇒ NIC가 활성화 되었을 때 네트워크 작동을 위해 초기화 하는 단계

→ice_open_internal()실행

→ice_vsi_open(vsi) 실행

vsi : virtual station interface, 가상화를 통해 여러 개의 논리 NIC로 사용할 수 있게 한다.

device라는 struct는 linux가 device를 관리하는데 사용함. “devm_” 은 device 메모리 관리 인터페이스를 통하므로, driver가 unload 되면 모든 자원을 해제하여 explicit하게 free를 할 필요가 없다는 장점이 있다.

[[→ice_vsi_setup_tx_rings(vsi)]]

[[→ice_vsi_setup_rx_rings(vsi)]]

[[→ ice_vsi_req_irq_msix(vsi, int_name)]]

==ice_vsi_req_msix()에서 devm_request_irq()를 실행하여 NIC의 IRQ를 등록하게 된다. 이때 이것은 커널의 전역변수인 irq_desc[]에 등록되게 되어 do_IRQ(할당받은 irq 번호)로 호출 될 때 여기서 찾게 된다.==

→ice_set_cpu_rx_rmap() ==(drivers/net/ethernet/intel/ice/ice_arfs.c) — 각각의 queue에 대하여 aRFS를 위해 매핑하는 함수.==

→ ice_vsi_start_all_rx_rings() → ice_vsi_ctrl_all_rx_rings() → 각각의 하나씩의 rx ring에 대하여 시작하고, 정상적으로 시작되는 것을 확인하고 함수 반환

→ ice_up_complete() 최종적으로 다 열렸는지 확인함.

→ice_vsi_start_all_rx_rings()

→ice_napi_enable_all()

→ice_vsi_ena_irq()

각각의 q_vector에 대하여

→ice_irq_dynamic_ena() → 인터럽트를 활성화 함.

![[Untitled 3.png|Untitled 3.png]]

→netif_tx_start_all_queues()

→netif_carrier_on()

---

napi_shcedule() 이후

NET_RX_SOFTIRQ는 enum으로, 3이라는 값을 가진다. 참고로 TX는 2이다.

1UL<<3으로, 4번째 비트를 켜게 된다. 이를 통해 CPU가 해당 값을 알아차리고\

jiffies를 사용하여 sotfIRQ의 timeout을 구현하게 된다. 이 때 최대 값은 MAX_SOFTIRQ_TIME이며, 이를 더하여 할당된 자원을 알 수 있게 된다.

- net_rx_action이 매핑되는 과정
    
    subsys_initcall(net_dev_init) ==(net/core/dev.c), 시스템 부팅시 일찍이 실행되게 하는 함수==
    
    - net_dev_init(void)
        - 코드
            
            ```C
            /*
             *       This is called single threaded during boot, so no need
             *       to take the rtnl semaphore.
             */
            static int __init net_dev_init(void)
            {
            	int i, rc = -ENOMEM;
            
            	BUG_ON(!dev_boot_phase);
            
            	net_dev_struct_check();
            
            	if (dev_proc_init())
            		goto out;
            
            	if (netdev_kobject_init())
            		goto out;
            
            	for (i = 0; i < PTYPE_HASH_SIZE; i++)
            		INIT_LIST_HEAD(&ptype_base[i]);
            
            	if (register_pernet_subsys(&netdev_net_ops))
            		goto out;
            
            	/*
            	 *	Initialise the packet receive queues.
            	 */
            
            	for_each_possible_cpu(i) {
            		struct work_struct *flush = per_cpu_ptr(&flush_works, i);
            		struct softnet_data *sd = &per_cpu(softnet_data, i);
            
            		INIT_WORK(flush, flush_backlog);
            
            		skb_queue_head_init(&sd->input_pkt_queue);
            		skb_queue_head_init(&sd->process_queue);
            \#ifdef CONFIG_XFRM_OFFLOAD
            		skb_queue_head_init(&sd->xfrm_backlog);
            \#endif
            		INIT_LIST_HEAD(&sd->poll_list);
            		sd->output_queue_tailp = &sd->output_queue;
            \#ifdef CONFIG_RPS
            		INIT_CSD(&sd->csd, rps_trigger_softirq, sd);
            		sd->cpu = i;
            \#endif
            		INIT_CSD(&sd->defer_csd, trigger_rx_softirq, sd);
            		spin_lock_init(&sd->defer_lock);
            
            		init_gro_hash(&sd->backlog);
            		sd->backlog.poll = process_backlog;
            		sd->backlog.weight = weight_p;
            
            		if (net_page_pool_create(i))
            			goto out;
            	}
            
            	dev_boot_phase = 0;
            
            	/* The loopback device is special if any other network devices
            	 * is present in a network namespace the loopback device must
            	 * be present. Since we now dynamically allocate and free the
            	 * loopback device ensure this invariant is maintained by
            	 * keeping the loopback device as the first device on the
            	 * list of network devices.  Ensuring the loopback devices
            	 * is the first device that appears and the last network device
            	 * that disappears.
            	 */
            	if (register_pernet_device(&loopback_net_ops))
            		goto out;
            
            	if (register_pernet_device(&default_device_ops))
            		goto out;
            
            	open_softirq(NET_TX_SOFTIRQ, net_tx_action);
            	open_softirq(NET_RX_SOFTIRQ, net_rx_action); //여기서 softirq array에 해당하는 함수인 net_rx_action 매핑이 이루어지게 됨.
            
            	rc = cpuhp_setup_state_nocalls(CPUHP_NET_DEV_DEAD, "net/dev:dead",
            				       NULL, dev_cpu_dead);
            	WARN_ON(rc < 0);
            	rc = 0;
            out:
            	if (rc < 0) {
            		for_each_possible_cpu(i) {
            			struct page_pool *pp_ptr;
            
            			pp_ptr = per_cpu(system_page_pool, i);
            			if (!pp_ptr)
            				continue;
            
            			page_pool_destroy(pp_ptr);
            			per_cpu(system_page_pool, i) = NULL;
            		}
            	}
            
            	return rc;
            }
            ```
            
- __do_softirq(void) (kernel/softirq.c)
    ---
    - 코드
        
        ```C
        asmlinkage __visible void __softirq_entry __do_softirq(void)
        {
        	handle_softirqs(false);
        }
        ```
        
    - handle_softirqs(false)
        
        ---
        
        - 코드
            
            ```C
            static void handle_softirqs(bool ksirqd)
            {
            	unsigned long end = jiffies + MAX_SOFTIRQ_TIME;
            	unsigned long old_flags = current->flags;
            	int max_restart = MAX_SOFTIRQ_RESTART;
            	struct softirq_action *h;
            	bool in_hardirq;
            	__u32 pending;
            	int softirq_bit;
            
            	/*
            	 * Mask out PF_MEMALLOC as the current task context is borrowed for the
            	 * softirq. A softirq handled, such as network RX, might set PF_MEMALLOC
            	 * again if the socket is related to swapping.
            	 */
            	current->flags &= ~PF_MEMALLOC;
            
            	pending = local_softirq_pending(); //softIRQ에 대한 모든 bitmap을 불러옴
            
            	softirq_handle_begin();
            	in_hardirq = lockdep_softirq_start();
            	account_softirq_enter(current);
            
            restart:
            	/* Reset the pending bitmask before enabling irqs */
            	set_softirq_pending(0);
            
            	local_irq_enable();
            
            	h = softirq_vec;
            
            	while ((softirq_bit = ffs(pending))) { //ffs : set 된 것 중 가장 낮은 거를 찾음. 즉 set된 모든 비트가 0이 될 때까지 아래의 코드를 반복함.
            		unsigned int vec_nr;
            		int prev_count;
            
            		h += softirq_bit - 1;
            
            		vec_nr = h - softirq_vec;
            		prev_count = preempt_count();
            
            		kstat_incr_softirqs_this_cpu(vec_nr);
            
            		trace_softirq_entry(vec_nr);
            		h->action(h);                             //실제 softIRQ가 실행되는 부분
            		trace_softirq_exit(vec_nr);
            		if (unlikely(prev_count != preempt_count())) {
            			pr_err("huh, entered softirq %u %s %p with preempt_count %08x, exited with %08x?\n",
            			       vec_nr, softirq_to_name[vec_nr], h->action,
            			       prev_count, preempt_count());
            			preempt_count_set(prev_count);
            		}
            		h++;
            		pending >>= softirq_bit;
            	}
            
            	if (!IS_ENABLED(CONFIG_PREEMPT_RT) && ksirqd)
            		rcu_softirq_qs();
            
            	local_irq_disable();
            
            	pending = local_softirq_pending();
            	if (pending) {
            		if (time_before(jiffies, end) && !need_resched() &&
            		    --max_restart)
            			goto restart;
            
            		wakeup_softirqd();
            	}
            
            	account_softirq_exit(current);
            	lockdep_softirq_end(in_hardirq);
            	softirq_handle_end();
            	current_restore_flags(old_flags, PF_MEMALLOC);
            }
            ```
            
        
        > h→action(h)에서 함수포인터를 통해 미리 매핑되어있던 net_rx_action이 실행되게 됨.
        
        - net_rx_action(struct softirq_action *h) ==(net/core/dev.c)==
            
            ---
            
            - 코드
                
                ```C
                static __latent_entropy void net_rx_action(struct softirq_action *h)
                {
                	struct softnet_data *sd = this_cpu_ptr(&softnet_data);
                	unsigned long time_limit = jiffies +
                		usecs_to_jiffies(READ_ONCE(net_hotdata.netdev_budget_usecs));
                	int budget = READ_ONCE(net_hotdata.netdev_budget);
                	LIST_HEAD(list);
                	LIST_HEAD(repoll);
                
                start:
                	sd->in_net_rx_action = true;
                	local_irq_disable();
                	list_splice_init(&sd->poll_list, &list);
                	local_irq_enable();
                
                	for (;;) {
                		struct napi_struct *n;
                
                		skb_defer_free_flush(sd);
                
                		if (list_empty(&list)) {
                			if (list_empty(&repoll)) {
                				sd->in_net_rx_action = false;
                				barrier();
                				/* We need to check if ____napi_schedule()
                				 * had refilled poll_list while
                				 * sd->in_net_rx_action was true.
                				 */
                				if (!list_empty(&sd->poll_list))
                					goto start;
                				if (!sd_has_rps_ipi_waiting(sd))
                					goto end;
                			}
                			break;
                		}
                
                		n = list_first_entry(&list, struct napi_struct, poll_list);
                		budget -= napi_poll(n, &repoll);  //napi_poll이 이루어지는 부분
                
                		/* If softirq window is exhausted then punt.
                		 * Allow this to run for 2 jiffies since which will allow
                		 * an average latency of 1.5/HZ.
                		 */
                		if (unlikely(budget <= 0 ||
                			     time_after_eq(jiffies, time_limit))) {
                			sd->time_squeeze++;
                			break;
                		}
                	}
                
                	local_irq_disable();
                
                	list_splice_tail_init(&sd->poll_list, &list);
                	list_splice_tail(&repoll, &list);
                	list_splice(&list, &sd->poll_list);
                	if (!list_empty(&sd->poll_list))
                		__raise_softirq_irqoff(NET_RX_SOFTIRQ);
                	else
                		sd->in_net_rx_action = false;
                
                	net_rps_action_and_irq_enable(sd);
                end:;
                }
                ```
                
            
            > jiffies를 통해 time out을 구현하였고, softnet_data 구조체에서 poll_list를 가져왔다. 이후 이러한 리스트를 바탕으로 napi_poll() 함수를 호출하여 폴링을 시작하게 된다. 이 때, budget에서
            
            - napi_poll()
                
                ---
                
                - 코드
                    
                    ```C
                    static int napi_poll(struct napi_struct *n, struct list_head *repoll)
                    {
                    	bool do_repoll = false;
                    	void *have;
                    	int work;
                    
                    	list_del_init(&n->poll_list);
                    
                    	have = netpoll_poll_lock(n);
                    
                    	work = __napi_poll(n, &do_repoll);
                    
                    	if (do_repoll)
                    		list_add_tail(&n->poll_list, repoll);
                    
                    	netpoll_poll_unlock(have);
                    
                    	return work;
                    }
                    ```
                    
                
                > poll_list를 지우고, netpoll_poll_lock을 걸게 된다.(멀티 코어 환경 대응) 이후 __napi_poll() 함수를 실행시켜서 폴링을 시작하게 된다. 또한 추가적으로 다시 폴링이 필요하다면 list_add_tail을 통해 repoll변수에 저장된 리스트를 가져오게 된다.
                
                - __napi_poll(struct napi_struct *n, bool *repoll)
                    
                    ---
                    
                    - 코드
                        
                        ```C
                        static int __napi_poll(struct napi_struct *n, bool *repoll)
                        {
                        	int work, weight;
                        
                        	weight = n->weight;
                        
                        	/* This NAPI_STATE_SCHED test is for avoiding a race
                        	 * with netpoll's poll_napi().  Only the entity which
                        	 * obtains the lock and sees NAPI_STATE_SCHED set will
                        	 * actually make the ->poll() call.  Therefore we avoid
                        	 * accidentally calling ->poll() when NAPI is not scheduled.
                        	 */
                        	work = 0;
                        	if (napi_is_scheduled(n)) {
                        		work = n->poll(n, weight);
                        		trace_napi_poll(n, work, weight);
                        
                        		xdp_do_check_flushed(n);
                        	}
                        
                        	if (unlikely(work > weight))
                        		netdev_err_once(n->dev, "NAPI poll function %pS returned %d, exceeding its budget of %d.\n",
                        				n->poll, work, weight);
                        
                        	if (likely(work < weight))
                        		return work;
                        
                        	/* Drivers must not modify the NAPI state if they
                        	 * consume the entire weight.  In such cases this code
                        	 * still "owns" the NAPI instance and therefore can
                        	 * move the instance around on the list at-will.
                        	 */
                        	if (unlikely(napi_disable_pending(n))) {
                        		napi_complete(n);
                        		return work;
                        	}
                        
                        	/* The NAPI context has more processing work, but busy-polling
                        	 * is preferred. Exit early.
                        	 */
                        	if (napi_prefer_busy_poll(n)) {
                        		if (napi_complete_done(n, work)) {
                        			/* If timeout is not set, we need to make sure
                        			 * that the NAPI is re-scheduled.
                        			 */
                        			napi_schedule(n);
                        		}
                        		return work;
                        	}
                        
                        	if (n->gro_bitmask) {
                        		/* flush too old packets
                        		 * If HZ < 1000, flush all packets.
                        		 */
                        		napi_gro_flush(n, HZ >= 1000);
                        	}
                        
                        	gro_normal_list(n);
                        
                        	/* Some drivers may have called napi_schedule
                        	 * prior to exhausting their budget.
                        	 */
                        	if (unlikely(!list_empty(&n->poll_list))) {
                        		pr_warn_once("%s: Budget exhausted after napi rescheduled\n",
                        			     n->dev ? n->dev->name : "backlog");
                        		return work;
                        	}
                        
                        	*repoll = true;
                        
                        	return work;
                        }
                        ```
                        
                    
                    > 반드시 NAPI_STATE_SCHED라는 상태여야지만 →poll()을 호출 할 수 있도록 하였다. 이는 netpoll의 poll_napi()와 서로 경쟁하지 않게 하기 위함이다. poll은 napi_struct 구조체의 함수포인터로 실행되게 된다. 이 napi_struct는 softnet_data로부터 비롯되는데, 이는 CPU당 부여되는 구조체이다.  
                    > 따라서 n→poll()함수를 통해 ice_napi_poll이 실행되게 된다.  
                    > weight은 budget을 의미한다.  
                    
                    [[ice_probe() ⇒ 실질적으로 처음 카드를 인식하는 단계]] [[ice_probe() ⇒ 실질적으로 처음 카드를 인식하는 단계]]
                    
                    - ice_napi_poll(struct napi_struct *napi, int budget) ==(ice/ice_txrx.c)==
                        
                        ---
                        
                        - 코드
                            
                            ```C
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
                            
                        
                        > napi_poll은 Rx와 Tx를 동시에 다루게 된다. 모든 tx링에 대하여 ice_xmit_zc()혹은 ice_clean_tx_irq()등의 함수로 다루게 되는데 일단 bottom up path이므로 추후에 살펴보고자 한다.  
                        > 각각의 rx_ring에 대하여 xsk_pool이 있다면 zc옵션이 있는 함수인 ice_clean_rx_irq_zc, 아니라면 ice_clean_rx_irq를 실행하게 된다.  
                        >   
                        > 특히 num_ring_rx를 확인하는 부분에서, 보통은 rx_ring이 1개임을 알 수 있다. unlikely로 if문에 들어가 있기 때문이다. 또한 이러한 경우, 각각의 rx_ring에 budget을 골고루 분배해야하기 때문에 budget_per_ring이 budget을 rx_ring의 갯수로 나눠준 값이 된다.  
                        
                        - ice_clean_rx_irq_zc(rx_ring, budget_per_ring) ==(ice/ice_xsk.c)==
                            
                            ---
                            
                            - 코드
                                
                                ```C
                                /**
                                 * ice_clean_rx_irq_zc - consumes packets from the hardware ring
                                 * @rx_ring: AF_XDP Rx ring
                                 * @budget: NAPI budget
                                 *
                                 * Returns number of processed packets on success, remaining budget on failure.
                                 */
                                int ice_clean_rx_irq_zc(struct ice_rx_ring *rx_ring, int budget)
                                {
                                	unsigned int total_rx_bytes = 0, total_rx_packets = 0;
                                	struct xsk_buff_pool *xsk_pool = rx_ring->xsk_pool;
                                	u32 ntc = rx_ring->next_to_clean;
                                	u32 ntu = rx_ring->next_to_use;
                                	struct xdp_buff *first = NULL;
                                	struct ice_tx_ring *xdp_ring;
                                	unsigned int xdp_xmit = 0;
                                	struct bpf_prog *xdp_prog;
                                	u32 cnt = rx_ring->count;
                                	bool failure = false;
                                	int entries_to_alloc;
                                
                                	/* ZC patch is enabled only when XDP program is set,
                                	 * so here it can not be NULL
                                	 */
                                	xdp_prog = READ_ONCE(rx_ring->xdp_prog);
                                	xdp_ring = rx_ring->xdp_ring;
                                
                                	if (ntc != rx_ring->first_desc)
                                		first = *ice_xdp_buf(rx_ring, rx_ring->first_desc);
                                
                                	while (likely(total_rx_packets < (unsigned int)budget)) {
                                		union ice_32b_rx_flex_desc *rx_desc;
                                		unsigned int size, xdp_res = 0;
                                		struct xdp_buff *xdp;
                                		struct sk_buff *skb;
                                		u16 stat_err_bits;
                                		u16 vlan_tci;
                                
                                		rx_desc = ICE_RX_DESC(rx_ring, ntc);
                                
                                		stat_err_bits = BIT(ICE_RX_FLEX_DESC_STATUS0_DD_S);
                                		if (!ice_test_staterr(rx_desc->wb.status_error0, stat_err_bits))
                                			break;
                                
                                		/* This memory barrier is needed to keep us from reading
                                		 * any other fields out of the rx_desc until we have
                                		 * verified the descriptor has been written back.
                                		 */
                                		dma_rmb();
                                
                                		if (unlikely(ntc == ntu))
                                			break;
                                
                                		xdp = *ice_xdp_buf(rx_ring, ntc);
                                
                                		size = le16_to_cpu(rx_desc->wb.pkt_len) &
                                				   ICE_RX_FLX_DESC_PKT_LEN_M;
                                
                                		xsk_buff_set_size(xdp, size);
                                		xsk_buff_dma_sync_for_cpu(xdp, xsk_pool);
                                
                                		if (!first) {
                                			first = xdp;
                                		} else if (ice_add_xsk_frag(rx_ring, first, xdp, size)) {
                                			break;
                                		}
                                
                                		if (++ntc == cnt)
                                			ntc = 0;
                                
                                		if (ice_is_non_eop(rx_ring, rx_desc))
                                			continue;
                                
                                		xdp_res = ice_run_xdp_zc(rx_ring, first, xdp_prog, xdp_ring);
                                		if (likely(xdp_res & (ICE_XDP_TX | ICE_XDP_REDIR))) {
                                			xdp_xmit |= xdp_res;
                                		} else if (xdp_res == ICE_XDP_EXIT) {
                                			failure = true;
                                			first = NULL;
                                			rx_ring->first_desc = ntc;
                                			break;
                                		} else if (xdp_res == ICE_XDP_CONSUMED) {
                                			xsk_buff_free(first);
                                		} else if (xdp_res == ICE_XDP_PASS) {
                                			goto construct_skb;
                                		}
                                
                                		total_rx_bytes += xdp_get_buff_len(first);
                                		total_rx_packets++;
                                
                                		first = NULL;
                                		rx_ring->first_desc = ntc;
                                		continue;
                                
                                construct_skb:
                                		/* XDP_PASS path */
                                		skb = ice_construct_skb_zc(rx_ring, first);
                                		if (!skb) {
                                			rx_ring->ring_stats->rx_stats.alloc_buf_failed++;
                                			break;
                                		}
                                
                                		first = NULL;
                                		rx_ring->first_desc = ntc;
                                
                                		if (eth_skb_pad(skb)) {
                                			skb = NULL;
                                			continue;
                                		}
                                
                                		total_rx_bytes += skb->len;
                                		total_rx_packets++;
                                
                                		vlan_tci = ice_get_vlan_tci(rx_desc);
                                
                                		ice_process_skb_fields(rx_ring, rx_desc, skb);
                                		ice_receive_skb(rx_ring, skb, vlan_tci);
                                	}
                                
                                	rx_ring->next_to_clean = ntc;
                                	entries_to_alloc = ICE_RX_DESC_UNUSED(rx_ring);
                                	if (entries_to_alloc > ICE_RING_QUARTER(rx_ring))
                                		failure |= !ice_alloc_rx_bufs_zc(rx_ring, entries_to_alloc);
                                
                                	ice_finalize_xdp_rx(xdp_ring, xdp_xmit, 0);
                                	ice_update_rx_ring_stats(rx_ring, total_rx_packets, total_rx_bytes);
                                
                                	if (xsk_uses_need_wakeup(xsk_pool)) {
                                		/* ntu could have changed when allocating entries above, so
                                		 * use rx_ring value instead of stack based one
                                		 */
                                		if (failure || ntc == rx_ring->next_to_use)
                                			xsk_set_rx_need_wakeup(xsk_pool);
                                		else
                                			xsk_clear_rx_need_wakeup(xsk_pool);
                                
                                		return (int)total_rx_packets;
                                	}
                                
                                	return failure ? budget : (int)total_rx_packets;
                                }
                                ```
                                
                            
                            > ~~xdp를 안쓴다고 가정하므로 일단 넘어가기로 하였다.~~
                            
                        - ice_clean_rx_irq(rx_ring, budget_per_ring) ==(ice/ice_txrx.c)==
                            
                            ---
                            
                            - 코드
                                
                                ```C
                                /**
                                 * ice_clean_rx_irq - Clean completed descriptors from Rx ring - bounce buf
                                 * @rx_ring: Rx descriptor ring to transact packets on
                                 * @budget: Total limit on number of packets to process
                                 *
                                 * This function provides a "bounce buffer" approach to Rx interrupt
                                 * processing. The advantage to this is that on systems that have
                                 * expensive overhead for IOMMU access this provides a means of avoiding
                                 * it by maintaining the mapping of the page to the system.
                                 *
                                 * Returns amount of work completed
                                 */
                                int ice_clean_rx_irq(struct ice_rx_ring *rx_ring, int budget)
                                {
                                	unsigned int total_rx_bytes = 0, total_rx_pkts = 0;
                                	unsigned int offset = rx_ring->rx_offset;
                                	struct xdp_buff *xdp = &rx_ring->xdp;
                                	u32 cached_ntc = rx_ring->first_desc;
                                	struct ice_tx_ring *xdp_ring = NULL;
                                	struct bpf_prog *xdp_prog = NULL;
                                	u32 ntc = rx_ring->next_to_clean;
                                	u32 cnt = rx_ring->count;
                                	u32 xdp_xmit = 0;
                                	u32 cached_ntu;
                                	bool failure;
                                	u32 first;
                                
                                	/* Frame size depend on rx_ring setup when PAGE_SIZE=4K */
                                \#if (PAGE_SIZE < 8192)
                                	xdp->frame_sz = ice_rx_frame_truesize(rx_ring, 0);
                                \#endif
                                
                                	xdp_prog = READ_ONCE(rx_ring->xdp_prog);
                                	if (xdp_prog) {
                                		xdp_ring = rx_ring->xdp_ring;
                                		cached_ntu = xdp_ring->next_to_use;
                                	}
                                
                                	/* start the loop to process Rx packets bounded by 'budget' */
                                	while (likely(total_rx_pkts < (unsigned int)budget)) {
                                		union ice_32b_rx_flex_desc *rx_desc;
                                		struct ice_rx_buf *rx_buf;
                                		struct sk_buff *skb;
                                		unsigned int size;
                                		u16 stat_err_bits;
                                		u16 vlan_tci;
                                
                                		/* get the Rx desc from Rx ring based on 'next_to_clean' */
                                		rx_desc = ICE_RX_DESC(rx_ring, ntc);
                                
                                		/* status_error_len will always be zero for unused descriptors
                                		 * because it's cleared in cleanup, and overlaps with hdr_addr
                                		 * which is always zero because packet split isn't used, if the
                                		 * hardware wrote DD then it will be non-zero
                                		 */
                                		stat_err_bits = BIT(ICE_RX_FLEX_DESC_STATUS0_DD_S);
                                		if (!ice_test_staterr(rx_desc->wb.status_error0, stat_err_bits))
                                			break;
                                
                                		/* This memory barrier is needed to keep us from reading
                                		 * any other fields out of the rx_desc until we know the
                                		 * DD bit is set.
                                		 */
                                		dma_rmb();
                                
                                		ice_trace(clean_rx_irq, rx_ring, rx_desc);
                                		if (rx_desc->wb.rxdid == FDIR_DESC_RXDID || !rx_ring->netdev) {
                                			struct ice_vsi *ctrl_vsi = rx_ring->vsi;
                                
                                			if (rx_desc->wb.rxdid == FDIR_DESC_RXDID &&
                                			    ctrl_vsi->vf)
                                				ice_vc_fdir_irq_handler(ctrl_vsi, rx_desc);
                                			if (++ntc == cnt)
                                				ntc = 0;
                                			rx_ring->first_desc = ntc;
                                			continue;
                                		}
                                
                                		size = le16_to_cpu(rx_desc->wb.pkt_len) &
                                			ICE_RX_FLX_DESC_PKT_LEN_M;
                                
                                		/* retrieve a buffer from the ring */
                                		rx_buf = ice_get_rx_buf(rx_ring, size, ntc);
                                
                                		if (!xdp->data) {
                                			void *hard_start;
                                
                                			hard_start = page_address(rx_buf->page) + rx_buf->page_offset -
                                				     offset;
                                			xdp_prepare_buff(xdp, hard_start, offset, size, !!offset);
                                \#if (PAGE_SIZE > 4096)
                                			/* At larger PAGE_SIZE, frame_sz depend on len size */
                                			xdp->frame_sz = ice_rx_frame_truesize(rx_ring, size);
                                \#endif
                                			xdp_buff_clear_frags_flag(xdp);
                                		} else if (ice_add_xdp_frag(rx_ring, xdp, rx_buf, size)) {
                                			break;
                                		}
                                		if (++ntc == cnt)
                                			ntc = 0;
                                
                                		/* skip if it is NOP desc */
                                		if (ice_is_non_eop(rx_ring, rx_desc))
                                			continue;
                                
                                		ice_run_xdp(rx_ring, xdp, xdp_prog, xdp_ring, rx_buf, rx_desc);
                                		if (rx_buf->act == ICE_XDP_PASS)
                                			goto construct_skb;
                                		total_rx_bytes += xdp_get_buff_len(xdp);
                                		total_rx_pkts++;
                                
                                		xdp->data = NULL;
                                		rx_ring->first_desc = ntc;
                                		rx_ring->nr_frags = 0;
                                		continue;
                                construct_skb:
                                		if (likely(ice_ring_uses_build_skb(rx_ring)))
                                			skb = ice_build_skb(rx_ring, xdp);
                                		else
                                			skb = ice_construct_skb(rx_ring, xdp);
                                		/* exit if we failed to retrieve a buffer */
                                		if (!skb) {
                                			rx_ring->ring_stats->rx_stats.alloc_page_failed++;
                                			rx_buf->act = ICE_XDP_CONSUMED;
                                			if (unlikely(xdp_buff_has_frags(xdp)))
                                				ice_set_rx_bufs_act(xdp, rx_ring,
                                						    ICE_XDP_CONSUMED);
                                			xdp->data = NULL;
                                			rx_ring->first_desc = ntc;
                                			rx_ring->nr_frags = 0;
                                			break;
                                		}
                                		xdp->data = NULL;
                                		rx_ring->first_desc = ntc;
                                		rx_ring->nr_frags = 0;
                                
                                		stat_err_bits = BIT(ICE_RX_FLEX_DESC_STATUS0_RXE_S);
                                		if (unlikely(ice_test_staterr(rx_desc->wb.status_error0,
                                					      stat_err_bits))) {
                                			dev_kfree_skb_any(skb);
                                			continue;
                                		}
                                
                                		vlan_tci = ice_get_vlan_tci(rx_desc);
                                
                                		/* pad the skb if needed, to make a valid ethernet frame */
                                		if (eth_skb_pad(skb))
                                			continue;
                                
                                		/* probably a little skewed due to removing CRC */
                                		total_rx_bytes += skb->len;
                                
                                		/* populate checksum, VLAN, and protocol */
                                		ice_process_skb_fields(rx_ring, rx_desc, skb);
                                
                                		ice_trace(clean_rx_irq_indicate, rx_ring, rx_desc, skb);
                                		/* send completed skb up the stack */
                                		ice_receive_skb(rx_ring, skb, vlan_tci);
                                
                                		/* update budget accounting */
                                		total_rx_pkts++;
                                	}
                                
                                	first = rx_ring->first_desc;
                                	while (cached_ntc != first) {
                                		struct ice_rx_buf *buf = &rx_ring->rx_buf[cached_ntc];
                                
                                		if (buf->act & (ICE_XDP_TX | ICE_XDP_REDIR)) {
                                			ice_rx_buf_adjust_pg_offset(buf, xdp->frame_sz);
                                			xdp_xmit |= buf->act;
                                		} else if (buf->act & ICE_XDP_CONSUMED) {
                                			buf->pagecnt_bias++;
                                		} else if (buf->act == ICE_XDP_PASS) {
                                			ice_rx_buf_adjust_pg_offset(buf, xdp->frame_sz);
                                		}
                                
                                		ice_put_rx_buf(rx_ring, buf);
                                		if (++cached_ntc >= cnt)
                                			cached_ntc = 0;
                                	}
                                	rx_ring->next_to_clean = ntc;
                                	/* return up to cleaned_count buffers to hardware */
                                	failure = ice_alloc_rx_bufs(rx_ring, ICE_RX_DESC_UNUSED(rx_ring));
                                
                                	if (xdp_xmit)
                                		ice_finalize_xdp_rx(xdp_ring, xdp_xmit, cached_ntu);
                                
                                	if (rx_ring->ring_stats)
                                		ice_update_rx_ring_stats(rx_ring, total_rx_pkts,
                                					 total_rx_bytes);
                                
                                	/* guarantee a trip back through this routine if there was a failure */
                                	return failure ? budget : (int)total_rx_pkts;
                                }
                                ```
                                
                            
                            > xdp를 무조건 셋팅하는 것으로 보인다. 우선 없다고 가정하고 보면 if(!xdp→data)를 통해 확인하여 ice_add_xdp_frag() 함수를 실행할 것이다. 그러고 나서 ice_run_xdp()를 실행한다.  
                            > 이후에 보면 construct_skb:라는 label이 존재한다. 이는 네트워크 스택에 링버퍼의 내용을 전달하는 뜻으로, ICE_XDP_PASS값을 rx_buf→act에서 가질 때 실행되게 된다. 그게 아니라면 중간의 continue; 때문에 construct_skb: 라벨 아래의 코드들은 실행되지 않게 된다.  
                            > construct_skb: 라벨 아래에서는 전통적인 네트워크 스택이 진행되게 된다.  
                            
                            - ice_get_rx_buf(rx_ring, size, ntc)
                                
                                ---
                                
                                - 코드
                                    
                                    ```C
                                    static struct ice_rx_buf *
                                    ice_get_rx_buf(struct ice_rx_ring *rx_ring, const unsigned int size,
                                    	       const unsigned int ntc)
                                    {
                                    	struct ice_rx_buf *rx_buf;
                                    
                                    	rx_buf = &rx_ring->rx_buf[ntc];
                                    	rx_buf->pgcnt =
                                    \#if (PAGE_SIZE < 8192)
                                    		page_count(rx_buf->page);
                                    \#else
                                    		0;
                                    \#endif
                                    	prefetchw(rx_buf->page);
                                    
                                    	if (!size)
                                    		return rx_buf;
                                    	/* we are reusing so sync this buffer for CPU use */
                                    	dma_sync_single_range_for_cpu(rx_ring->dev, rx_buf->dma,
                                    				      rx_buf->page_offset, size,
                                    				      DMA_FROM_DEVICE);
                                    
                                    	/* We have pulled a buffer for use, so decrement pagecnt_bias */
                                    	rx_buf->pagecnt_bias--;
                                    
                                    	return rx_buf;
                                    }
                                    ```
                                    
                                
                                > Rx Buffer를 사용하기 위해 가져와서 동기화시키는 함수이다. prefetchw(rx_buf→page)함수를 사용한다. 그후 dma_sync_single_range_for_cpu() 함수를 통해 싱크를 진행한다.
                                
                            - xdp_prepare_buff(xdp, hard_start, offset, size, !!offset) ==(include/net/xdp.h)==
                                
                                ---
                                
                                - 코드
                                    
                                    ```C++
                                    static __always_inline void
                                    xdp_prepare_buff(struct xdp_buff *xdp, unsigned char *hard_start,
                                    		 int headroom, int data_len, const bool meta_valid)
                                    {
                                    	unsigned char *data = hard_start + headroom;
                                    
                                    	xdp->data_hard_start = hard_start;
                                    	xdp->data = data;
                                    	xdp->data_end = data + data_len;
                                    	xdp->data_meta = meta_valid ? data : data + 1;
                                    }
                                    ```
                                    
                                
                                > 주어진 xdp 포인터에다가 hard_start, data, data_end, data_meta를 넣어서 준비시켜준다.
                                
                            - ice_add_xdp_frag(rx_ring, xdp, rx_buf, size) ==(ice/ice_txrx.c)==
                                
                                ---
                                
                                - 코드
                                    
                                    ```C
                                    /**
                                     * ice_add_xdp_frag - Add contents of Rx buffer to xdp buf as a frag
                                     * @rx_ring: Rx descriptor ring to transact packets on
                                     * @xdp: xdp buff to place the data into
                                     * @rx_buf: buffer containing page to add
                                     * @size: packet length from rx_desc
                                     *
                                     * This function will add the data contained in rx_buf->page to the xdp buf.
                                     * It will just attach the page as a frag.
                                     */
                                    static int
                                    ice_add_xdp_frag(struct ice_rx_ring *rx_ring, struct xdp_buff *xdp,
                                    		 struct ice_rx_buf *rx_buf, const unsigned int size)
                                    {
                                    	struct skb_shared_info *sinfo = xdp_get_shared_info_from_buff(xdp);
                                    
                                    	if (!size)
                                    		return 0;
                                    
                                    	if (!xdp_buff_has_frags(xdp)) {
                                    		sinfo->nr_frags = 0;
                                    		sinfo->xdp_frags_size = 0;
                                    		xdp_buff_set_frags_flag(xdp);
                                    	}
                                    
                                    	if (unlikely(sinfo->nr_frags == MAX_SKB_FRAGS)) {
                                    		ice_set_rx_bufs_act(xdp, rx_ring, ICE_XDP_CONSUMED);
                                    		return -ENOMEM;
                                    	}
                                    
                                    	__skb_fill_page_desc_noacc(sinfo, sinfo->nr_frags++, rx_buf->page,
                                    				   rx_buf->page_offset, size);
                                    	sinfo->xdp_frags_size += size;
                                    	/* remember frag count before XDP prog execution; bpf_xdp_adjust_tail()
                                    	 * can pop off frags but driver has to handle it on its own
                                    	 */
                                    	rx_ring->nr_frags = sinfo->nr_frags;
                                    
                                    	if (page_is_pfmemalloc(rx_buf->page))
                                    		xdp_buff_set_frag_pfmemalloc(xdp);
                                    
                                    	return 0;
                                    }
                                    ```
                                    
                                
                                > xdp buf에다가 frag로써 Rx buffer의 컨텐츠를 추가하게 된다.
                                
                            - ice_run_xdp(rx_ring, xdp, xdp_prog, xdp_ring, rx_buf, rx_desc)
                                
                                ---
                                
                                - 코드
                                    
                                    ```C
                                    /**
                                     * ice_run_xdp - Executes an XDP program on initialized xdp_buff
                                     * @rx_ring: Rx ring
                                     * @xdp: xdp_buff used as input to the XDP program
                                     * @xdp_prog: XDP program to run
                                     * @xdp_ring: ring to be used for XDP_TX action
                                     * @rx_buf: Rx buffer to store the XDP action
                                     * @eop_desc: Last descriptor in packet to read metadata from
                                     *
                                     * Returns any of ICE_XDP_{PASS, CONSUMED, TX, REDIR}
                                     */
                                    static void
                                    ice_run_xdp(struct ice_rx_ring *rx_ring, struct xdp_buff *xdp,
                                    	    struct bpf_prog *xdp_prog, struct ice_tx_ring *xdp_ring,
                                    	    struct ice_rx_buf *rx_buf, union ice_32b_rx_flex_desc *eop_desc)
                                    {
                                    	unsigned int ret = ICE_XDP_PASS;
                                    	u32 act;
                                    
                                    	if (!xdp_prog)
                                    		goto exit;
                                    
                                    	ice_xdp_meta_set_desc(xdp, eop_desc);
                                    
                                    	act = bpf_prog_run_xdp(xdp_prog, xdp);
                                    	switch (act) {
                                    	case XDP_PASS:
                                    		break;
                                    	case XDP_TX:
                                    		if (static_branch_unlikely(&ice_xdp_locking_key))
                                    			spin_lock(&xdp_ring->tx_lock);
                                    		ret = __ice_xmit_xdp_ring(xdp, xdp_ring, false);
                                    		if (static_branch_unlikely(&ice_xdp_locking_key))
                                    			spin_unlock(&xdp_ring->tx_lock);
                                    		if (ret == ICE_XDP_CONSUMED)
                                    			goto out_failure;
                                    		break;
                                    	case XDP_REDIRECT:
                                    		if (xdp_do_redirect(rx_ring->netdev, xdp, xdp_prog))
                                    			goto out_failure;
                                    		ret = ICE_XDP_REDIR;
                                    		break;
                                    	default:
                                    		bpf_warn_invalid_xdp_action(rx_ring->netdev, xdp_prog, act);
                                    		fallthrough;
                                    	case XDP_ABORTED:
                                    out_failure:
                                    		trace_xdp_exception(rx_ring->netdev, xdp_prog, act);
                                    		fallthrough;
                                    	case XDP_DROP:
                                    		ret = ICE_XDP_CONSUMED;
                                    	}
                                    exit:
                                    	ice_set_rx_bufs_act(xdp, rx_ring, ret);
                                    }
                                    ```
                                    
                                
                                > bpf_prog_run_xdp(xdp_prog, xdp) 함수를 를 통해서 act를 얻게 되고, 이걸 바탕으로 switch문으로 들어가게 된다. XDP_PASS, XDP_TX, XDP_REDIRECT, XDP_ABORTED, XDP_DROP 등의 enum이 있으며, 가운데 bpf_prog_run_xdp()를 통해 bpf가 실행되는데, 여기서 XDP_PASS act가 되어 ret이 ICE_XDP_PASS가 되면 네트워크 스택을 지나지 않는 것이고, 아니라면 XDP_DROP에서 ret이 ICE_XDP_CONSUMED로 전통적인 네트워크 스택을 지나게 될 것이다.
                                
                                - __ice_xmit_xdp_ring(xdp, xdp_ring, false) ==(ice/ice_txrx_lib.c)==
                                    
                                    ---
                                    
                                    - 코드
                                        
                                    
                                    > Frame을 XDP ring으로 제출하여 전송을 할수 있도록 한다. ice_run_xdp 자체가 tx든 rx든 통합되어 실행되므로, bottom up path에서는 본 코드는 실행되지 않을 것이라고 생각한다. 여기서부터는 본격적으로 XDP이다.
                                    
                            
                            > [!important]  
                            > construct_skb:  
                            
                            - ice_ring_uses_build_skb(rx_ring) ==(ice/ice_txrx.h)==
                                
                                ---
                                
                                - 코드
                                    
                                    ```C
                                    static inline bool ice_ring_uses_build_skb(struct ice_rx_ring *ring)
                                    {
                                    	return !!(ring->flags & ICE_RX_FLAGS_RING_BUILD_SKB);
                                    }
                                    ```
                                    
                                
                                > 간단한 인라인 함수이다. rx_ring의 flag에서 ICE_RX_FLAGS_RINGS_BUILD_SKB가 켜져 있는지만 확인하는 함수이다.
                                
                            - ice_build_skb(rx_ring, xdp) ==(ice/ice_txrx.c)==
                                
                                ---
                                
                                - 코드
                                    
                                    ```C
                                    /**
                                     * ice_build_skb - Build skb around an existing buffer
                                     * @rx_ring: Rx descriptor ring to transact packets on
                                     * @xdp: xdp_buff pointing to the data
                                     *
                                     * This function builds an skb around an existing XDP buffer, taking care
                                     * to set up the skb correctly and avoid any memcpy overhead. Driver has
                                     * already combined frags (if any) to skb_shared_info.
                                     */
                                    static struct sk_buff *
                                    ice_build_skb(struct ice_rx_ring *rx_ring, struct xdp_buff *xdp)
                                    {
                                    	u8 metasize = xdp->data - xdp->data_meta;
                                    	struct skb_shared_info *sinfo = NULL;
                                    	unsigned int nr_frags;
                                    	struct sk_buff *skb;
                                    
                                    	if (unlikely(xdp_buff_has_frags(xdp))) {
                                    		sinfo = xdp_get_shared_info_from_buff(xdp);
                                    		nr_frags = sinfo->nr_frags;
                                    	}
                                    
                                    	/* Prefetch first cache line of first page. If xdp->data_meta
                                    	 * is unused, this points exactly as xdp->data, otherwise we
                                    	 * likely have a consumer accessing first few bytes of meta
                                    	 * data, and then actual data.
                                    	 */
                                    	net_prefetch(xdp->data_meta);
                                    	/* build an skb around the page buffer */
                                    	skb = napi_build_skb(xdp->data_hard_start, xdp->frame_sz);
                                    	if (unlikely(!skb))
                                    		return NULL;
                                    
                                    	/* must to record Rx queue, otherwise OS features such as
                                    	 * symmetric queue won't work
                                    	 */
                                    	skb_record_rx_queue(skb, rx_ring->q_index);
                                    
                                    	/* update pointers within the skb to store the data */
                                    	skb_reserve(skb, xdp->data - xdp->data_hard_start);
                                    	__skb_put(skb, xdp->data_end - xdp->data);
                                    	if (metasize)
                                    		skb_metadata_set(skb, metasize);
                                    
                                    	if (unlikely(xdp_buff_has_frags(xdp)))
                                    		xdp_update_skb_shared_info(skb, nr_frags,
                                    					   sinfo->xdp_frags_size,
                                    					   nr_frags * xdp->frame_sz,
                                    					   xdp_buff_is_frag_pfmemalloc(xdp));
                                    
                                    	return skb;
                                    }
                                    ```
                                    
                                
                                > 존재하는 XDP 버퍼에다가 skb 버퍼를 만드는 함수이다. build를 할 수있는지 위의 flag에 따라 본 함수가 실행되거나 아래의 ice_construct_skb가 실행되게 된다.
                                
                                - napi_build_skb(xdp→data_hard_start, xdp→frame_sz) ==(net/core/skbuff.c)==
                                    
                                    ---
                                    
                                    - 코드
                                        
                                        ```C
                                        /**
                                         * napi_build_skb - build a network buffer
                                         * @data: data buffer provided by caller
                                         * @frag_size: size of data
                                         *
                                         * Version of __napi_build_skb() that takes care of skb->head_frag
                                         * and skb->pfmemalloc when the data is a page or page fragment.
                                         *
                                         * Returns a new &sk_buff on success, %NULL on allocation failure.
                                         */
                                        struct sk_buff *napi_build_skb(void *data, unsigned int frag_size)
                                        {
                                        	struct sk_buff *skb = __napi_build_skb(data, frag_size);
                                        
                                        	if (likely(skb) && frag_size) {
                                        		skb->head_frag = 1;
                                        		skb_propagate_pfmemalloc(virt_to_head_page(data), skb);
                                        	}
                                        
                                        	return skb;
                                        }
                                        EXPORT_SYMBOL(napi_build_skb);
                                        ```
                                        
                                    
                                    > __napi_build_skb를 실행하여 skb 포인터를 획득하고, 이를 다시 반환한다. 이후 skb_propagate_pfmemalloc을 호출하게 된다.
                                    
                                    - __napi_build_skb(data, frag_size)
                                        
                                        ---
                                        
                                        - 코드
                                            
                                            ```C
                                            /**
                                             * __napi_build_skb - build a network buffer
                                             * @data: data buffer provided by caller
                                             * @frag_size: size of data
                                             *
                                             * Version of __build_skb() that uses NAPI percpu caches to obtain
                                             * skbuff_head instead of inplace allocation.
                                             *
                                             * Returns a new &sk_buff on success, %NULL on allocation failure.
                                             */
                                            static struct sk_buff *__napi_build_skb(void *data, unsigned int frag_size)
                                            {
                                            	struct sk_buff *skb;
                                            
                                            	skb = napi_skb_cache_get();
                                            	if (unlikely(!skb))
                                            		return NULL;
                                            
                                            	memset(skb, 0, offsetof(struct sk_buff, tail));
                                            	__build_skb_around(skb, data, frag_size);
                                            
                                            	return skb;
                                            }
                                            ```
                                            
                                        
                                        > 이 함수는 우선 napi_skb_cache_get()을 통해 CPU 로컬한 캐시에서 skb_cache를 가져오는 작업을 하게 된다. 이는 성능향상을 위한 작업이며, 이후 skb의 크기만큼 memset을 통해 초기화를 해주고 __build_skb_around()를 통해 마저 skb를 구성하게 된다.
                                        
                                    - skb_propagate_pfmemalloc(virt_to_head_page(data), skb)
                                        
                                          
                                        
                                - __skb_put(skb, xdp→data_end - xdp→data) ==(include/linux/skbuff.h)==
                                    
                                    ---
                                    
                                    - 코드
                                        
                                        ```C
                                        static inline void *__skb_put(struct sk_buff *skb, unsigned int len)
                                        {
                                        	void *tmp = skb_tail_pointer(skb);
                                        	SKB_LINEAR_ASSERT(skb);
                                        	skb->tail += len;
                                        	skb->len  += len;
                                        	return tmp;
                                        }
                                        ```
                                        
                                    
                                    > 통째로 집어 넣는 코드. 좀더 찾아봐야함.
                                    
                            - ice_construct_skb(rx_ring, xdp)
                                
                                ---
                                
                                - 코드
                                    
                                    ```C
                                    /**
                                     * ice_construct_skb - Allocate skb and populate it
                                     * @rx_ring: Rx descriptor ring to transact packets on
                                     * @xdp: xdp_buff pointing to the data
                                     *
                                     * This function allocates an skb. It then populates it with the page
                                     * data from the current receive descriptor, taking care to set up the
                                     * skb correctly.
                                     */
                                    static struct sk_buff *
                                    ice_construct_skb(struct ice_rx_ring *rx_ring, struct xdp_buff *xdp)
                                    {
                                    	unsigned int size = xdp->data_end - xdp->data;
                                    	struct skb_shared_info *sinfo = NULL;
                                    	struct ice_rx_buf *rx_buf;
                                    	unsigned int nr_frags = 0;
                                    	unsigned int headlen;
                                    	struct sk_buff *skb;
                                    
                                    	/* prefetch first cache line of first page */
                                    	net_prefetch(xdp->data);
                                    
                                    	if (unlikely(xdp_buff_has_frags(xdp))) {
                                    		sinfo = xdp_get_shared_info_from_buff(xdp);
                                    		nr_frags = sinfo->nr_frags;
                                    	}
                                    
                                    	/* allocate a skb to store the frags */
                                    	skb = __napi_alloc_skb(&rx_ring->q_vector->napi, ICE_RX_HDR_SIZE,
                                    			       GFP_ATOMIC | __GFP_NOWARN);
                                    	if (unlikely(!skb))
                                    		return NULL;
                                    
                                    	rx_buf = &rx_ring->rx_buf[rx_ring->first_desc];
                                    	skb_record_rx_queue(skb, rx_ring->q_index);
                                    	/* Determine available headroom for copy */
                                    	headlen = size;
                                    	if (headlen > ICE_RX_HDR_SIZE)
                                    		headlen = eth_get_headlen(skb->dev, xdp->data, ICE_RX_HDR_SIZE);
                                    
                                    	/* align pull length to size of long to optimize memcpy performance */
                                    	memcpy(__skb_put(skb, headlen), xdp->data, ALIGN(headlen,
                                    							 sizeof(long)));
                                    
                                    	/* if we exhaust the linear part then add what is left as a frag */
                                    	size -= headlen;
                                    	if (size) {
                                    		/* besides adding here a partial frag, we are going to add
                                    		 * frags from xdp_buff, make sure there is enough space for
                                    		 * them
                                    		 */
                                    		if (unlikely(nr_frags >= MAX_SKB_FRAGS - 1)) {
                                    			dev_kfree_skb(skb);
                                    			return NULL;
                                    		}
                                    		skb_add_rx_frag(skb, 0, rx_buf->page,
                                    				rx_buf->page_offset + headlen, size,
                                    				xdp->frame_sz);
                                    	} else {
                                    		/* buffer is unused, change the act that should be taken later
                                    		 * on; data was copied onto skb's linear part so there's no
                                    		 * need for adjusting page offset and we can reuse this buffer
                                    		 * as-is
                                    		 */
                                    		rx_buf->act = ICE_SKB_CONSUMED;
                                    	}
                                    
                                    	if (unlikely(xdp_buff_has_frags(xdp))) {
                                    		struct skb_shared_info *skinfo = skb_shinfo(skb);
                                    
                                    		memcpy(&skinfo->frags[skinfo->nr_frags], &sinfo->frags[0],
                                    		       sizeof(skb_frag_t) * nr_frags);
                                    
                                    		xdp_update_skb_shared_info(skb, skinfo->nr_frags + nr_frags,
                                    					   sinfo->xdp_frags_size,
                                    					   nr_frags * xdp->frame_sz,
                                    					   xdp_buff_is_frag_pfmemalloc(xdp));
                                    	}
                                    
                                    	return skb;
                                    }
                                    ```
                                    
                                
                                > 첫번째 차이점은 napi_build_skb vs __napi_alloc_skb 실행이다. 이후 skb_record_rx_queue를 실행하게 된다. 이후 memcpy를 통해 xdp→data를 직접 가져오게 된다. 이를 sk_buff가 가르키고 있는 data 헤더에다가 넣기 시작한다.
                                
                                - __napi_alloc_skb(&rx_ring→q_vector→napi, ICE_RX_HDR_SIZE, GFP_ATOMIC | __GFP_NOWARN) ==(net/core/skbuff.c)==
                                    
                                    ---
                                    
                                    - 코드
                                        
                                        ```C
                                        /**
                                         *	__napi_alloc_skb - allocate skbuff for rx in a specific NAPI instance
                                         *	@napi: napi instance this buffer was allocated for
                                         *	@len: length to allocate
                                         *	@gfp_mask: get_free_pages mask, passed to alloc_skb and alloc_pages
                                         *
                                         *	Allocate a new sk_buff for use in NAPI receive.  This buffer will
                                         *	attempt to allocate the head from a special reserved region used
                                         *	only for NAPI Rx allocation.  By doing this we can save several
                                         *	CPU cycles by avoiding having to disable and re-enable IRQs.
                                         *
                                         *	%NULL is returned if there is no free memory.
                                         */
                                        struct sk_buff *__napi_alloc_skb(struct napi_struct *napi, unsigned int len,
                                        				 gfp_t gfp_mask)
                                        {
                                        	struct napi_alloc_cache *nc;
                                        	struct sk_buff *skb;
                                        	bool pfmemalloc;
                                        	void *data;
                                        
                                        	DEBUG_NET_WARN_ON_ONCE(!in_softirq());
                                        	len += NET_SKB_PAD + NET_IP_ALIGN;
                                        
                                        	/* If requested length is either too small or too big,
                                        	 * we use kmalloc() for skb->head allocation.
                                        	 * When the small frag allocator is available, prefer it over kmalloc
                                        	 * for small fragments
                                        	 */
                                        	if ((!NAPI_HAS_SMALL_PAGE_FRAG && len <= SKB_WITH_OVERHEAD(1024)) ||
                                        	    len > SKB_WITH_OVERHEAD(PAGE_SIZE) ||
                                        	    (gfp_mask & (__GFP_DIRECT_RECLAIM | GFP_DMA))) {
                                        		skb = __alloc_skb(len, gfp_mask, SKB_ALLOC_RX | SKB_ALLOC_NAPI,
                                        				  NUMA_NO_NODE);
                                        		if (!skb)
                                        			goto skb_fail;
                                        		goto skb_success;
                                        	}
                                        
                                        	nc = this_cpu_ptr(&napi_alloc_cache);
                                        
                                        	if (sk_memalloc_socks())
                                        		gfp_mask |= __GFP_MEMALLOC;
                                        
                                        	if (NAPI_HAS_SMALL_PAGE_FRAG && len <= SKB_WITH_OVERHEAD(1024)) {
                                        		/* we are artificially inflating the allocation size, but
                                        		 * that is not as bad as it may look like, as:
                                        		 * - 'len' less than GRO_MAX_HEAD makes little sense
                                        		 * - On most systems, larger 'len' values lead to fragment
                                        		 *   size above 512 bytes
                                        		 * - kmalloc would use the kmalloc-1k slab for such values
                                        		 * - Builds with smaller GRO_MAX_HEAD will very likely do
                                        		 *   little networking, as that implies no WiFi and no
                                        		 *   tunnels support, and 32 bits arches.
                                        		 */
                                        		len = SZ_1K;
                                        
                                        		data = page_frag_alloc_1k(&nc->page_small, gfp_mask);
                                        		pfmemalloc = NAPI_SMALL_PAGE_PFMEMALLOC(nc->page_small);
                                        	} else {
                                        		len = SKB_HEAD_ALIGN(len);
                                        
                                        		data = page_frag_alloc(&nc->page, len, gfp_mask);
                                        		pfmemalloc = nc->page.pfmemalloc;
                                        	}
                                        
                                        	if (unlikely(!data))
                                        		return NULL;
                                        
                                        	skb = __napi_build_skb(data, len);
                                        	if (unlikely(!skb)) {
                                        		skb_free_frag(data);
                                        		return NULL;
                                        	}
                                        
                                        	if (pfmemalloc)
                                        		skb->pfmemalloc = 1;
                                        	skb->head_frag = 1;
                                        
                                        skb_success:
                                        	skb_reserve(skb, NET_SKB_PAD + NET_IP_ALIGN);
                                        	skb->dev = napi->dev;
                                        
                                        skb_fail:
                                        	return skb;
                                        }
                                        ```
                                        
                                    
                                    > 만약 별 문제가 없다면 __alloc_skb()를 통해 skb를 새로 할당하게 되고, 이후의 과정은 __napi_build_skb등을 거치며 위의 ice_build_skb와 같아지게 된다. 즉, 사용할 skb가 할당 되어있는지 여부에 의해 사용될 함수가 결정된다고 볼 수 있다.  
                                    > cache에서 get하는 것은 기존의 skb를 재사용 하는 것으로, 여기서는 skb를 직접 다시 만들고 있다. data 포인터는 sk_buff가 가르킬 page를 메모리 할당한 것이다.  
                                    
                                    - __alloc_skb(len, gfp_mask, SKB_ALLOC_RX | SKB_ALLOC_NAPI, NUMA_NO_NODE) ==(net/core/skbuff.c)==
                                        
                                        ---
                                        
                                        - 코드
                                            
                                            ```C
                                            /**
                                             *	__alloc_skb	-	allocate a network buffer
                                             *	@size: size to allocate
                                             *	@gfp_mask: allocation mask
                                             *	@flags: If SKB_ALLOC_FCLONE is set, allocate from fclone cache
                                             *		instead of head cache and allocate a cloned (child) skb.
                                             *		If SKB_ALLOC_RX is set, __GFP_MEMALLOC will be used for
                                             *		allocations in case the data is required for writeback
                                             *	@node: numa node to allocate memory on
                                             *
                                             *	Allocate a new &sk_buff. The returned buffer has no headroom and a
                                             *	tail room of at least size bytes. The object has a reference count
                                             *	of one. The return is the buffer. On a failure the return is %NULL.
                                             *
                                             *	Buffers may only be allocated from interrupts using a @gfp_mask of
                                             *	%GFP_ATOMIC.
                                             */
                                            struct sk_buff *__alloc_skb(unsigned int size, gfp_t gfp_mask,
                                            			    int flags, int node)
                                            {
                                            	struct kmem_cache *cache;
                                            	struct sk_buff *skb;
                                            	bool pfmemalloc;
                                            	u8 *data;
                                            
                                            	cache = (flags & SKB_ALLOC_FCLONE)
                                            		? net_hotdata.skbuff_fclone_cache : net_hotdata.skbuff_cache;
                                            
                                            	if (sk_memalloc_socks() && (flags & SKB_ALLOC_RX))
                                            		gfp_mask |= __GFP_MEMALLOC;
                                            
                                            	/* Get the HEAD */
                                            	if ((flags & (SKB_ALLOC_FCLONE | SKB_ALLOC_NAPI)) == SKB_ALLOC_NAPI &&
                                            	    likely(node == NUMA_NO_NODE || node == numa_mem_id()))
                                            		skb = napi_skb_cache_get();
                                            	else
                                            		skb = kmem_cache_alloc_node(cache, gfp_mask & ~GFP_DMA, node);
                                            	if (unlikely(!skb))
                                            		return NULL;
                                            	prefetchw(skb);
                                            
                                            	/* We do our best to align skb_shared_info on a separate cache
                                            	 * line. It usually works because kmalloc(X > SMP_CACHE_BYTES) gives
                                            	 * aligned memory blocks, unless SLUB/SLAB debug is enabled.
                                            	 * Both skb->head and skb_shared_info are cache line aligned.
                                            	 */
                                            	data = kmalloc_reserve(&size, gfp_mask, node, &pfmemalloc);
                                            	if (unlikely(!data))
                                            		goto nodata;
                                            	/* kmalloc_size_roundup() might give us more room than requested.
                                            	 * Put skb_shared_info exactly at the end of allocated zone,
                                            	 * to allow max possible filling before reallocation.
                                            	 */
                                            	prefetchw(data + SKB_WITH_OVERHEAD(size));
                                            
                                            	/*
                                            	 * Only clear those fields we need to clear, not those that we will
                                            	 * actually initialise below. Hence, don't put any more fields after
                                            	 * the tail pointer in struct sk_buff!
                                            	 */
                                            	memset(skb, 0, offsetof(struct sk_buff, tail));
                                            	__build_skb_around(skb, data, size);
                                            	skb->pfmemalloc = pfmemalloc;
                                            
                                            	if (flags & SKB_ALLOC_FCLONE) {
                                            		struct sk_buff_fclones *fclones;
                                            
                                            		fclones = container_of(skb, struct sk_buff_fclones, skb1);
                                            
                                            		skb->fclone = SKB_FCLONE_ORIG;
                                            		refcount_set(&fclones->fclone_ref, 1);
                                            	}
                                            
                                            	return skb;
                                            
                                            nodata:
                                            	kmem_cache_free(cache, skb);
                                            	return NULL;
                                            }
                                            ```
                                            
                                        
                                        > 새로운 skb를 할당하는 함수. numa local한 캐시에서 sk_buff 할당 받은게 있다면 꺼내오고 아니면 새로 할당하게 된다. 이때, sk_buff는 초기에 CPU마다 대량으로 할당되어 있고, napi_skb_cache_get()을 통해 가져다가 쓸 수 있게 된다. 이후 kmalloc_reserve()를 호출하여 실제 data를 담을 공간을 할당 받는다.
                                        
                                    - __napi_build_skb(data, len)
                                        
                                        > 앞서 page_frag_alloc(), page_frag_alloc_1k()와 같은 함수들로부터 할당 받은 data 영역을 바탕으로 새롭게 skb를 만들어주는 함수이다. 내부 함수들의 호출로 sk_buff의 포인터들을 *data 영역에다가 가르키도록 셋팅하게 된다.  
                                        >   
                                        
                                - skb_record_rx_queue(skb, rx_ring→q_index)
                                    
                                    - 코드
                                        
                                        ```C
                                        static inline void skb_record_rx_queue(struct sk_buff *skb, u16 rx_queue)
                                        {
                                        	skb->queue_mapping = rx_queue + 1;
                                        }
                                        ```
                                        
                                    
                                    > skb에 해당하는 rx_queue를 기록하는 간단한 인라인 함수이다.
                                    
                            - ice_process_skb_fields(rx_ring, rx_desc, skb) ==(ice/ice_txrx_lib.c)==
                                
                                ---
                                
                                - 코드
                                    
                                    ```C
                                    /**
                                     * ice_process_skb_fields - Populate skb header fields from Rx descriptor
                                     * @rx_ring: Rx descriptor ring packet is being transacted on
                                     * @rx_desc: pointer to the EOP Rx descriptor
                                     * @skb: pointer to current skb being populated
                                     *
                                     * This function checks the ring, descriptor, and packet information in
                                     * order to populate the hash, checksum, VLAN, protocol, and
                                     * other fields within the skb.
                                     */
                                    void
                                    ice_process_skb_fields(struct ice_rx_ring *rx_ring,
                                    		       union ice_32b_rx_flex_desc *rx_desc,
                                    		       struct sk_buff *skb)
                                    {
                                    	u16 ptype = ice_get_ptype(rx_desc);
                                    
                                    	ice_rx_hash_to_skb(rx_ring, rx_desc, skb, ptype);
                                    
                                    	/* modifies the skb - consumes the enet header */
                                    	skb->protocol = eth_type_trans(skb, rx_ring->netdev);
                                    
                                    	ice_rx_csum(rx_ring, skb, rx_desc, ptype);
                                    
                                    	if (rx_ring->ptp_rx)
                                    		ice_ptp_rx_hwts_to_skb(rx_ring, rx_desc, skb);
                                    }
                                    ```
                                    
                                
                                > skb 필드를 처리하기 위해 사용되는 함수. ring, descriptor, packet informaion등의 정보를 체크하고, hash, checksum, VLAN, protocol 등의 필드를 채우게 된다.
                                
                            - ice_receive_skb(rx_ring, skb, vlan_tci) ==(ice/ice_txrx_lib.c)==
                                
                                ---
                                
                                - 코드
                                    
                                    ```C
                                    /**
                                     * ice_receive_skb - Send a completed packet up the stack
                                     * @rx_ring: Rx ring in play
                                     * @skb: packet to send up
                                     * @vlan_tci: VLAN TCI for packet
                                     *
                                     * This function sends the completed packet (via. skb) up the stack using
                                     * gro receive functions (with/without VLAN tag)
                                     */
                                    void
                                    ice_receive_skb(struct ice_rx_ring *rx_ring, struct sk_buff *skb, u16 vlan_tci)
                                    {
                                    	if ((vlan_tci & VLAN_VID_MASK) && rx_ring->vlan_proto)
                                    		__vlan_hwaccel_put_tag(skb, rx_ring->vlan_proto,
                                    				       vlan_tci);
                                    
                                    	napi_gro_receive(&rx_ring->q_vector->napi, skb);
                                    }
                                    ```
                                    
                                
                                > 처리가 완료된 패킷을 스택으로 올려보내주는 역할을 하게 된다. napi_gro_receive() 호출하는 함수이다. gro가 가능하다면, gro 처리를 하여 skb를 만들게 된다.
                                
                                - napi_gro_receive(&rx_ring→q_vector→napi, skb) ==(net/core/gro.c)==
                                    
                                    ---
                                    
                                    - 코드
                                        
                                        ```C
                                        gro_result_t napi_gro_receive(struct napi_struct *napi, struct sk_buff *skb)
                                        {
                                        	gro_result_t ret;
                                        
                                        	skb_mark_napi_id(skb, napi);
                                        	trace_napi_gro_receive_entry(skb);
                                        
                                        	skb_gro_reset_offset(skb, 0);
                                        
                                        	ret = napi_skb_finish(napi, skb, dev_gro_receive(napi, skb));
                                        	trace_napi_gro_receive_exit(ret);
                                        
                                        	return ret;
                                        }
                                        ```
                                        
                                    
                                    > dev_gro_receive() 함수에서 skb와 napi를 통해 실질적으로 gro를 처리하게 됨. 전후로 기타작업을 하는 함수들이 호출되고 있고, napi_skb_finish()함수를 dev_gro_receive() 함수의 return 값을 인수로 하여 호출하게 됨.
                                    
                                    - dev_gro_receive(napi, skb)
                                        
                                        ---
                                        
                                        - 코드
                                            
                                            ```C
                                            static enum gro_result dev_gro_receive(struct napi_struct *napi, struct sk_buff *skb)
                                            {
                                            	u32 bucket = skb_get_hash_raw(skb) & (GRO_HASH_BUCKETS - 1);
                                            	struct gro_list *gro_list = &napi->gro_hash[bucket];
                                            	struct list_head *head = &net_hotdata.offload_base;
                                            	struct packet_offload *ptype;
                                            	__be16 type = skb->protocol;
                                            	struct sk_buff *pp = NULL;
                                            	enum gro_result ret;
                                            	int same_flow;
                                            
                                            	if (netif_elide_gro(skb->dev))
                                            		goto normal;
                                            
                                            	gro_list_prepare(&gro_list->list, skb);
                                            
                                            	rcu_read_lock();
                                            	list_for_each_entry_rcu(ptype, head, list) {
                                            		if (ptype->type == type && ptype->callbacks.gro_receive)
                                            			goto found_ptype;
                                            	}
                                            	rcu_read_unlock();
                                            	goto normal;
                                            
                                            found_ptype:
                                            	skb_set_network_header(skb, skb_gro_offset(skb));
                                            	skb_reset_mac_len(skb);
                                            	BUILD_BUG_ON(sizeof_field(struct napi_gro_cb, zeroed) != sizeof(u32));
                                            	BUILD_BUG_ON(!IS_ALIGNED(offsetof(struct napi_gro_cb, zeroed),
                                            					sizeof(u32))); /* Avoid slow unaligned acc */
                                            	*(u32 *)&NAPI_GRO_CB(skb)->zeroed = 0;
                                            	NAPI_GRO_CB(skb)->flush = skb_has_frag_list(skb);
                                            	NAPI_GRO_CB(skb)->is_atomic = 1;
                                            	NAPI_GRO_CB(skb)->count = 1;
                                            	if (unlikely(skb_is_gso(skb))) {
                                            		NAPI_GRO_CB(skb)->count = skb_shinfo(skb)->gso_segs;
                                            		/* Only support TCP and non DODGY users. */
                                            		if (!skb_is_gso_tcp(skb) ||
                                            		    (skb_shinfo(skb)->gso_type & SKB_GSO_DODGY))
                                            			NAPI_GRO_CB(skb)->flush = 1;
                                            	}
                                            
                                            	/* Setup for GRO checksum validation */
                                            	switch (skb->ip_summed) {
                                            	case CHECKSUM_COMPLETE:
                                            		NAPI_GRO_CB(skb)->csum = skb->csum;
                                            		NAPI_GRO_CB(skb)->csum_valid = 1;
                                            		break;
                                            	case CHECKSUM_UNNECESSARY:
                                            		NAPI_GRO_CB(skb)->csum_cnt = skb->csum_level + 1;
                                            		break;
                                            	}
                                            
                                            	pp = INDIRECT_CALL_INET(ptype->callbacks.gro_receive,
                                            				ipv6_gro_receive, inet_gro_receive,
                                            				&gro_list->list, skb);  //IP별 함수가 호출되는 위치
                                            
                                            	rcu_read_unlock();
                                            
                                            	if (PTR_ERR(pp) == -EINPROGRESS) {
                                            		ret = GRO_CONSUMED;
                                            		goto ok;
                                            	}
                                            
                                            	same_flow = NAPI_GRO_CB(skb)->same_flow;
                                            	ret = NAPI_GRO_CB(skb)->free ? GRO_MERGED_FREE : GRO_MERGED;
                                            
                                            	if (pp) {
                                            		skb_list_del_init(pp);
                                            		napi_gro_complete(napi, pp);
                                            		gro_list->count--;
                                            	}
                                            
                                            	if (same_flow)
                                            		goto ok;
                                            
                                            	if (NAPI_GRO_CB(skb)->flush)
                                            		goto normal;
                                            
                                            	if (unlikely(gro_list->count >= MAX_GRO_SKBS))
                                            		gro_flush_oldest(napi, &gro_list->list);
                                            	else
                                            		gro_list->count++;
                                            
                                            	/* Must be called before setting NAPI_GRO_CB(skb)->{age|last} */
                                            	gro_try_pull_from_frag0(skb);
                                            	NAPI_GRO_CB(skb)->age = jiffies;
                                            	NAPI_GRO_CB(skb)->last = skb;
                                            	if (!skb_is_gso(skb))
                                            		skb_shinfo(skb)->gso_size = skb_gro_len(skb);
                                            	list_add(&skb->list, &gro_list->list);
                                            	ret = GRO_HELD;
                                            ok:
                                            	if (gro_list->count) {
                                            		if (!test_bit(bucket, &napi->gro_bitmask))
                                            			__set_bit(bucket, &napi->gro_bitmask);
                                            	} else if (test_bit(bucket, &napi->gro_bitmask)) {
                                            		__clear_bit(bucket, &napi->gro_bitmask);
                                            	}
                                            
                                            	return ret;
                                            
                                            normal:
                                            	ret = GRO_NORMAL;
                                            	gro_try_pull_from_frag0(skb);
                                            	goto ok;
                                            }
                                            ```
                                            
                                        
                                        > 실질적으로 gro를 처리하는 함수. 인수로 받아온 skb 포인터에다가 패킷 정보를 쌓기 시작함. 코드 중간에 pp = INDIRECT_CALL_INET(~~~)이라는 함수가 호출되는데 여기서 IPv4와 IPv6로 나뉘어 콜백이 되게 됨. gro_list도 함께 전달해주며, 이는 &napi→gro_hash[bucket]에서 가져온 결과임. 더 깊은 호출은 net/ipv4/af_inet.c의 inet_gro_receive() 함수를 보면 됨.
                                        
                                        - gro_list_prepare()
                                            
                                            - 코드
                                                
                                                ```C
                                                static void gro_list_prepare(const struct list_head *head,
                                                			     const struct sk_buff *skb)
                                                {
                                                	unsigned int maclen = skb->dev->hard_header_len;
                                                	u32 hash = skb_get_hash_raw(skb);
                                                	struct sk_buff *p;
                                                
                                                	list_for_each_entry(p, head, list) {
                                                		unsigned long diffs;
                                                
                                                		NAPI_GRO_CB(p)->flush = 0;
                                                
                                                		if (hash != skb_get_hash_raw(p)) {
                                                			NAPI_GRO_CB(p)->same_flow = 0;
                                                			continue;
                                                		}
                                                
                                                		diffs = (unsigned long)p->dev ^ (unsigned long)skb->dev;
                                                		diffs |= p->vlan_all ^ skb->vlan_all;
                                                		diffs |= skb_metadata_differs(p, skb);
                                                		if (maclen == ETH_HLEN)
                                                			diffs |= compare_ether_header(skb_mac_header(p),
                                                						      skb_mac_header(skb));
                                                		else if (!diffs)
                                                			diffs = memcmp(skb_mac_header(p),
                                                				       skb_mac_header(skb),
                                                				       maclen);
                                                
                                                		/* in most common scenarions 'slow_gro' is 0
                                                		 * otherwise we are already on some slower paths
                                                		 * either skip all the infrequent tests altogether or
                                                		 * avoid trying too hard to skip each of them individually
                                                		 */
                                                		if (!diffs && unlikely(skb->slow_gro | p->slow_gro)) {
                                                			diffs |= p->sk != skb->sk;
                                                			diffs |= skb_metadata_dst_cmp(p, skb);
                                                			diffs |= skb_get_nfct(p) ^ skb_get_nfct(skb);
                                                
                                                			diffs |= gro_list_prepare_tc_ext(skb, p, diffs);
                                                		}
                                                
                                                		NAPI_GRO_CB(p)->same_flow = !diffs;
                                                	}
                                                }
                                                ```
                                                
                                            
                                            > 리스트의 각각의 entry에 대하여 skb와 헤더, 즉 메타데이터 등을 비교함으로써 같은 flow인지 확인하는 과정을 거친다. 주로 XOR 함수를 통해 그 결과를 diff에 저장하여, 만약 다르다면 1, 같다면 0이 저장 될 것이다. 이를 다시 각각의 skb에 same_flow에 !diff로 저장하게 된다.
                                            
                                        - inet_gro_receive() ==(net/ipv4/af_inet.c)==
                                            
                                            - 코드
                                                
                                                ```C
                                                struct sk_buff *inet_gro_receive(struct list_head *head, struct sk_buff *skb)
                                                {
                                                	const struct net_offload *ops;
                                                	struct sk_buff *pp = NULL;
                                                	const struct iphdr *iph;
                                                	struct sk_buff *p;
                                                	unsigned int hlen;
                                                	unsigned int off;
                                                	unsigned int id;
                                                	int flush = 1;
                                                	int proto;
                                                
                                                	off = skb_gro_offset(skb);
                                                	hlen = off + sizeof(*iph);
                                                	iph = skb_gro_header(skb, hlen, off);
                                                	if (unlikely(!iph))
                                                		goto out;
                                                
                                                	proto = iph->protocol;
                                                
                                                	ops = rcu_dereference(inet_offloads[proto]);
                                                	if (!ops || !ops->callbacks.gro_receive)
                                                		goto out;
                                                
                                                	if (*(u8 *)iph != 0x45)
                                                		goto out;
                                                
                                                	if (ip_is_fragment(iph))
                                                		goto out;
                                                
                                                	if (unlikely(ip_fast_csum((u8 *)iph, 5)))
                                                		goto out;
                                                
                                                	NAPI_GRO_CB(skb)->proto = proto;
                                                	id = ntohl(*(__be32 *)&iph->id);
                                                	flush = (u16)((ntohl(*(__be32 *)iph) ^ skb_gro_len(skb)) | (id & ~IP_DF));
                                                	id >>= 16;
                                                
                                                	list_for_each_entry(p, head, list) {
                                                		struct iphdr *iph2;
                                                		u16 flush_id;
                                                
                                                		if (!NAPI_GRO_CB(p)->same_flow)
                                                			continue;
                                                
                                                		iph2 = (struct iphdr *)(p->data + off);
                                                		/* The above works because, with the exception of the top
                                                		 * (inner most) layer, we only aggregate pkts with the same
                                                		 * hdr length so all the hdrs we'll need to verify will start
                                                		 * at the same offset.
                                                		 */
                                                		if ((iph->protocol ^ iph2->protocol) |
                                                		    ((__force u32)iph->saddr ^ (__force u32)iph2->saddr) |
                                                		    ((__force u32)iph->daddr ^ (__force u32)iph2->daddr)) {
                                                			NAPI_GRO_CB(p)->same_flow = 0;
                                                			continue;
                                                		}
                                                
                                                		/* All fields must match except length and checksum. */
                                                		NAPI_GRO_CB(p)->flush |=
                                                			(iph->ttl ^ iph2->ttl) |
                                                			(iph->tos ^ iph2->tos) |
                                                			((iph->frag_off ^ iph2->frag_off) & htons(IP_DF));
                                                
                                                		NAPI_GRO_CB(p)->flush |= flush;
                                                
                                                		/* We need to store of the IP ID check to be included later
                                                		 * when we can verify that this packet does in fact belong
                                                		 * to a given flow.
                                                		 */
                                                		flush_id = (u16)(id - ntohs(iph2->id));
                                                
                                                		/* This bit of code makes it much easier for us to identify
                                                		 * the cases where we are doing atomic vs non-atomic IP ID
                                                		 * checks.  Specifically an atomic check can return IP ID
                                                		 * values 0 - 0xFFFF, while a non-atomic check can only
                                                		 * return 0 or 0xFFFF.
                                                		 */
                                                		if (!NAPI_GRO_CB(p)->is_atomic ||
                                                		    !(iph->frag_off & htons(IP_DF))) {
                                                			flush_id ^= NAPI_GRO_CB(p)->count;
                                                			flush_id = flush_id ? 0xFFFF : 0;
                                                		}
                                                
                                                		/* If the previous IP ID value was based on an atomic
                                                		 * datagram we can overwrite the value and ignore it.
                                                		 */
                                                		if (NAPI_GRO_CB(skb)->is_atomic)
                                                			NAPI_GRO_CB(p)->flush_id = flush_id;
                                                		else
                                                			NAPI_GRO_CB(p)->flush_id |= flush_id;
                                                	}
                                                
                                                	NAPI_GRO_CB(skb)->is_atomic = !!(iph->frag_off & htons(IP_DF));
                                                	NAPI_GRO_CB(skb)->flush |= flush;
                                                	skb_set_network_header(skb, off);
                                                	/* The above will be needed by the transport layer if there is one
                                                	 * immediately following this IP hdr.
                                                	 */
                                                	NAPI_GRO_CB(skb)->inner_network_offset = off;
                                                
                                                	/* Note : No need to call skb_gro_postpull_rcsum() here,
                                                	 * as we already checked checksum over ipv4 header was 0
                                                	 */
                                                	skb_gro_pull(skb, sizeof(*iph));
                                                	skb_set_transport_header(skb, skb_gro_offset(skb));
                                                
                                                	pp = indirect_call_gro_receive(tcp4_gro_receive, udp4_gro_receive,
                                                				       ops->callbacks.gro_receive, head, skb);
                                                
                                                out:
                                                	skb_gro_flush_final(skb, pp, flush);
                                                
                                                	return pp;
                                                }
                                                ```
                                                
                                            
                                            > ipv4 에서 gro receive를 처리하는 함수이다. indirect call을 통해서 호출되게 되며, 기존의 gro_list와 병합하려는 패킷이 서로 맞는지 L3에서의 필요한 처리들과 검사들을 진행하게 되며, 여기서는 출발지 주소와 목적지 주소가 같은지, ttl과 tos (time to live, type of service)등이 같은지 여부를 통해 병합해도 되는 패킷인지 검사하게 된다. 이 때 병합하려는 패킷들의 모든 리스트를 검사하게 된다. 그 다음 L3 헤더 길이 만큼 skb_gro_pull을 통해 data_offset에다가 그 길이만큼 더하여 다음 L4 헤더를 볼 수 있게 한다.
                                            
                                            - tcp4_gro_receive() ==(net/ipv4/tcp_offload.c)==
                                                
                                                - 코드
                                                    
                                                    ```C
                                                    INDIRECT_CALLABLE_SCOPE
                                                    struct sk_buff *tcp4_gro_receive(struct list_head *head, struct sk_buff *skb)
                                                    {
                                                    	/* Don't bother verifying checksum if we're going to flush anyway. */
                                                    	if (!NAPI_GRO_CB(skb)->flush &&
                                                    	    skb_gro_checksum_validate(skb, IPPROTO_TCP,
                                                    				      inet_gro_compute_pseudo)) {
                                                    		NAPI_GRO_CB(skb)->flush = 1;
                                                    		return NULL;
                                                    	}
                                                    
                                                    	return tcp_gro_receive(head, skb);
                                                    }
                                                    ```
                                                    
                                                
                                                > 간단하게 flush가 필요한지 여부와 checksum이 유효한지 여부를 따져서 더 processing이 진행되는지 확인하는 If문이 하나가 있고, 아니라면 tcp_gro_receive()함수를 호출하여 계속 진행하게 된다.
                                                
                                                - tcp_gro_receive(head, skb)
                                                    
                                                    - 코드
                                                        
                                                        ```C
                                                        struct sk_buff *tcp_gro_receive(struct list_head *head, struct sk_buff *skb)
                                                        {
                                                        	struct sk_buff *pp = NULL;
                                                        	struct sk_buff *p;
                                                        	struct tcphdr *th;
                                                        	struct tcphdr *th2;
                                                        	unsigned int len;
                                                        	unsigned int thlen;
                                                        	__be32 flags;
                                                        	unsigned int mss = 1;
                                                        	unsigned int hlen;
                                                        	unsigned int off;
                                                        	int flush = 1;
                                                        	int i;
                                                        
                                                        	off = skb_gro_offset(skb);
                                                        	hlen = off + sizeof(*th);
                                                        	th = skb_gro_header(skb, hlen, off);
                                                        	if (unlikely(!th))
                                                        		goto out;
                                                        
                                                        	thlen = th->doff * 4;
                                                        	if (thlen < sizeof(*th))
                                                        		goto out;
                                                        
                                                        	hlen = off + thlen;
                                                        	if (!skb_gro_may_pull(skb, hlen)) {
                                                        		th = skb_gro_header_slow(skb, hlen, off);
                                                        		if (unlikely(!th))
                                                        			goto out;
                                                        	}
                                                        
                                                        	skb_gro_pull(skb, thlen);
                                                        
                                                        	len = skb_gro_len(skb);
                                                        	flags = tcp_flag_word(th);
                                                        
                                                        	list_for_each_entry(p, head, list) {
                                                        		if (!NAPI_GRO_CB(p)->same_flow)
                                                        			continue;
                                                        
                                                        		th2 = tcp_hdr(p);
                                                        
                                                        		if (*(u32 *)&th->source ^ *(u32 *)&th2->source) {
                                                        			NAPI_GRO_CB(p)->same_flow = 0;
                                                        			continue;
                                                        		}
                                                        
                                                        		goto found;
                                                        	}
                                                        	p = NULL;
                                                        	goto out_check_final;
                                                        
                                                        found:
                                                        	/* Include the IP ID check below from the inner most IP hdr */
                                                        	flush = NAPI_GRO_CB(p)->flush;
                                                        	flush |= (__force int)(flags & TCP_FLAG_CWR);
                                                        	flush |= (__force int)((flags ^ tcp_flag_word(th2)) &
                                                        		  ~(TCP_FLAG_CWR | TCP_FLAG_FIN | TCP_FLAG_PSH));
                                                        	flush |= (__force int)(th->ack_seq ^ th2->ack_seq);
                                                        	for (i = sizeof(*th); i < thlen; i += 4)
                                                        		flush |= *(u32 *)((u8 *)th + i) ^
                                                        			 *(u32 *)((u8 *)th2 + i);
                                                        
                                                        	/* When we receive our second frame we can made a decision on if we
                                                        	 * continue this flow as an atomic flow with a fixed ID or if we use
                                                        	 * an incrementing ID.
                                                        	 */
                                                        	if (NAPI_GRO_CB(p)->flush_id != 1 ||
                                                        	    NAPI_GRO_CB(p)->count != 1 ||
                                                        	    !NAPI_GRO_CB(p)->is_atomic)
                                                        		flush |= NAPI_GRO_CB(p)->flush_id;
                                                        	else
                                                        		NAPI_GRO_CB(p)->is_atomic = false;
                                                        
                                                        	mss = skb_shinfo(p)->gso_size;
                                                        
                                                        	/* If skb is a GRO packet, make sure its gso_size matches prior packet mss.
                                                        	 * If it is a single frame, do not aggregate it if its length
                                                        	 * is bigger than our mss.
                                                        	 */
                                                        	if (unlikely(skb_is_gso(skb)))
                                                        		flush |= (mss != skb_shinfo(skb)->gso_size);
                                                        	else
                                                        		flush |= (len - 1) >= mss;
                                                        
                                                        	flush |= (ntohl(th2->seq) + skb_gro_len(p)) ^ ntohl(th->seq);
                                                        \#ifdef CONFIG_TLS_DEVICE
                                                        	flush |= p->decrypted ^ skb->decrypted;
                                                        \#endif
                                                        
                                                        	if (flush || skb_gro_receive(p, skb)) {
                                                        		mss = 1;
                                                        		goto out_check_final;
                                                        	}
                                                        
                                                        	tcp_flag_word(th2) |= flags & (TCP_FLAG_FIN | TCP_FLAG_PSH);
                                                        
                                                        out_check_final:
                                                        	/* Force a flush if last segment is smaller than mss. */
                                                        	if (unlikely(skb_is_gso(skb)))
                                                        		flush = len != NAPI_GRO_CB(skb)->count * skb_shinfo(skb)->gso_size;
                                                        	else
                                                        		flush = len < mss;
                                                        
                                                        	flush |= (__force int)(flags & (TCP_FLAG_URG | TCP_FLAG_PSH |
                                                        					TCP_FLAG_RST | TCP_FLAG_SYN |
                                                        					TCP_FLAG_FIN));
                                                        
                                                        	if (p && (!NAPI_GRO_CB(skb)->same_flow || flush))
                                                        		pp = p;
                                                        
                                                        out:
                                                        	NAPI_GRO_CB(skb)->flush |= (flush != 0);
                                                        
                                                        	return pp;
                                                        }
                                                        ```
                                                        
                                                    
                                                    > L4에서 같은 flow인지 확인하는 부분이 들어있다. 함께 들어온 gro_list의 각각의 패킷들에 대하여 같은 포트에서 온건지 확인하여 같은 포트인것이 확인 되면, 바로 found label로 가게 된다. 해당하는 패킷과 찾은 skb를 병합하기 위해 skb_gro_receive()를 실행하게 된다.
                                                    
                                                    - skb_gro_receive(p, skb) ==(net/core/gro.c)==
                                                        
                                                        - 코드
                                                            
                                                            ```C
                                                            int skb_gro_receive(struct sk_buff *p, struct sk_buff *skb)
                                                            {
                                                            	struct skb_shared_info *pinfo, *skbinfo = skb_shinfo(skb);
                                                            	unsigned int offset = skb_gro_offset(skb);
                                                            	unsigned int headlen = skb_headlen(skb);
                                                            	unsigned int len = skb_gro_len(skb);
                                                            	unsigned int delta_truesize;
                                                            	unsigned int gro_max_size;
                                                            	unsigned int new_truesize;
                                                            	struct sk_buff *lp;
                                                            	int segs;
                                                            
                                                            	/* Do not splice page pool based packets w/ non-page pool
                                                            	 * packets. This can result in reference count issues as page
                                                            	 * pool pages will not decrement the reference count and will
                                                            	 * instead be immediately returned to the pool or have frag
                                                            	 * count decremented.
                                                            	 */
                                                            	if (p->pp_recycle != skb->pp_recycle)
                                                            		return -ETOOMANYREFS;
                                                            
                                                            	/* pairs with WRITE_ONCE() in netif_set_gro(_ipv4)_max_size() */
                                                            	gro_max_size = p->protocol == htons(ETH_P_IPV6) ?
                                                            			READ_ONCE(p->dev->gro_max_size) :
                                                            			READ_ONCE(p->dev->gro_ipv4_max_size);
                                                            
                                                            	if (unlikely(p->len + len >= gro_max_size || NAPI_GRO_CB(skb)->flush))
                                                            		return -E2BIG;
                                                            
                                                            	if (unlikely(p->len + len >= GRO_LEGACY_MAX_SIZE)) {
                                                            		if (NAPI_GRO_CB(skb)->proto != IPPROTO_TCP ||
                                                            		    (p->protocol == htons(ETH_P_IPV6) &&
                                                            		     skb_headroom(p) < sizeof(struct hop_jumbo_hdr)) ||
                                                            		    p->encapsulation)
                                                            			return -E2BIG;
                                                            	}
                                                            
                                                            	segs = NAPI_GRO_CB(skb)->count;
                                                            	lp = NAPI_GRO_CB(p)->last;
                                                            	pinfo = skb_shinfo(lp);
                                                            
                                                            	if (headlen <= offset) {
                                                            		skb_frag_t *frag;
                                                            		skb_frag_t *frag2;
                                                            		int i = skbinfo->nr_frags;
                                                            		int nr_frags = pinfo->nr_frags + i;
                                                            
                                                            		if (nr_frags > MAX_SKB_FRAGS)
                                                            			goto merge;
                                                            
                                                            		offset -= headlen;
                                                            		pinfo->nr_frags = nr_frags;
                                                            		skbinfo->nr_frags = 0;
                                                            
                                                            		frag = pinfo->frags + nr_frags;
                                                            		frag2 = skbinfo->frags + i;
                                                            		do {
                                                            			*--frag = *--frag2;
                                                            		} while (--i); //여기가 기존 frag에 역순으로 병합하는 부분임
                                                            
                                                            		skb_frag_off_add(frag, offset);
                                                            		skb_frag_size_sub(frag, offset);
                                                            
                                                            		/* all fragments truesize : remove (head size + sk_buff) */
                                                            		new_truesize = SKB_TRUESIZE(skb_end_offset(skb));
                                                            		delta_truesize = skb->truesize - new_truesize;
                                                            
                                                            		skb->truesize = new_truesize;
                                                            		skb->len -= skb->data_len;
                                                            		skb->data_len = 0;
                                                            
                                                            		NAPI_GRO_CB(skb)->free = NAPI_GRO_FREE;
                                                            		goto done;
                                                            	} else if (skb->head_frag) {
                                                            		int nr_frags = pinfo->nr_frags;
                                                            		skb_frag_t *frag = pinfo->frags + nr_frags;
                                                            		struct page *page = virt_to_head_page(skb->head);
                                                            		unsigned int first_size = headlen - offset;
                                                            		unsigned int first_offset;
                                                            
                                                            		if (nr_frags + 1 + skbinfo->nr_frags > MAX_SKB_FRAGS)
                                                            			goto merge;
                                                            
                                                            		first_offset = skb->data -
                                                            			       (unsigned char *)page_address(page) +
                                                            			       offset;
                                                            
                                                            		pinfo->nr_frags = nr_frags + 1 + skbinfo->nr_frags;
                                                            
                                                            		skb_frag_fill_page_desc(frag, page, first_offset, first_size);
                                                            
                                                            		memcpy(frag + 1, skbinfo->frags, sizeof(*frag) * skbinfo->nr_frags);
                                                            		/* We dont need to clear skbinfo->nr_frags here */
                                                            
                                                            		new_truesize = SKB_DATA_ALIGN(sizeof(struct sk_buff));
                                                            		delta_truesize = skb->truesize - new_truesize;
                                                            		skb->truesize = new_truesize;
                                                            		NAPI_GRO_CB(skb)->free = NAPI_GRO_FREE_STOLEN_HEAD;
                                                            		goto done;
                                                            	}
                                                            
                                                            merge:
                                                            	/* sk ownership - if any - completely transferred to the aggregated packet */
                                                            	skb->destructor = NULL;
                                                            	skb->sk = NULL;
                                                            	delta_truesize = skb->truesize;
                                                            	if (offset > headlen) {
                                                            		unsigned int eat = offset - headlen;
                                                            
                                                            		skb_frag_off_add(&skbinfo->frags[0], eat);
                                                            		skb_frag_size_sub(&skbinfo->frags[0], eat);
                                                            		skb->data_len -= eat;
                                                            		skb->len -= eat;
                                                            		offset = headlen;
                                                            	}
                                                            
                                                            	__skb_pull(skb, offset);
                                                            
                                                            	if (NAPI_GRO_CB(p)->last == p)
                                                            		skb_shinfo(p)->frag_list = skb;
                                                            	else
                                                            		NAPI_GRO_CB(p)->last->next = skb;
                                                            	NAPI_GRO_CB(p)->last = skb;
                                                            	__skb_header_release(skb);
                                                            	lp = p;
                                                            
                                                            done:
                                                            	NAPI_GRO_CB(p)->count += segs;
                                                            	p->data_len += len;
                                                            	p->truesize += delta_truesize;
                                                            	p->len += len;
                                                            	if (lp != p) {
                                                            		lp->data_len += len;
                                                            		lp->truesize += delta_truesize;
                                                            		lp->len += len;
                                                            	}
                                                            	NAPI_GRO_CB(skb)->same_flow = 1;
                                                            	return 0;
                                                            }
                                                            ```
                                                            
                                                        
                                                        > 기존의 패킷에다가 skb를 새로 덧붙이게 된다. skb_shared_info에다가 frag를 붙이게 되고, 만약 여유공간이 없다면 새로 만들게 된다.
                                                        
                                            - udp4_gro_receive() ==(net/ipv4/udp_offload.c)==
                                                
                                                - 코드
                                                    
                                                    ```C
                                                    INDIRECT_CALLABLE_SCOPE
                                                    struct sk_buff *udp4_gro_receive(struct list_head *head, struct sk_buff *skb)
                                                    {
                                                    	struct udphdr *uh = udp_gro_udphdr(skb);
                                                    	struct sock *sk = NULL;
                                                    	struct sk_buff *pp;
                                                    
                                                    	if (unlikely(!uh))
                                                    		goto flush;
                                                    
                                                    	/* Don't bother verifying checksum if we're going to flush anyway. */
                                                    	if (NAPI_GRO_CB(skb)->flush)
                                                    		goto skip;
                                                    
                                                    	if (skb_gro_checksum_validate_zero_check(skb, IPPROTO_UDP, uh->check,
                                                    						 inet_gro_compute_pseudo))
                                                    		goto flush;
                                                    	else if (uh->check)
                                                    		skb_gro_checksum_try_convert(skb, IPPROTO_UDP,
                                                    					     inet_gro_compute_pseudo);
                                                    skip:
                                                    	NAPI_GRO_CB(skb)->is_ipv6 = 0;
                                                    
                                                    	if (static_branch_unlikely(&udp_encap_needed_key))
                                                    		sk = udp4_gro_lookup_skb(skb, uh->source, uh->dest);
                                                    
                                                    	pp = udp_gro_receive(head, skb, uh, sk);
                                                    	return pp;
                                                    
                                                    flush:
                                                    	NAPI_GRO_CB(skb)->flush = 1;
                                                    	return NULL;
                                                    }
                                                    ```
                                                    
                                                
                                                > udp가 목적지 포트를 찾아 소켓을 받아서 이를 udp_gro_receive()에 인수로 같이 전달하는 부분이 추가 되었다.
                                                
                                                - udp_gro_receive()
                                                    
                                                    - 코드
                                                        
                                                        ```C
                                                        struct sk_buff *udp_gro_receive(struct list_head *head, struct sk_buff *skb,
                                                        				struct udphdr *uh, struct sock *sk)
                                                        {
                                                        	struct sk_buff *pp = NULL;
                                                        	struct sk_buff *p;
                                                        	struct udphdr *uh2;
                                                        	unsigned int off = skb_gro_offset(skb);
                                                        	int flush = 1;
                                                        
                                                        	/* We can do L4 aggregation only if the packet can't land in a tunnel
                                                        	 * otherwise we could corrupt the inner stream. Detecting such packets
                                                        	 * cannot be foolproof and the aggregation might still happen in some
                                                        	 * cases. Such packets should be caught in udp_unexpected_gso later.
                                                        	 */
                                                        	NAPI_GRO_CB(skb)->is_flist = 0;
                                                        	if (!sk || !udp_sk(sk)->gro_receive) {
                                                        		/* If the packet was locally encapsulated in a UDP tunnel that
                                                        		 * wasn't detected above, do not GRO.
                                                        		 */
                                                        		if (skb->encapsulation)
                                                        			goto out;
                                                        
                                                        		if (skb->dev->features & NETIF_F_GRO_FRAGLIST)
                                                        			NAPI_GRO_CB(skb)->is_flist = sk ? !udp_test_bit(GRO_ENABLED, sk) : 1;
                                                        
                                                        		if ((!sk && (skb->dev->features & NETIF_F_GRO_UDP_FWD)) ||
                                                        		    (sk && udp_test_bit(GRO_ENABLED, sk)) || NAPI_GRO_CB(skb)->is_flist)
                                                        			return call_gro_receive(udp_gro_receive_segment, head, skb);
                                                        
                                                        		/* no GRO, be sure flush the current packet */
                                                        		goto out;
                                                        	}
                                                        
                                                        	if (NAPI_GRO_CB(skb)->encap_mark ||
                                                        	    (uh->check && skb->ip_summed != CHECKSUM_PARTIAL &&
                                                        	     NAPI_GRO_CB(skb)->csum_cnt == 0 &&
                                                        	     !NAPI_GRO_CB(skb)->csum_valid))
                                                        		goto out;
                                                        
                                                        	/* mark that this skb passed once through the tunnel gro layer */
                                                        	NAPI_GRO_CB(skb)->encap_mark = 1;
                                                        
                                                        	flush = 0;
                                                        
                                                        	list_for_each_entry(p, head, list) {
                                                        		if (!NAPI_GRO_CB(p)->same_flow)
                                                        			continue;
                                                        
                                                        		uh2 = (struct udphdr   *)(p->data + off);
                                                        
                                                        		/* Match ports and either checksums are either both zero
                                                        		 * or nonzero.
                                                        		 */
                                                        		if ((*(u32 *)&uh->source != *(u32 *)&uh2->source) ||
                                                        		    (!uh->check ^ !uh2->check)) {
                                                        			NAPI_GRO_CB(p)->same_flow = 0;
                                                        			continue;
                                                        		}
                                                        	}
                                                        
                                                        	skb_gro_pull(skb, sizeof(struct udphdr)); /* pull encapsulating udp header */
                                                        	skb_gro_postpull_rcsum(skb, uh, sizeof(struct udphdr));
                                                        	pp = call_gro_receive_sk(udp_sk(sk)->gro_receive, sk, head, skb);
                                                        
                                                        out:
                                                        	skb_gro_flush_final(skb, pp, flush);
                                                        	return pp;
                                                        }
                                                        ```
                                                        
                                                    
                                                    > 여기서는 직접적인 패킷 처리가 이루어지지 않고, 함수포인터를 통해 이루어지게 된다. call_gro_receive() 함수 혹은 call_gro_receive_sk() 함수를 통해 이루어지게 되는데, 앞의 함수는 udp_gro_receive_segment()라는 함수를 호출하게 되고 뒤의 함수는 udp_sk타입으로 본 소켓에서 gro_receive라는 멤버 변수인 함수 포인터로 실행되게 된다.
                                                    
                                                    - udp_gro_receive_segment()
                                                        
                                                        - 코드
                                                            
                                                            ```C
                                                            \#define UDP_GRO_CNT_MAX 64
                                                            static struct sk_buff *udp_gro_receive_segment(struct list_head *head,
                                                            					       struct sk_buff *skb)
                                                            {
                                                            	struct udphdr *uh = udp_gro_udphdr(skb);
                                                            	struct sk_buff *pp = NULL;
                                                            	struct udphdr *uh2;
                                                            	struct sk_buff *p;
                                                            	unsigned int ulen;
                                                            	int ret = 0;
                                                            	int flush;
                                                            
                                                            	/* requires non zero csum, for symmetry with GSO */
                                                            	if (!uh->check) {
                                                            		NAPI_GRO_CB(skb)->flush = 1;
                                                            		return NULL;
                                                            	}
                                                            
                                                            	/* Do not deal with padded or malicious packets, sorry ! */
                                                            	ulen = ntohs(uh->len);
                                                            	if (ulen <= sizeof(*uh) || ulen != skb_gro_len(skb)) {
                                                            		NAPI_GRO_CB(skb)->flush = 1;
                                                            		return NULL;
                                                            	}
                                                            	/* pull encapsulating udp header */
                                                            	skb_gro_pull(skb, sizeof(struct udphdr));
                                                            
                                                            	list_for_each_entry(p, head, list) {
                                                            		if (!NAPI_GRO_CB(p)->same_flow)
                                                            			continue;
                                                            
                                                            		uh2 = udp_hdr(p);
                                                            
                                                            		/* Match ports only, as csum is always non zero */
                                                            		if ((*(u32 *)&uh->source != *(u32 *)&uh2->source)) {
                                                            			NAPI_GRO_CB(p)->same_flow = 0;
                                                            			continue;
                                                            		}
                                                            
                                                            		if (NAPI_GRO_CB(skb)->is_flist != NAPI_GRO_CB(p)->is_flist) {
                                                            			NAPI_GRO_CB(skb)->flush = 1;
                                                            			return p;
                                                            		}
                                                            
                                                            		flush = NAPI_GRO_CB(p)->flush;
                                                            
                                                            		if (NAPI_GRO_CB(p)->flush_id != 1 ||
                                                            		    NAPI_GRO_CB(p)->count != 1 ||
                                                            		    !NAPI_GRO_CB(p)->is_atomic)
                                                            			flush |= NAPI_GRO_CB(p)->flush_id;
                                                            		else
                                                            			NAPI_GRO_CB(p)->is_atomic = false;
                                                            
                                                            		/* Terminate the flow on len mismatch or if it grow "too much".
                                                            		 * Under small packet flood GRO count could elsewhere grow a lot
                                                            		 * leading to excessive truesize values.
                                                            		 * On len mismatch merge the first packet shorter than gso_size,
                                                            		 * otherwise complete the GRO packet.
                                                            		 */
                                                            		if (ulen > ntohs(uh2->len) || flush) {
                                                            			pp = p;
                                                            		} else {
                                                            			if (NAPI_GRO_CB(skb)->is_flist) {
                                                            				if (!pskb_may_pull(skb, skb_gro_offset(skb))) {
                                                            					NAPI_GRO_CB(skb)->flush = 1;
                                                            					return NULL;
                                                            				}
                                                            				if ((skb->ip_summed != p->ip_summed) ||
                                                            				    (skb->csum_level != p->csum_level)) {
                                                            					NAPI_GRO_CB(skb)->flush = 1;
                                                            					return NULL;
                                                            				}
                                                            				ret = skb_gro_receive_list(p, skb);
                                                            			} else {
                                                            				skb_gro_postpull_rcsum(skb, uh,
                                                            						       sizeof(struct udphdr));
                                                            
                                                            				ret = skb_gro_receive(p, skb);
                                                            			}
                                                            		}
                                                            
                                                            		if (ret || ulen != ntohs(uh2->len) ||
                                                            		    NAPI_GRO_CB(p)->count >= UDP_GRO_CNT_MAX)
                                                            			pp = p;
                                                            
                                                            		return pp;
                                                            	}
                                                            
                                                            	/* mismatch, but we never need to flush */
                                                            	return NULL;
                                                            }
                                                            ```
                                                            
                                                        
                                                        > 여기서 다시 list_for_each_entry() 매크로를 통해 list의 모든 패킷을 검사하게 된다. 그렇게 같은 flow인지 확인하여 skb_gro_receive_list() 혹은 skb_gro_receive() 함수를 호출하게 됨으로써, 결국에는 tcp든 udp든 ipv4든 ipv6든 헤더만 확인하고 결국 skb_gro_receive() 함수를 호출하여 gro를 수행하게 된다는 것을 알 수 있다.
                                                        
                                        - ipv6_gro_receive() ==(net/ipv6/ip6_offload.c)==
                                            
                                            - 코드
                                                
                                                ```undefined
                                                INDIRECT_CALLABLE_SCOPE struct sk_buff *ipv6_gro_receive(struct list_head *head,
                                                							 struct sk_buff *skb)
                                                {
                                                	const struct net_offload *ops;
                                                	struct sk_buff *pp = NULL;
                                                	struct sk_buff *p;
                                                	struct ipv6hdr *iph;
                                                	unsigned int nlen;
                                                	unsigned int hlen;
                                                	unsigned int off;
                                                	u16 flush = 1;
                                                	int proto;
                                                
                                                	off = skb_gro_offset(skb);
                                                	hlen = off + sizeof(*iph);
                                                	iph = skb_gro_header(skb, hlen, off);
                                                	if (unlikely(!iph))
                                                		goto out;
                                                
                                                	skb_set_network_header(skb, off);
                                                	NAPI_GRO_CB(skb)->inner_network_offset = off;
                                                
                                                	flush += ntohs(iph->payload_len) != skb->len - hlen;
                                                
                                                	proto = iph->nexthdr;
                                                	ops = rcu_dereference(inet6_offloads[proto]);
                                                	if (!ops || !ops->callbacks.gro_receive) {
                                                		proto = ipv6_gro_pull_exthdrs(skb, hlen, proto);
                                                
                                                		ops = rcu_dereference(inet6_offloads[proto]);
                                                		if (!ops || !ops->callbacks.gro_receive)
                                                			goto out;
                                                
                                                		iph = skb_gro_network_header(skb);
                                                	} else {
                                                		skb_gro_pull(skb, sizeof(*iph));
                                                	}
                                                
                                                	skb_set_transport_header(skb, skb_gro_offset(skb));
                                                
                                                	NAPI_GRO_CB(skb)->proto = proto;
                                                
                                                	flush--;
                                                	nlen = skb_network_header_len(skb);
                                                
                                                	list_for_each_entry(p, head, list) {
                                                		const struct ipv6hdr *iph2;
                                                		__be32 first_word; /* <Version:4><Traffic_Class:8><Flow_Label:20> */
                                                
                                                		if (!NAPI_GRO_CB(p)->same_flow)
                                                			continue;
                                                
                                                		iph2 = (struct ipv6hdr *)(p->data + off);
                                                		first_word = *(__be32 *)iph ^ *(__be32 *)iph2;
                                                
                                                		/* All fields must match except length and Traffic Class.
                                                		 * XXX skbs on the gro_list have all been parsed and pulled
                                                		 * already so we don't need to compare nlen
                                                		 * (nlen != (sizeof(*iph2) + ipv6_exthdrs_len(iph2, &ops)))
                                                		 * memcmp() alone below is sufficient, right?
                                                		 */
                                                		 if ((first_word & htonl(0xF00FFFFF)) ||
                                                		     !ipv6_addr_equal(&iph->saddr, &iph2->saddr) ||
                                                		     !ipv6_addr_equal(&iph->daddr, &iph2->daddr) ||
                                                		     iph->nexthdr != iph2->nexthdr) {
                                                not_same_flow:
                                                			NAPI_GRO_CB(p)->same_flow = 0;
                                                			continue;
                                                		}
                                                		if (unlikely(nlen > sizeof(struct ipv6hdr))) {
                                                			if (memcmp(iph + 1, iph2 + 1,
                                                				   nlen - sizeof(struct ipv6hdr)))
                                                				goto not_same_flow;
                                                		}
                                                		/* flush if Traffic Class fields are different */
                                                		NAPI_GRO_CB(p)->flush |= !!((first_word & htonl(0x0FF00000)) |
                                                			(__force __be32)(iph->hop_limit ^ iph2->hop_limit));
                                                		NAPI_GRO_CB(p)->flush |= flush;
                                                
                                                		/* If the previous IP ID value was based on an atomic
                                                		 * datagram we can overwrite the value and ignore it.
                                                		 */
                                                		if (NAPI_GRO_CB(skb)->is_atomic)
                                                			NAPI_GRO_CB(p)->flush_id = 0;
                                                	}
                                                
                                                	NAPI_GRO_CB(skb)->is_atomic = true;
                                                	NAPI_GRO_CB(skb)->flush |= flush;
                                                
                                                	skb_gro_postpull_rcsum(skb, iph, nlen);
                                                
                                                	pp = indirect_call_gro_receive_l4(tcp6_gro_receive, udp6_gro_receive,
                                                					 ops->callbacks.gro_receive, head, skb);
                                                
                                                out:
                                                	skb_gro_flush_final(skb, pp, flush);
                                                
                                                	return pp;
                                                }
                                                ```
                                                
                                            
                                            > Ipv6에서 L3를 검사하는 헤더이다. 앞서 inet_gro_receive()와 마찬가지로 모든 entry에 대하여 같은 L3 flow인지 확인하게 된다.
                                            
                                            - tcp6_gro_receive() ==(net/ipv6/tcpv6_offload.c)==
                                                
                                                - 코드
                                                    
                                                    ```C
                                                    INDIRECT_CALLABLE_SCOPE
                                                    struct sk_buff *tcp6_gro_receive(struct list_head *head, struct sk_buff *skb)
                                                    {
                                                    	/* Don't bother verifying checksum if we're going to flush anyway. */
                                                    	if (!NAPI_GRO_CB(skb)->flush &&
                                                    	    skb_gro_checksum_validate(skb, IPPROTO_TCP,
                                                    				      ip6_gro_compute_pseudo)) {
                                                    		NAPI_GRO_CB(skb)->flush = 1;
                                                    		return NULL;
                                                    	}
                                                    
                                                    	return tcp_gro_receive(head, skb);
                                                    }
                                                    ```
                                                    
                                                
                                                > 결국 다시 tcp_gro_receive()를 호출하게 된다. 여기서 달라진 점은 inet_gro_compute_pseudo를 인수로 갖냐, ip6_gro_compute_pseudo를 인수로 갖냐이다.
                                                
                                            - udp6_gro_receive() ==(net/ipv6/udp_offload.c)==
                                                
                                                - 코드
                                                    
                                                    ```C
                                                    INDIRECT_CALLABLE_SCOPE
                                                    struct sk_buff *udp6_gro_receive(struct list_head *head, struct sk_buff *skb)
                                                    {
                                                    	struct udphdr *uh = udp_gro_udphdr(skb);
                                                    	struct sock *sk = NULL;
                                                    	struct sk_buff *pp;
                                                    
                                                    	if (unlikely(!uh))
                                                    		goto flush;
                                                    
                                                    	/* Don't bother verifying checksum if we're going to flush anyway. */
                                                    	if (NAPI_GRO_CB(skb)->flush)
                                                    		goto skip;
                                                    
                                                    	if (skb_gro_checksum_validate_zero_check(skb, IPPROTO_UDP, uh->check,
                                                    						 ip6_gro_compute_pseudo))
                                                    		goto flush;
                                                    	else if (uh->check)
                                                    		skb_gro_checksum_try_convert(skb, IPPROTO_UDP,
                                                    					     ip6_gro_compute_pseudo);
                                                    
                                                    skip:
                                                    	NAPI_GRO_CB(skb)->is_ipv6 = 1;
                                                    
                                                    	if (static_branch_unlikely(&udpv6_encap_needed_key))
                                                    		sk = udp6_gro_lookup_skb(skb, uh->source, uh->dest);
                                                    
                                                    	pp = udp_gro_receive(head, skb, uh, sk);
                                                    	return pp;
                                                    
                                                    flush:
                                                    	NAPI_GRO_CB(skb)->flush = 1;
                                                    	return NULL;
                                                    }
                                                    ```
                                                    
                                                
                                                > tcp보다는 조금 더 복잡하다. 여기서는 sk를 함께 인자로 넘겨서 udp_gro_receive가 실행되는데, sk는 socket이다. 해당 패킷의 목적지 포트에 맞는 소켓을 찾아가는 것이다.
                                                
                                        - napi_gro_complete(napi, skb) ==(net/core/gro.c)==
                                            
                                            - 코드
                                                
                                                ```C
                                                static void napi_gro_complete(struct napi_struct *napi, struct sk_buff *skb)
                                                {
                                                	struct list_head *head = &net_hotdata.offload_base;
                                                	struct packet_offload *ptype;
                                                	__be16 type = skb->protocol;
                                                	int err = -ENOENT;
                                                
                                                	BUILD_BUG_ON(sizeof(struct napi_gro_cb) > sizeof(skb->cb));
                                                
                                                	if (NAPI_GRO_CB(skb)->count == 1) {
                                                		skb_shinfo(skb)->gso_size = 0;
                                                		goto out;
                                                	}
                                                
                                                	rcu_read_lock();
                                                	list_for_each_entry_rcu(ptype, head, list) {
                                                		if (ptype->type != type || !ptype->callbacks.gro_complete)
                                                			continue;
                                                
                                                		err = INDIRECT_CALL_INET(ptype->callbacks.gro_complete,
                                                					 ipv6_gro_complete, inet_gro_complete,
                                                					 skb, 0);
                                                		break;
                                                	}
                                                	rcu_read_unlock();
                                                
                                                	if (err) {
                                                		WARN_ON(&ptype->list == head);
                                                		kfree_skb(skb);
                                                		return;
                                                	}
                                                
                                                out:
                                                	gro_normal_one(napi, skb, NAPI_GRO_CB(skb)->count);
                                                }
                                                ```
                                                
                                            
                                            > IP 버전에 따라 서로 다른 gro_complet를 callback할 수 있도록 하는 함수이다. 아니라면 만약 skb→count가 1이라면 해당 gro_list는 합치지 않았으므로 out 라벨로 건너뛰어 바로 gro_normal_one()함수를 호출하게 된다. 아니라면 head를 돌면서 각각의 gro_complete를 실행하게 된다. 이후 최종적으로 gro_normal_one() 함수를 실행하게 된다.  
                                            >   
                                            > INDIRECT_CALL을 하면서 부르는 함수들은 IP버전별, tcp/udp 별로 나뉘게 되며, 해당하는 함수에서 해당하는 헤더에 대한 정보를 gro를 한 뒤의 최신정보로 업데이트 하는 함수들이다. 이는 위의 ~_gro_receive()와 같은 모양새를 하고 있다고 보면 된다.  
                                            
                                            - inet_gro_complete() ==(net/ipv4/af_inet.c)==
                                                
                                                - 코드
                                                    
                                                    ```C
                                                    int inet_gro_complete(struct sk_buff *skb, int nhoff)
                                                    {
                                                    	struct iphdr *iph = (struct iphdr *)(skb->data + nhoff);
                                                    	const struct net_offload *ops;
                                                    	__be16 totlen = iph->tot_len;
                                                    	int proto = iph->protocol;
                                                    	int err = -ENOSYS;
                                                    
                                                    	if (skb->encapsulation) {
                                                    		skb_set_inner_protocol(skb, cpu_to_be16(ETH_P_IP));
                                                    		skb_set_inner_network_header(skb, nhoff);
                                                    	}
                                                    
                                                    	iph_set_totlen(iph, skb->len - nhoff);
                                                    	csum_replace2(&iph->check, totlen, iph->tot_len);
                                                    
                                                    	ops = rcu_dereference(inet_offloads[proto]);
                                                    	if (WARN_ON(!ops || !ops->callbacks.gro_complete))
                                                    		goto out;
                                                    
                                                    	/* Only need to add sizeof(*iph) to get to the next hdr below
                                                    	 * because any hdr with option will have been flushed in
                                                    	 * inet_gro_receive().
                                                    	 */
                                                    	err = INDIRECT_CALL_2(ops->callbacks.gro_complete,
                                                    			      tcp4_gro_complete, udp4_gro_complete,
                                                    			      skb, nhoff + sizeof(*iph));
                                                    
                                                    out:
                                                    	return err;
                                                    }
                                                    ```
                                                    
                                                
                                                > 다시 tcp와 udp로 콜백을 하는 함수이다. 위의 inet_gro_receive()와 그 결이 비슷하다고 볼 수 있다.
                                                
                                                - tcp4_gro_complete() ==(net/ipv4/tcp_offload.c)==
                                                    
                                                    - 코드
                                                        
                                                        ```C
                                                        INDIRECT_CALLABLE_SCOPE
                                                        struct sk_buff *tcp4_gro_receive(struct list_head *head, struct sk_buff *skb)
                                                        {
                                                        	/* Don't bother verifying checksum if we're going to flush anyway. */
                                                        	if (!NAPI_GRO_CB(skb)->flush &&
                                                        	    skb_gro_checksum_validate(skb, IPPROTO_TCP,
                                                        				      inet_gro_compute_pseudo)) {
                                                        		NAPI_GRO_CB(skb)->flush = 1;
                                                        		return NULL;
                                                        	}
                                                        
                                                        	return tcp_gro_receive(head, skb);
                                                        }
                                                        ```
                                                        
                                                    
                                                    > 여기도 단순하게 checksum 유효성만 검사하고, tcp_gro_receive()를 호출해주는 과정만 존재한다.
                                                    
                                                    - tcp_gro_complete(head, skb)
                                                        
                                                        - 코드
                                                            
                                                            ```C
                                                            void tcp_gro_complete(struct sk_buff *skb)
                                                            {
                                                            	struct tcphdr *th = tcp_hdr(skb);
                                                            	struct skb_shared_info *shinfo;
                                                            
                                                            	if (skb->encapsulation)
                                                            		skb->inner_transport_header = skb->transport_header;
                                                            
                                                            	skb->csum_start = (unsigned char *)th - skb->head;
                                                            	skb->csum_offset = offsetof(struct tcphdr, check);
                                                            	skb->ip_summed = CHECKSUM_PARTIAL;
                                                            
                                                            	shinfo = skb_shinfo(skb);
                                                            	shinfo->gso_segs = NAPI_GRO_CB(skb)->count;
                                                            
                                                            	if (th->cwr)
                                                            		shinfo->gso_type |= SKB_GSO_TCP_ECN;
                                                            }
                                                            ```
                                                            
                                                        
                                                        > skb와 shinfo에 대하여 gro가 완료되었을 때를 기준으로 정보를 업데이트 해주는 함수이다. 따로 반환값은 존재하지 않았다.
                                                        
                                            - gro_normal_one() ==(include/net/gro.h)==
                                                
                                                - 코드
                                                    
                                                    ```C
                                                    /* Queue one GRO_NORMAL SKB up for list processing. If batch size exceeded,
                                                     * pass the whole batch up to the stack.
                                                     */
                                                    static inline void gro_normal_one(struct napi_struct *napi, struct sk_buff *skb, int segs)
                                                    {
                                                    	list_add_tail(&skb->list, &napi->rx_list);
                                                    	napi->rx_count += segs;
                                                    	if (napi->rx_count >= READ_ONCE(net_hotdata.gro_normal_batch))
                                                    		gro_normal_list(napi);
                                                    }
                                                    ```
                                                    
                                                
                                                > skb→list에다가 napi→rx_list를 추가하게 되며, 합친 segment 수만큼 rx_count를 증가하게 된다. 만약 gro_normal_batch보다 크기가 커진다면 이 전체를 스택에 넘기게 된다.
                                                
                                                - gro_normal_list(napi)
                                                    
                                                    - 코드
                                                        
                                                        ```C
                                                        /* Pass the currently batched GRO_NORMAL SKBs up to the stack. */
                                                        static inline void gro_normal_list(struct napi_struct *napi)
                                                        {
                                                        	if (!napi->rx_count)
                                                        		return;
                                                        	netif_receive_skb_list_internal(&napi->rx_list);
                                                        	INIT_LIST_HEAD(&napi->rx_list);
                                                        	napi->rx_count = 0;
                                                        }
                                                        ```
                                                        
                                                    
                                                    > 최근에 병합된 GRO_NORMAL SKB를 스택 위로 전달하는 함수이다. netif_receive_skb_list_internal() 함수를 실행함으로써 수행된다.
                                                    
                                                    - netif_receive_skb_list_internal(head) ==(net/core/dev.c)==
                                                        
                                                        - 코드
                                                            
                                                            ```C
                                                            void netif_receive_skb_list_internal(struct list_head *head)
                                                            {
                                                            	struct sk_buff *skb, *next;
                                                            	struct list_head sublist;
                                                            
                                                            	INIT_LIST_HEAD(&sublist);
                                                            	list_for_each_entry_safe(skb, next, head, list) {
                                                            		net_timestamp_check(READ_ONCE(net_hotdata.tstamp_prequeue),
                                                            				    skb);
                                                            		skb_list_del_init(skb);
                                                            		if (!skb_defer_rx_timestamp(skb))
                                                            			list_add_tail(&skb->list, &sublist);
                                                            	}
                                                            	list_splice_init(&sublist, head);
                                                            
                                                            	rcu_read_lock();
                                                            \#ifdef CONFIG_RPS
                                                            	if (static_branch_unlikely(&rps_needed)) {
                                                            		list_for_each_entry_safe(skb, next, head, list) {
                                                            			struct rps_dev_flow voidflow, *rflow = &voidflow;
                                                            			int cpu = get_rps_cpu(skb->dev, skb, &rflow);
                                                            
                                                            			if (cpu >= 0) {
                                                            				/* Will be handled, remove from list */
                                                            				skb_list_del_init(skb);
                                                            				enqueue_to_backlog(skb, cpu, &rflow->last_qtail);
                                                            			}
                                                            		}
                                                            	}
                                                            \#endif
                                                            	__netif_receive_skb_list(head);
                                                            	rcu_read_unlock();
                                                            }
                                                            ```
                                                            
                                                        
                                                        > timestamp를 확인하고 만약 rps가 설정되어 있을 경우 enqueue_to_backlog()를 통해 특정 cpu에 해당 flow를 할당하게 된다. 아니라면 __netif_receive_skb_list(head)를 호출하여 처리를 이어나가게 된다.
                                                        
                                                        - __netif_receive_skb_list(head)
                                                            
                                                            - 코드
                                                                
                                                                ```C
                                                                static void __netif_receive_skb_list(struct list_head *head)
                                                                {
                                                                	unsigned long noreclaim_flag = 0;
                                                                	struct sk_buff *skb, *next;
                                                                	bool pfmemalloc = false; /* Is current sublist PF_MEMALLOC? */
                                                                
                                                                	list_for_each_entry_safe(skb, next, head, list) {
                                                                		if ((sk_memalloc_socks() && skb_pfmemalloc(skb)) != pfmemalloc) {
                                                                			struct list_head sublist;
                                                                
                                                                			/* Handle the previous sublist */
                                                                			list_cut_before(&sublist, head, &skb->list);
                                                                			if (!list_empty(&sublist))
                                                                				__netif_receive_skb_list_core(&sublist, pfmemalloc);
                                                                			pfmemalloc = !pfmemalloc;
                                                                			/* See comments in __netif_receive_skb */
                                                                			if (pfmemalloc)
                                                                				noreclaim_flag = memalloc_noreclaim_save();
                                                                			else
                                                                				memalloc_noreclaim_restore(noreclaim_flag);
                                                                		}
                                                                	}
                                                                	/* Handle the remaining sublist */
                                                                	if (!list_empty(head))
                                                                		__netif_receive_skb_list_core(head, pfmemalloc);
                                                                	/* Restore pflags */
                                                                	if (pfmemalloc)
                                                                		memalloc_noreclaim_restore(noreclaim_flag);
                                                                }
                                                                ```
                                                                
                                                            
                                                            > 리스트의 각각의 skb에 대하여 pfmemalloc과 관련한 처리를 수행한다. 관련하여 더 찾아보려 하였으나 네트워크 스택과 깊이 관련이 있지 않은 것으로 판단하여 멈추었다. 그후 만약 리스트가 비어있지 않다면 __netif_receive_skb_list_core()를 호출하게 된다.
                                                            
                                                            - __netif_receive_skb_list_core(head, pfmemealloc)
                                                                
                                                                - 코드
                                                                    
                                                                    ```C
                                                                    static void __netif_receive_skb_list_core(struct list_head *head, bool pfmemalloc)
                                                                    {
                                                                    	/* Fast-path assumptions:
                                                                    	 * - There is no RX handler.
                                                                    	 * - Only one packet_type matches.
                                                                    	 * If either of these fails, we will end up doing some per-packet
                                                                    	 * processing in-line, then handling the 'last ptype' for the whole
                                                                    	 * sublist.  This can't cause out-of-order delivery to any single ptype,
                                                                    	 * because the 'last ptype' must be constant across the sublist, and all
                                                                    	 * other ptypes are handled per-packet.
                                                                    	 */
                                                                    	/* Current (common) ptype of sublist */
                                                                    	struct packet_type *pt_curr = NULL;
                                                                    	/* Current (common) orig_dev of sublist */
                                                                    	struct net_device *od_curr = NULL;
                                                                    	struct list_head sublist;
                                                                    	struct sk_buff *skb, *next;
                                                                    
                                                                    	INIT_LIST_HEAD(&sublist);
                                                                    	list_for_each_entry_safe(skb, next, head, list) {
                                                                    		struct net_device *orig_dev = skb->dev;
                                                                    		struct packet_type *pt_prev = NULL;
                                                                    
                                                                    		skb_list_del_init(skb);
                                                                    		__netif_receive_skb_core(&skb, pfmemalloc, &pt_prev);
                                                                    		if (!pt_prev)
                                                                    			continue;
                                                                    		if (pt_curr != pt_prev || od_curr != orig_dev) {
                                                                    			/* dispatch old sublist */
                                                                    			__netif_receive_skb_list_ptype(&sublist, pt_curr, od_curr);
                                                                    			/* start new sublist */
                                                                    			INIT_LIST_HEAD(&sublist);
                                                                    			pt_curr = pt_prev;
                                                                    			od_curr = orig_dev;
                                                                    		}
                                                                    		list_add_tail(&skb->list, &sublist);
                                                                    	}
                                                                    
                                                                    	/* dispatch final sublist */
                                                                    	__netif_receive_skb_list_ptype(&sublist, pt_curr, od_curr);
                                                                    }
                                                                    ```
                                                                    
                                                                
                                                                > head 리스트에 있는 각각의 skb 패킷을 순회하게 되고, 이를 처리하기 위해 __netif_receive_skb_core()함수를 호출하게 된다.
                                                                
                                                                - __netif_receive_skb_core(skb, pfmemalloc, pt_prev)
                                                                    
                                                                    - 코드
                                                                        
                                                                        ```C
                                                                        static int __netif_receive_skb_core(struct sk_buff **pskb, bool pfmemalloc,
                                                                        				    struct packet_type **ppt_prev)
                                                                        {
                                                                        	struct packet_type *ptype, *pt_prev;
                                                                        	rx_handler_func_t *rx_handler;
                                                                        	struct sk_buff *skb = *pskb;
                                                                        	struct net_device *orig_dev;
                                                                        	bool deliver_exact = false;
                                                                        	int ret = NET_RX_DROP;
                                                                        	__be16 type;
                                                                        
                                                                        	net_timestamp_check(!READ_ONCE(net_hotdata.tstamp_prequeue), skb);
                                                                        
                                                                        	trace_netif_receive_skb(skb);
                                                                        
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
                                                                        
                                                                        skip_taps:
                                                                        \#ifdef CONFIG_NET_INGRESS
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
                                                                        \#endif
                                                                        	skb_reset_redirect(skb);
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
                                                                        
                                                                    
                                                                    > 각종 체크를 통해 패킷을 드랍하거나 스택을 올려보내는 처리를 수행한다. 중간에 모든 entry에 대하여 deliver_skb() 함수를 호출해서 스택 위로 올려보내는 역할을 수행하게 된다.
                                                                    
                                                                    - deliver_skb()
                                                                        
                                                                        - 코드
                                                                            
                                                                            ```C
                                                                            static inline int deliver_skb(struct sk_buff *skb,
                                                                            			      struct packet_type *pt_prev,
                                                                            			      struct net_device *orig_dev)
                                                                            {
                                                                            	if (unlikely(skb_orphan_frags_rx(skb, GFP_ATOMIC)))
                                                                            		return -ENOMEM;
                                                                            	refcount_inc(&skb->users);
                                                                            	return pt_prev->func(skb, skb->dev, pt_prev, orig_dev);
                                                                            }
                                                                            ```
                                                                            
                                                                        
                                                                        > refcount_inc를 통해 해당 skb를 reference 하는 count를 증가시키고, pt_prev→func 콜백으로 어떤 함수를 호출하여 작업을 수행하게 된다. net_device의 ptype_all이라는 멤버 변수를 가져다가 여기서 func을 쓰게 되는데, 이는 좀더 탐색해보아야 할 것 같다.  
                                                                        >   
                                                                        > pt_prev에서 가지고 있는 packet_type들은 보통 net_device의 ptype_list에서 가져오게 된다. 이 리스트는 해당하는 패킷 타입에 대해 다룰 수 있는 함수를 매핑하는 리스트로, dev_add_pack()이라는 함수가 이 리스트에 추가하는 함수인 것을 확인하여 grep으로 이 함수를 호출하는 지점을 찾아다녔고, 그 결과 net/ipv4/af_inet.c에서 ip_packet_type을 추가하는 것을 확인할 수 있었다. 이는 같은 파일 안에 static struct로 정의되어 있으며, ip_rcv를 함수로 설정한다. 이 함수는 net/ipv4/ip_input.c에 있다. 그 아래로 함수 스택이 더 있지만 우선 gro를 끝내고 보면 될 것 같다.  
                                                                        
                                    - napi_skb_finish()
                                        
                                        ---
                                        
                                        - 코드
                                            
                                            ```C
                                            static gro_result_t napi_skb_finish(struct napi_struct *napi,
                                            				    struct sk_buff *skb,
                                            				    gro_result_t ret)
                                            {
                                            	switch (ret) {
                                            	case GRO_NORMAL:
                                            		gro_normal_one(napi, skb, 1);
                                            		break;
                                            
                                            	case GRO_MERGED_FREE:
                                            		if (NAPI_GRO_CB(skb)->free == NAPI_GRO_FREE_STOLEN_HEAD)
                                            			napi_skb_free_stolen_head(skb);
                                            		else if (skb->fclone != SKB_FCLONE_UNAVAILABLE)
                                            			__kfree_skb(skb);
                                            		else
                                            			__napi_kfree_skb(skb, SKB_CONSUMED);
                                            		break;
                                            
                                            	case GRO_HELD:
                                            	case GRO_MERGED:
                                            	case GRO_CONSUMED:
                                            		break;
                                            	}
                                            
                                            	return ret;
                                            }
                                            ```
                                            
                                        
                                        > 위의 dev_gro_receive() 함수의 return 값을 바탕으로 switch문을 통해 뒤 처리를 하는 함수. GRO_~~ enum을 사용함.
                                        
                            
                            ==while문 종료, polling 끝남==
                            
                            - ice_put_rx_buf(rx_ring, buf)
                                
                                ---
                                
                                - 코드
                                    
                                    ```C
                                    /**
                                     * ice_put_rx_buf - Clean up used buffer and either recycle or free
                                     * @rx_ring: Rx descriptor ring to transact packets on
                                     * @rx_buf: Rx buffer to pull data from
                                     *
                                     * This function will clean up the contents of the rx_buf. It will either
                                     * recycle the buffer or unmap it and free the associated resources.
                                     */
                                    static void
                                    ice_put_rx_buf(struct ice_rx_ring *rx_ring, struct ice_rx_buf *rx_buf)
                                    {
                                    	if (!rx_buf)
                                    		return;
                                    
                                    	if (ice_can_reuse_rx_page(rx_buf)) {
                                    		/* hand second half of page back to the ring */
                                    		ice_reuse_rx_page(rx_ring, rx_buf);
                                    	} else {
                                    		/* we are not reusing the buffer so unmap it */
                                    		dma_unmap_page_attrs(rx_ring->dev, rx_buf->dma,
                                    				     ice_rx_pg_size(rx_ring), DMA_FROM_DEVICE,
                                    				     ICE_RX_DMA_ATTR);
                                    		__page_frag_cache_drain(rx_buf->page, rx_buf->pagecnt_bias);
                                    	}
                                    
                                    	/* clear contents of buffer_info */
                                    	rx_buf->page = NULL;
                                    }
                                    ```
                                    
                                
                                > napi를 통해 skb로 올라간 뒤, 다 쓰여진 버퍼를 재활용하거나 아니면 할당 해제 해주는 함수. 만약 재사용이 가능하다면 ice_reuse_rx_page(rx_ring, rx_buf)를 통하여 사용하게 된다. 아니라면, dma_unmap_page_attrs를 통해 unmap을 하게 된다.
                                
                            - ice_alloc_rx_bufs(rx_ring, ICE_RX_DESC_UNUSED(rx_ring))
                                
                                ---
                                
                                - 코드
                                    
                                    ```C
                                    /**
                                     * ice_alloc_rx_bufs - Replace used receive buffers
                                     * @rx_ring: ring to place buffers on
                                     * @cleaned_count: number of buffers to replace
                                     *
                                     * Returns false if all allocations were successful, true if any fail. Returning
                                     * true signals to the caller that we didn't replace cleaned_count buffers and
                                     * there is more work to do.
                                     *
                                     * First, try to clean "cleaned_count" Rx buffers. Then refill the cleaned Rx
                                     * buffers. Then bump tail at most one time. Grouping like this lets us avoid
                                     * multiple tail writes per call.
                                     */
                                    bool ice_alloc_rx_bufs(struct ice_rx_ring *rx_ring, unsigned int cleaned_count)
                                    {
                                    	union ice_32b_rx_flex_desc *rx_desc;
                                    	u16 ntu = rx_ring->next_to_use;
                                    	struct ice_rx_buf *bi;
                                    
                                    	/* do nothing if no valid netdev defined */
                                    	if ((!rx_ring->netdev && rx_ring->vsi->type != ICE_VSI_CTRL) ||
                                    	    !cleaned_count)
                                    		return false;
                                    
                                    	/* get the Rx descriptor and buffer based on next_to_use */
                                    	rx_desc = ICE_RX_DESC(rx_ring, ntu);
                                    	bi = &rx_ring->rx_buf[ntu];
                                    
                                    	do {
                                    		/* if we fail here, we have work remaining */
                                    		if (!ice_alloc_mapped_page(rx_ring, bi))
                                    			break;
                                    
                                    		/* sync the buffer for use by the device */
                                    		dma_sync_single_range_for_device(rx_ring->dev, bi->dma,
                                    						 bi->page_offset,
                                    						 rx_ring->rx_buf_len,
                                    						 DMA_FROM_DEVICE);
                                    
                                    		/* Refresh the desc even if buffer_addrs didn't change
                                    		 * because each write-back erases this info.
                                    		 */
                                    		rx_desc->read.pkt_addr = cpu_to_le64(bi->dma + bi->page_offset);
                                    
                                    		rx_desc++;
                                    		bi++;
                                    		ntu++;
                                    		if (unlikely(ntu == rx_ring->count)) {
                                    			rx_desc = ICE_RX_DESC(rx_ring, 0);
                                    			bi = rx_ring->rx_buf;
                                    			ntu = 0;
                                    		}
                                    
                                    		/* clear the status bits for the next_to_use descriptor */
                                    		rx_desc->wb.status_error0 = 0;
                                    
                                    		cleaned_count--;
                                    	} while (cleaned_count);
                                    
                                    	if (rx_ring->next_to_use != ntu)
                                    		ice_release_rx_desc(rx_ring, ntu);
                                    
                                    	return !!cleaned_count;
                                    }
                                    ```
                                    
                                
                                > 사용된 수신 버퍼를 대체한다. 우선 사용된 Rx Buffer를 제거하고, 새로운 깨끗한 Rx buffer를 보충하게 된다.
                                
                            - ice_finalize_xdp_rx(xdp_ring, xdp_xmit, cached_ntu) ==(ice/ice_txrx_lib.c)==
                                
                                ---
                                
                                - 코드
                                    
                                    ```C
                                    /**
                                     * ice_finalize_xdp_rx - Bump XDP Tx tail and/or flush redirect map
                                     * @xdp_ring: XDP ring
                                     * @xdp_res: Result of the receive batch
                                     * @first_idx: index to write from caller
                                     *
                                     * This function bumps XDP Tx tail and/or flush redirect map, and
                                     * should be called when a batch of packets has been processed in the
                                     * napi loop.
                                     */
                                    void ice_finalize_xdp_rx(struct ice_tx_ring *xdp_ring, unsigned int xdp_res,
                                    			 u32 first_idx)
                                    {
                                    	struct ice_tx_buf *tx_buf = &xdp_ring->tx_buf[first_idx];
                                    
                                    	if (xdp_res & ICE_XDP_REDIR)
                                    		xdp_do_flush();
                                    
                                    	if (xdp_res & ICE_XDP_TX) {
                                    		if (static_branch_unlikely(&ice_xdp_locking_key))
                                    			spin_lock(&xdp_ring->tx_lock);
                                    		/* store index of descriptor with RS bit set in the first
                                    		 * ice_tx_buf of given NAPI batch
                                    		 */
                                    		tx_buf->rs_idx = ice_set_rs_bit(xdp_ring);
                                    		ice_xdp_ring_update_tail(xdp_ring);
                                    		if (static_branch_unlikely(&ice_xdp_locking_key))
                                    			spin_unlock(&xdp_ring->tx_lock);
                                    	}
                                    }
                                    ```
                                    
                                
                                > Tx일 때만 사용되므로 집중해서 볼 필요는 없다.
                                
                            - ice_update_rx_ring_stats(rx_ring, total_rx_pkts, total_rx_bytes) ==(ice/ice_lib.c)==
                                
                                ---
                                
                                - 코드
                                    
                                    ```C
                                    /**
                                     * ice_update_rx_ring_stats - Update Rx ring specific counters
                                     * @rx_ring: ring to update
                                     * @pkts: number of processed packets
                                     * @bytes: number of processed bytes
                                     */
                                    void ice_update_rx_ring_stats(struct ice_rx_ring *rx_ring, u64 pkts, u64 bytes)
                                    {
                                    	u64_stats_update_begin(&rx_ring->ring_stats->syncp);
                                    	ice_update_ring_stats(&rx_ring->ring_stats->stats, pkts, bytes);
                                    	u64_stats_update_end(&rx_ring->ring_stats->syncp);
                                    }
                                    ```
                                    
                                
                                > rx ring의 상태를 업데이트하는 코드
                                
                    
                    - napi_complete(n) / napi_complete_done(n, 0)
                        
                        ---
                        
                        - 코드
                            
                            ```C
                            static inline bool napi_complete(struct napi_struct *n)
                            {
                            	return napi_complete_done(n, 0);
                            }
                            
                            bool napi_complete_done(struct napi_struct *n, int work_done)
                            {
                            	unsigned long flags, val, new, timeout = 0;
                            	bool ret = true;
                            
                            	/*
                            	 * 1) Don't let napi dequeue from the cpu poll list
                            	 *    just in case its running on a different cpu.
                            	 * 2) If we are busy polling, do nothing here, we have
                            	 *    the guarantee we will be called later.
                            	 */
                            	if (unlikely(n->state & (NAPIF_STATE_NPSVC |
                            				 NAPIF_STATE_IN_BUSY_POLL)))
                            		return false;
                            
                            	if (work_done) {
                            		if (n->gro_bitmask)
                            			timeout = READ_ONCE(n->dev->gro_flush_timeout);
                            		n->defer_hard_irqs_count = READ_ONCE(n->dev->napi_defer_hard_irqs);
                            	}
                            	if (n->defer_hard_irqs_count > 0) {
                            		n->defer_hard_irqs_count--;
                            		timeout = READ_ONCE(n->dev->gro_flush_timeout);
                            		if (timeout)
                            			ret = false;
                            	}
                            	if (n->gro_bitmask) {
                            		/* When the NAPI instance uses a timeout and keeps postponing
                            		 * it, we need to bound somehow the time packets are kept in
                            		 * the GRO layer
                            		 */
                            		napi_gro_flush(n, !!timeout);
                            	}
                            
                            	gro_normal_list(n);
                            
                            	if (unlikely(!list_empty(&n->poll_list))) {
                            		/* If n->poll_list is not empty, we need to mask irqs */
                            		local_irq_save(flags);
                            		list_del_init(&n->poll_list);
                            		local_irq_restore(flags);
                            	}
                            	WRITE_ONCE(n->list_owner, -1);
                            
                            	val = READ_ONCE(n->state);
                            	do {
                            		WARN_ON_ONCE(!(val & NAPIF_STATE_SCHED));
                            
                            		new = val & ~(NAPIF_STATE_MISSED | NAPIF_STATE_SCHED |
                            			      NAPIF_STATE_SCHED_THREADED |
                            			      NAPIF_STATE_PREFER_BUSY_POLL);
                            
                            		/* If STATE_MISSED was set, leave STATE_SCHED set,
                            		 * because we will call napi->poll() one more time.
                            		 * This C code was suggested by Alexander Duyck to help gcc.
                            		 */
                            		new |= (val & NAPIF_STATE_MISSED) / NAPIF_STATE_MISSED *
                            						    NAPIF_STATE_SCHED;
                            	} while (!try_cmpxchg(&n->state, &val, new));
                            
                            	if (unlikely(val & NAPIF_STATE_MISSED)) {
                            		__napi_schedule(n);
                            		return false;
                            	}
                            
                            	if (timeout)
                            		hrtimer_start(&n->timer, ns_to_ktime(timeout),
                            			      HRTIMER_MODE_REL_PINNED);
                            	return ret;
                            }
                            ```
                            
                        
                        > napi가 잘 되었는지 확인하고 기타 작업을 처리하는 함수. gro_bitmask와 defer_hard_irqs_count등을 확인하여 timeout관련 처리를 하고, 만약 gro_bitmask가 존재한다면 napi_gro_flush()를 실행하게 됨. 이후 gro_normal_list(n)을 하게 됨. 만약 NAPIF_STATE_MISSED가 있다면 다시 __napi_schedule(n)을 통해 napi가 실행 될 수 있도록 함.
                        
                    - napi_prefer_busy_poll(n)
                        
                        ---
                        
                        - 코드
                            
                            ```C++
                            static inline bool napi_prefer_busy_poll(struct napi_struct *n)
                            {
                            	return test_bit(NAPI_STATE_PREFER_BUSY_POLL, &n->state);
                            }
                            ```
                            
                        
                        > napi_struct의 state 변수가 NAPI_STATE_PREFER_BUSY_POLL이 셋팅 되어있는지 test_bit을 하게 되는 함수
                        
                    - gro_normal_list(n)
                        
                        ---
                        
                        - 코드
                            
                            ```C
                            /* Pass the currently batched GRO_NORMAL SKBs up to the stack. */
                            static inline void gro_normal_list(struct napi_struct *napi)
                            {
                            	if (!napi->rx_count)
                            		return;
                            	netif_receive_skb_list_internal(&napi->rx_list);
                            	INIT_LIST_HEAD(&napi->rx_list);
                            	napi->rx_count = 0;
                            }
                            ```
                            
                        
                        > 최근에 GRO로 합쳐진 skb를 스택으로 올려주는 함수. netif_receive_skb_list_internal(&napi→rx_list)를 호출하여 내부적으로 필요한 작업을 수행하게 된다.
                        
            - net_rps_action_and_irq_enable(sd)
                
                ---
                
                - 코드
                    
                      
                    
                

  

---

![[Untitled 1 2.png|Untitled 1 2.png]]

[https://pr0gr4m.github.io/linux/kernel/sk_buff/](https://pr0gr4m.github.io/linux/kernel/sk_buff/)

![[Untitled 2 2.png|Untitled 2 2.png]]

### 앞으로 찾아봐야 할 것들

---

1. Ctrl VSI가 어떤 역할을 하는가?

- sk_buff 등장하는곳
    
    →ice_napi_add()
    
    →ice_napi_poll()
    
    →ice_clean_rx_irq()
    
    →ice_receive_skb()
    
    ⇒ sk_buff 출현, 이미 napi를 완료했고, 스택에 넘겨주는 상황