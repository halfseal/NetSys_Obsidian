![[황재훈/Research Intern/pastNotion/intel ice/dev_gro_receive()/img/Untitled.png|Untitled.png]]

[https://blog.cloudflare.com/the-tale-of-a-single-register-value](https://blog.cloudflare.com/the-tale-of-a-single-register-value)

기준점 skb는 어디서 정해지는가?

dev_gro_receive에서 변수로 받는 skb로 list에서 비교를 한다

  

dev_gro_receive(…., skb) → napi_gro_receive(…., skb) → ice_receive_skb(…., skb) → ice_clean_rx_irq()

ice_clean_rx_irq(struct ice_rx_ring *rx_ring)에서 rx_ring에 존재하는 xdp에서 skb를 만든다.

rx_ring은 ice_napi_poll 함수의 q_vector 에서 가져온다

  

ice_for_each_rx_ring(rx_ring, q_vector->rx) {

`struct ice_rx_ring` : descriptor ring

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
    		//xdp buffer를 감싸서 skb를 생성, memcpy overhead가 없어진다
    			skb = ice_build_skb(rx_ring, xdp);
    		else
    		//직접 만든다, xdp_buff에 있는 걸로 skb 만든
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
    }T
    ```
    
- `skb = ice_construct_skb(rx_ring, xdp);`
    
    ```JavaScript
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
    

  

dev_gro_result는 gro를 실행하고 그 결과를 gro_result의 형태로 리턴한다

gro_result

```JavaScript
enum gro_result {
	GRO_MERGED,
	GRO_MERGED_FREE,
	GRO_HELD,
	GRO_NORMAL,
	GRO_CONSUMED,  //gro가 실행되고 있는 상태
};
typedef enum gro_result gro_result_t;
```

- `static enum gro_result dev_gro_receive(struct napi_struct *napi, struct sk_buff *skb)` 코
    
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
    	// gro list 안에 있는 skb 속 control block안에 있는 same flow 변수들을 설정해준다
    	rcu_read_lock();
    	list_for_each_entry_rcu(ptype, head, list) {
    		if (ptype->type == type && ptype->callbacks.gro_receive)
    		// list에 있는 pkt들의 protocol이 입력받은 skb와 동일한지 + gro_receive 인지 확인
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
    

```JavaScript
	u32 bucket = skb_get_hash_raw(skb) & (GRO_HASH_BUCKETS - 1);
	// 나머지 계산
	// hash 번호에 맞는 gro list를 가져온다
	struct gro_list *gro_list = &napi->gro_hash[bucket];
	...
	gro_list_prepare(&gro_list->list, skb);
	// gro list 안에 있는 skb 속 control block안에 있는 same flow 변수들을 설정해준다
	rcu_read_lock();
	list_for_each_entry_rcu(ptype, head, list) {
		if (ptype->type == type && ptype->callbacks.gro_receive)
		// list에 있는 pkt들의 protocol이 입력받은 skb와 동일한지 + gro_receive 인지 확인
			goto found_ptype;
	}
```

1. `gro_list_prepare(&gro_list->list, skb);`
    - code
        
        ```JavaScript
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
        		}  // hash 값이 다르면 same_flow가 아니라고 하고 나간다
        		
        		// 같은 경우
        		// diff 값에 따라 pkt의 same_flow 값을 정한다
        		// 모든 비교값들이 동일해야한다
        		// skb->dev, vlan_all, metadata, mac header
        		// slow gro 인 경우, 추가적인 비교 skb->sk, metadata_dst??
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
        		// or 연산한 값들중 하나라도 다른것이 있으면 
        		// !diff에서 0이 되어 same_flow가 아닌 것으로 처리가 된다
        	}
        }
        ```
        
2. `struct packet_offload * ptype` 비교하기
    
    1. 같은 protocol에서 왔는지, gro receive인지 확인
    2. type에는 ethernet frame에 있는 16bit 영역, frame size, 어떤 프로토콜이 사용되었는지 기록되어 있어서 data link layer에서 payload를 어떻게 열지 결정한다. vlan tagging에도 사용된다
    
    - **struct packet_offload, offload_callbacks**
        
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
        

```JavaScript
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
		// gso로 생성된 skb인 경우, count 변수에 gso_segment개수로 설정한다
		/* Only support TCP and non DODGY users. */
		if (!skb_is_gso_tcp(skb) ||
		    (skb_shinfo(skb)->gso_type & SKB_GSO_DODGY))
			NAPI_GRO_CB(skb)->flush = 1;
	}
```

1. `NAPI_GRO_CB(skb)` \#define NAPI_GRO_CB(skb) ((struct napi_gro_cb *)(skb)->cb)
    
    1. skb의 gro control block에 접근하게 해준다
    
    - struct napi_gro_cb 코드
        
        ```JavaScript
        struct napi_gro_cb {
        	union {
        		struct {
        			/* Virtual address of skb_shinfo(skb)->frags[0].page + offset. */
        			void	*frag0;
        
        			/* Length of frag0. */
        			unsigned int frag0_len;
        		};
        
        		struct {
        			/* used in skb_gro_receive() slow path */
        			struct sk_buff *last;
        
        			/* jiffies when first packet was created/queued */
        			unsigned long age;
        		};
        	};
        
        	/* This indicates where we are processing relative to skb->data. */
        	int	data_offset;
        
        	/* This is non-zero if the packet cannot be merged with the new skb. */
        	u16	flush;
        
        	/* Save the IP ID here and check when we get to the transport layer */
        	u16	flush_id;
        
        	/* Number of segments aggregated. */
        	u16	count;
        
        	/* Used in ipv6_gro_receive() and foo-over-udp and esp-in-udp */
        	u16	proto;
        
        /* Used in napi_gro_cb::free */
        \#define NAPI_GRO_FREE             1
        \#define NAPI_GRO_FREE_STOLEN_HEAD 2
        	/* portion of the cb set to zero at every gro iteration */
        	struct_group(zeroed,
        
        		/* Start offset for remote checksum offload */
        		u16	gro_remcsum_start;
        
        		/* This is non-zero if the packet may be of the same flow. */
        		u8	same_flow:1;
        
        		/* Used in tunnel GRO receive */
        		u8	encap_mark:1;
        
        		/* GRO checksum is valid */
        		u8	csum_valid:1;
        
        		/* Number of checksums via CHECKSUM_UNNECESSARY */
        		u8	csum_cnt:3;
        
        		/* Free the skb? */
        		u8	free:2;
        
        		/* Used in foo-over-udp, set in udp[46]_gro_receive */
        		u8	is_ipv6:1;
        
        		/* Used in GRE, set in fou/gue_gro_receive */
        		u8	is_fou:1;
        
        		/* Used to determine if flush_id can be ignored */
        		u8	is_atomic:1;
        
        		/* Number of gro_receive callbacks this packet already went through */
        		u8 recursion_counter:4;
        
        		/* GRO is done by frag_list pointer chaining. */
        		u8	is_flist:1;
        	);
        
        	/* used to support CHECKSUM_COMPLETE for tunneling protocols */
        	__wsum	csum;
        
        	/* L3 offsets */
        	union {
        		struct {
        			u16 network_offset;
        			u16 inner_network_offset;
        		};
        		u16 network_offsets[2];
        	};
        };
        ```
        
    - 멤버 변수들
        - data_offset : skb→data 에서 시작하는 offset, 어디를 처리하는지 알려준다
        - flush : 다른 pkt과 merge가 불가능하면 0이 아닌 값이 들어있다
        - count : 합쳐진 segment의 개수
        - same_flow : 같은 flow에 있으면 0이 아닌 값이 들어있다
2. `\#define skb_shinfo(SKB)` ((struct skb_shared_info *)(skb_end_pointer(SKB)))
    
    - struct skb_shared_info 코드
        
        ```JavaScript
        /* This data is invariant across clones and lives at
         * the end of the header data, ie. at skb->end.
         */
        struct skb_shared_info {
        	__u8		flags;
        	__u8		meta_len;
        	__u8		nr_frags;
        	__u8		tx_flags;
        	unsigned short	gso_size;
        	/* Warning: this field is not always filled in (UFO)! */
        	unsigned short	gso_segs;
        	struct sk_buff	*frag_list;
        	union {
        		struct skb_shared_hwtstamps hwtstamps;
        		struct xsk_tx_metadata_compl xsk_meta;
        	};
        	unsigned int	gso_type;
        	u32		tskey;
        
        	/*
        	 * Warning : all fields before dataref are cleared in __alloc_skb()
        	 */
        	atomic_t	dataref;
        	unsigned int	xdp_frags_size;
        
        	/* Intermediate layers must ensure that destructor_arg
        	 * remains valid until skb destructor */
        	void *		destructor_arg;
        
        	/* must be last field, see pskb_expand_head() */
        	skb_frag_t	frags[MAX_SKB_FRAGS];
        };
        ```
        
    
    ```JavaScript
     	pp = INDIRECT_CALL_INET(ptype->callbacks.gro_receive,
    			ipv6_gro_receive, inet_gro_receive,
    			&gro_list->list, skb);
    
    	rcu_read_unlock();
    ```
    
3. INDIRECT_CALL_INET(ptype->callbacks.gro_receive, ipv6_gro_receive, inet_gro_receive, &gro_list->list, skb)

- `struct sk_buff *ipv6_gro_receive(struct list_head *head, struct sk_buff *skb)` 코드 
    
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
    

  

1. 주어진 skb의 헤더를 열어보고 ipv6에 맞는 규격인지 확인한다. header이 길이 payload의 크기 비교, 콜백 함수가 gro_receive인지 확인, 아닐 경우 flush
    - skb의 header 판별 코드
        
        ```JavaScript
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
        ```
        

  

1. 이제 gro list 에 있는 pkt들의 헤더를 비교한다. ipheader에 있는 정보를 열어보고 변수로 주어진 skb와 비교해서 같은 flow에서 왔는지 판별한다. header의 첫 32 bit, 출발 주소, 도착주소

```JavaScript
list_for_each_entry(p, head, list) {
    const struct ipv6hdr *iph2;
    __be32 first_word; /* <Version:4><Traffic_Class:8><Flow_Label:20> */

    if (!NAPI_GRO_CB(p)->same_flow)
        continue;

    iph2 = (struct ipv6hdr *)(p->data + off);
    first_word = *(__be32 *)iph ^ *(__be32 *)iph2;
    // ipv6 header의 처음 4bit는 버전 ipv6인 경우 6, ipv4 이면 4 
    // 8비트는 packet classification에 사용, 패킷의 우선순위 정한다
    // 나머지 20비트는 flow label, 같은 flow일 경우 동일한 값 가져야한다

		// first word 에서 다른 부분이 있는 경우, source addr, destination addr 비교
		// 하나라도 다르면 not same flow 설정후 종료
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
    NAPI_GRO_CB(p)->flush |= !!((first_word & htonl(0x0FF00000)) |
        (__force __be32)(iph->hop_limit ^ iph2->hop_limit));
    NAPI_GRO_CB(p)->flush |= flush;
    // traffic class를 위에서 비교안하고 여기서 하는 이유?
    // 위의 경우는 same_flow를, 아래는 flush를 설정하는데,
    // same_flow 이어도 flush를 하는 경우가 있을까?

    if (NAPI_GRO_CB(skb)->is_atomic)
        NAPI_GRO_CB(p)->flush_id = 0;
}

NAPI_GRO_CB(skb)->is_atomic = true;
NAPI_GRO_CB(skb)->flush |= flush;
skb_gro_postpull_rcsum(skb, iph, nlen);

pp = indirect_call_gro_receive_l4(tcp6_gro_receive, udp6_gro_receive,
                                  ops->callbacks.gro_receive, head, skb);
```

  

checksum 확인하고 tcp_gro_receive(head, skb) 호출한다. checksum 틀리면 null 리턴

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
    

tcp header를 열어봐서 같은 flow에서 왔는지 확인한다.

- `tcp_gro_receive()` 코드
    
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
    }7
    ```
    

```JavaScript
	list_for_each_entry(p, head, list) {
		if (!NAPI_GRO_CB(p)->same_flow)
			continue;

		th2 = tcp_hdr(p);

		if (*(u32 *)&th->source ^ *(u32 *)&th2->source) {
		// sending device port number를 비교한다.
			NAPI_GRO_CB(p)->same_flow = 0;
			continue;
		}

		goto found;
	}
```

```JavaScript
	/* Include the IP ID check below from the inner most IP hdr */
	flush = NAPI_GRO_CB(p)->flush;
	flush |= (__force int)(flags & TCP_FLAG_CWR);
	// flag에 tcp_flag_cwr 가 켜져있으면 flush 한다
	flush |= (__force int)((flags ^ tcp_flag_word(th2)) &
		  ~(TCP_FLAG_CWR | TCP_FLAG_FIN | TCP_FLAG_PSH));
	// tcp_flag_cwr, tcp_flag_fin, tcp_flag_psh를 제외하고 flag에 차이가 있으면 flush한다
	flush |= (__force int)(th->ack_seq ^ th2->ack_seq);
	// ack_seq 가 다르면 flush 한다?? 같은 flow면 ack_seq가 달라야 정상인거 아닌가?
	for (i = sizeof(*th); i < thlen; i += 4)
		flush |= *(u32 *)((u8 *)th + i) ^
			 *(u32 *)((u8 *)th2 + i);
	// 남 header부분을 모두 비교해서 다르면 flush
```

tcp_control_flag

- cwr : congestion window reduced,
- fin : finish 세션 종료시킨다
- psh : push, 버퍼가 채워지지 않더라도 상위 layer로 데이터를 전달
- urg : urgent
- ack : acknowledgment
- syn : synchronization, 연결을 초기화하기 위한 순서번호의 동기

```JavaScript
\#endif

	if (flush || skb_gro_receive(p, skb)) {
		mss = 1;
		goto out_check_final;
	}

	tcp_flag_word(th2) |= flags & (TCP_FLAG_FIN | TCP_FLAG_PSH);

out_check_final:
```

  

성공할 경우 0을 리턴한다.

skb를 p의 frag list로 옮긴다

- `int skb_gro_receive(struct sk_buff *p, struct sk_buff *skb)` code
    
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
    	// p 는 gro list에서온 skb, skb는 비교 기준 skb
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
    			//gro max size의 개수보다 많은 pkt이면 merge 실행한다
    
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
    	// lp 는 grolist출신 skb의 frag list에서의 마지막 skb
    	pinfo = skb_shinfo(lp);
    	// lp의 skb shared info 가 pinfo
    
    	if (headlen <= offset) {
    		skb_frag_t *frag;
    		skb_frag_t *frag2;
    		int i = skbinfo->nr_frags;
    		// i 는 비교대상 skb가 가진 frag 개수
    
    		int nr_frags = pinfo->nr_frags + i;
    		// nr_frags에는 p가 가진 frag 개수 + 비교skb가 가진 frag 개수를 합친다
    
    
    		if (nr_frags > MAX_SKB_FRAGS)
    			goto merge;
    
    		offset -= headlen;
    		pinfo->nr_frags = nr_frags;
    		// pinfo frag 개수 변수에 전체 frag 개수를 넣는다
    		skbinfo->nr_frags = 0;
    		// 비교대상 skb에 frag 개수는 0으로 설정한다
    		// 비교대상 skb의 frag들을 모두 p로 이동한다
    		
    		frag = pinfo->frags + nr_frags;
    		frag2 = skbinfo->frags + i;
    		// p에 frag들이 들어갈 자리를 비교대상 skb의 frag개수만큼 확보한다
    		// 비교대상 skb의 마지막 frag를 가르키도록 주소를 설정한다
    		do {
    			*--frag = *--frag2;
    			// 마지막 frag부터 p로 뒤에서부터 앞으로 이동시킨다
    		} while (--i);
    		// 1 2 3
    		// 4 5
    		
    		// 1 2 3 * *
    		// 4 5
    		
    		// 1 2 3 * 5
    		// 4 5
    		
    		// 1 2 3 4 5
    		// 4 5
    
    		skb_frag_off_add(frag, offset);
    		skb_frag_size_sub(frag, offset);
    		// offset 바꿔주는 함수들
    
    		/* all fragments truesize : remove (head size + sk_buff) */
    		new_truesize = SKB_TRUESIZE(skb_end_offset(skb));
    		delta_truesize = skb->truesize - new_truesize;
    
    		skb->truesize = new_truesize;
    		skb->len -= skb->data_len;
    		skb->data_len = 0;
    		// skb meta data들 바꿔주기
    
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
    		// frag->offset += delta; eat만큼 offset 더해준다.
    		skb_frag_size_sub(&skbinfo->frags[0], eat);
    		// 	frag->len -= delta; Decrements the size of a skb fragment by @delta
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
    	// lp = NAPI_GRO_CB(p)->last;
    	case 1
    	// a b c p
    	// a b c p skb
    	case 2
    	// a b c p e
    	// a b c p e skb
     
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
    

==**// skb: A**==

==**// gro list: a1 a2 b1 b2 b3**==

  

==**// A a1 a2**==

==**// a1 A**==

  

[https://lwn.net/Articles/800310/](https://lwn.net/Articles/800310/)

```Plain
This patchset adds support to do GRO/GSO by chaining packets
of the same flow at the SKB frag_list pointer. This avoids
the overhead to merge payloads into one big packet, and
on the other end, if GSO is needed it avggoids the overhead
of splitting the big packet back to the native form.
```

  

[https://www.kernel.org/doc/html/v5.7/networking/segmentation-offloads.html](https://www.kernel.org/doc/html/v5.7/networking/segmentation-offloads.html)

Generic receive offload is the complement to GSO. Ideally any frame assembled by GRO should be segmented to create an identical sequence of frames using GSO, and any sequence of frames segmented by GSO should be able to be reassembled back to the original by GRO. The only exception to this is IPv4 ID in the case that the DF bit is set for a given IP header. If the value of the IPv4 ID is not sequentially incrementing it will be altered so that it is when a frame assembled via GRO is segmented via GSO.

skb->data_len tells how many bytes of paged data there are in the SKB

  

  

1. \#define MAX_GRO_SKBS 8
2. napi_skb_finish()
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
        

  

  

1. `indirect_call_gro_receive_l4(tcp6_gro_receive, udp6_gro_receive, ops->callbacks.gro_receive, head, skb);`
    
      
    

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