ice_clean_rx_irq

```JavaScript
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
				ice_set_rx_bufs_act(xdp, rx_ring, ICE_XDP_CONSUMED);
			xdp->data = NULL;
			rx_ring->first_desc = ntc;
			rx_ring->nr_frags = 0;
			break;
		}
		xdp->data = NULL;
		rx_ring->first_desc = ntc;
		rx_ring->nr_frags = 0;
```

[https://codebrowser.dev/linux/linux/drivers/net/ethernet/intel/ice/ice_txrx.c.html#ice_build_skb](https://codebrowser.dev/linux/linux/drivers/net/ethernet/intel/ice/ice_txrx.c.html#ice_build_skb)

- ice_build_skb()
    
    ```JavaScript
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
    

XDP buffer의 공간을 활용해서 skb를 만든다. XDP를 감싸서 skb를 생성한다.

이로 memcpy overhead를 피할 수 있다.

  

ice_build_skb 가 호출한 napi_build_skb()

- napi_build_skb()
    
    ```JavaScript
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
    ```
    
      
    
- __napi_build_skb()
    
    ```JavaScript
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
    

  

ice_clean_rx_irq() → ice_build_skb() → napi_build_skb() → __napi_build_skb()

- napi_skb_cache_get()
    
    ```JavaScript
    static struct sk_buff *napi_skb_cache_get(void)
    {
    	struct napi_alloc_cache *nc = this_cpu_ptr(&napi_alloc_cache);
    	struct sk_buff *skb;
    
    	if (unlikely(!nc->skb_count)) {
    		nc->skb_count = kmem_cache_alloc_bulk(net_hotdata.skbuff_cache,
    						      GFP_ATOMIC,
    						      NAPI_SKB_CACHE_BULK,
    						      nc->skb_cache);
    		if (unlikely(!nc->skb_count))
    			return NULL;
    	}
    
    	skb = nc->skb_cache[--nc->skb_count];
    	kasan_mempool_unpoison_object(skb, kmem_cache_size(net_hotdata.skbuff_cache));
    
    	return skb;
    }
    ```
    

- cache slab allocation 과정에서 cache slab를 free하지 않고 다시 사용할 수 있도록하는 함수이다

- __build_skb_around()
    
    ```JavaScript
    /* Caller must provide SKB that is memset cleared */
    static void __build_skb_around(struct sk_buff *skb, void *data,
    			       unsigned int frag_size)
    {
    	unsigned int size = frag_size;
    
    	/* frag_size == 0 is considered deprecated now. Callers
    	 * using slab buffer should use slab_build_skb() instead.
    	 */
    	if (WARN_ONCE(size == 0, "Use slab_build_skb() instead"))
    		data = __slab_build_skb(skb, data, &size);
    
    	__finalize_skb_around(skb, data, size);
    }
    ```
    

  

ice_clean_rx_irq() → ice_build_skb() → napi_build_skb() → __napi_build_skb() → __build_skb_around()

- __slab_build_skb()
    
    ```JavaScript
    static inline void *__slab_build_skb(struct sk_buff *skb, void *data,
    				     unsigned int *size)
    {
    	void *resized;
    
    	/* Must find the allocation size (and grow it to match). */
    	*size = ksize(data);
    	/* krealloc() will immediately return "data" when
    	 * "ksize(data)" is requested: it is the existing upper
    	 * bounds. As a result, GFP_ATOMIC will be ignored. Note
    	 * that this "new" pointer needs to be passed back to the
    	 * caller for use so the __alloc_size hinting will be
    	 * tracked correctly.
    	 */
    	resized = krealloc(data, *size, GFP_ATOMIC);
    	WARN_ON_ONCE(resized != data);
    	return resized;
    }
    ```
    
- __finalize_skb_around()
    
    ```JavaScript
    static inline void __finalize_skb_around(struct sk_buff *skb, void *data,
    					 unsigned int size)
    {
    	struct skb_shared_info *shinfo;
    
    	size -= SKB_DATA_ALIGN(sizeof(struct skb_shared_info));
    
    	/* Assumes caller memset cleared SKB */
    	skb->truesize = SKB_TRUESIZE(size);
    	refcount_set(&skb->users, 1);
    	skb->head = data;
    	skb->data = data;
    	skb_reset_tail_pointer(skb);
    	skb_set_end_offset(skb, size);
    	skb->mac_header = (typeof(skb->mac_header))~0U;
    	skb->transport_header = (typeof(skb->transport_header))~0U;
    	skb->alloc_cpu = raw_smp_processor_id();
    	/* make sure we initialize shinfo sequentially */
    	shinfo = skb_shinfo(skb);
    	memset(shinfo, 0, offsetof(struct skb_shared_info, dataref));
    	atomic_set(&shinfo->dataref, 1);
    
    	skb_set_kcov_handle(skb, kcov_common_handle());
    }
    ```
    

  

- ice_construct_skb() code
    
    ```JavaScript
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
    

skb를 할당하고 page 데이터를 넣는다. rx_ring은 rx descriptor ring으로 pkt transact에 사용된다.

  

skb_record_rx_queue()

- [skb](https://codebrowser.dev/linux/linux/include/linux/skbuff.h.html#1200skb)->[queue_mapping](https://codebrowser.dev/linux/linux/include/linux/skbuff.h.html#sk_buff::queue_mapping) = [rx_queue](https://codebrowser.dev/linux/linux/include/linux/skbuff.h.html#1201rx_queue) + 1;
- 사용가능한 rx queue 옮기기

ip header 에서 다르다는게 밝혀지면 tcp header부분은 그냥 넘어가는지, 그래도 끝까지 실행되는지