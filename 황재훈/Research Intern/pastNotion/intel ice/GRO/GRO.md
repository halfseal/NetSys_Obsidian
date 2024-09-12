# **Network data processing begins**

[https://www.slideshare.net/slideshow/linux-network-stack/39611307](https://www.slideshare.net/slideshow/linux-network-stack/39611307)

- img
    
    ![[황재훈/Research Intern/pastNotion/intel ice/GRO/img/Untitled.png|Untitled.png]]
    

  

GRO [https://lwn.net/Articles/358910/](https://lwn.net/Articles/358910/)

1. MTU (pkt size) : 1500 byte
2. Jumbo frame (9kB)
3. 10G netw → 800,000 pkt / s
4. interent에서는 1500 byte보다 작은 mtu를 사용하는 경우도 있다, 큰 경우 방화벽이 쪼개버린다
5. LRO는 보이는 pkt을 모두 합쳐버림
    1. system이 router인 경우, header가 바뀌면 안된다
    2. 위성 통신도 header에 의존해서 바뀌면 안된다
    3. virtual netw에서 LRO는 큰 문제, bridge break 문제. LRO를 꺼야할 정도
6. GRO는 merge 기준이 까다롭다
    1. mac header가 동일, tcp/ip header는 조금만 다를 수 있다
    2. checksum은 달라도 된다
    3. ip id field는 증가하는 경우에만 가능 (incremental)?
    4. TCP timestamps은 동일해야
7. LRO와 달리, tcp/ipv4 이외에도 사용가능

  

napi gro receive

GRO (generic receive offloading)

1. LRO의 sw 버전
2. 비슷한 종류의 pkt를 하나로 합친다
3. information loss
    1. 중요한 option, flag가 있을 경우, 정보가 날라간다
    2. LRO는 pkt을 자주 합치기 때문에, 정보 손실에 의해 권장되지 않는다
    3. GRO는 이를 막기위해 pkt 합치는 기준이 까다롭다

  

- `int ice_clean_rx_irq(struct ice_rx_ring *rx_ring, int budget)`
    
    ```JavaScript
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
    	while (likely(total_rx_pkts < (unsigned int)budget)) { explain
    		union ice_32b_rx_flex_desc *rx_desc;
    		struct ice_rx_buf *rx_buf;
    		struct sk_buff *skb;
    		unsigned int size;
    		u16 stat_err_bits;
    		u16 vlan_tci;
    
    		/* get the Rx desc from Rx ring based on 'next_to_clean' */
    		rx_desc = ICE_RX_DESC(rx_ring, ntc);  explain
    
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
    		dma_rmb(); explain
    
    		ice_trace(clean_rx_irq, rx_ring, rx_desc); explain
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
    

1. total_rx_pkts < budget

- rx pkt의 수가 budget 보다 작은 경우에만 실행

1. rx_desc = ICE_RX_DESC(rx_ring, ntc)
2. `vlan_tci = ice_get_vlan_tci(rx_desc);`

- vlan tci types
    
    ```JavaScript
    \#define VLAN_PRIO_MASK		0xe000 /* Priority Code Point */
    \#define VLAN_PRIO_SHIFT		13
    \#define VLAN_CFI_MASK		0x1000 /* Canonical Format Indicator / Drop Eligible Indicator */
    \#define VLAN_VID_MASK		0x0fff /* VLAN Identifier */
    \#define VLAN_N_VID		4096
    ```
    

  

- `void ice_receive_skb(struct ice_rx_ring *rx_ring, struct sk_buff *skb, u16 vlan_tci)`
    
    ```JavaScript
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
    
- `gro_result_t napi_gro_receive(struct napi_struct *napi, struct sk_buff *skb)`
    
    ```JavaScript
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
    EXPORT_SYMBOL(napi_gro_receive);
    ```
    

  

- `static enum gro_result dev_gro_receive(struct napi_struct *napi, struct sk_buff *skb)`
    
    ```JavaScript
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
    				&gro_list->list, skb);
    
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
    
- `static inline bool netif_elide_gro(const struct net_device *dev)`
    
    ```JavaScript
    static inline bool netif_elide_gro(const struct net_device *dev)
    {
    	if (!(dev->features & NETIF_F_GRO) || dev->xdp_prog)
    		return true;
    	return false;
    }
    ```
    

1. pkt offload type
    
    1. packet을 보통 cpu가 처리하는데 이를 offload 할 때, 필요한 정보들이 저장되어있다
    2. 어떤 종류의 pkt인지 ethernet, tcp, ip
    3. gro, gso 로 가는 offload handler
    
    - `struct packet_offload, offload_callbacks`
        
        ```JavaScript
        struct packet_offload {
        	__be16			 type;	/* This is really htons(ether_type). */
        	u16			 priority;
        	struct offload_callbacks callbacks;
        	struct list_head	 list;
        };
        
        struct offload_callbacks {
        	struct sk_buff		*(*gso_segment)(struct sk_buff *skb,
        						netdev_features_t features);
        	struct sk_buff		*(*gro_receive)(struct list_head *head,
        						struct sk_buff *skb);
        	int			(*gro_complete)(struct sk_buff *skb, int nhoff);
        };
        ```
        

- `gro_result_t napi_skb_finish(struct napi_struct *napi, struct sk_buff *skb, gro_result_t ret)`
    
    ```JavaScript
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
    

  

- `struct sk_buff *ipv6_gro_receive(struct list_head *head, struct sk_buff *skb)`
    
    ```JavaScript
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
    

1. `indirect_call_gro_receive_l4(tcp6_gro_receive, udp6_gro_receive, ops->callbacks.gro_receive, head, skb);`
    
    - `tcp6_gro_receive(struct list_head *head, struct sk_buff *skb)`
        
        ```JavaScript
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
        
    - `tcp_gro_receive()`
        
        ```JavaScript
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
        
    - `int skb_gro_receive(struct sk_buff *p, struct sk_buff *skb)`
        
        ```JavaScript
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
        		} while (--i);
        
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
        
    
      
    

- `enum gro_result`
    
    enum gro_result {  
    GRO_MERGED,  
    GRO_MERGED_FREE,  
    GRO_HELD,  
    GRO_NORMAL,  
    GRO_CONSUMED,  
    };  
    typedef enum gro_result gro_result_t;  
    
- `static __latent_entropy void net_rx_action(struct softirq_action *h)`
    
    ```JavaScript
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
    		budget -= napi_poll(n, &repoll);
    
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
    

  

1. net_rx_action() 함수가 ksoftirqd kernel thread에 의해서 실행된다
    1. 현재 cpu에 있는 poll list 속 napi struct를 처리한다
    2. napi struct는 driver에서 napi_schedule 에 의해서 추가되거나
    3. rx pkt steering 에 의해서 inter process interrupt에 의해 추가된다