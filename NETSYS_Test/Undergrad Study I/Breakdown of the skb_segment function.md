---
creator: Maike
type: Hands-On
created: 2022-06-14
---
```
/**
 *	skb_segment - Perform protocol segmentation on skb.
 *	@head_skb: buffer to segment
 *	@features: features for the output path (see dev->features)
 *
 *	This function performs segmentation on the given skb.  It returns
 *	a pointer to the first in a list of new skbs for the segments.
 *	In case of error it returns ERR_PTR(err).
 */
struct sk_buff *skb_segment(struct sk_buff *head_skb,
			    netdev_features_t features)
{
	struct sk_buff *segs = NULL;
	struct sk_buff *tail = NULL;
	struct sk_buff *list_skb = skb_shinfo(head_skb)->frag_list;
	skb_frag_t *frag = skb_shinfo(head_skb)->frags;
	unsigned int mss = skb_shinfo(head_skb)->gso_size;

    //[Maike]   doffset is the distance from MAC header to data 
	unsigned int doffset = head_skb->data - skb_mac_header(head_skb);
	
    struct sk_buff *frag_skb = head_skb;
	unsigned int offset = doffset;
	unsigned int tnl_hlen = skb_tnl_header_len(head_skb);
	unsigned int partial_segs = 0;
	unsigned int headroom;
	unsigned int len = head_skb->len;
	__be16 proto;
	bool csum, sg;
	int nfrags = skb_shinfo(head_skb)->nr_frags;
	int err = -ENOMEM;
	int i = 0;
	int pos;


    
    //===[Maike]=====================================================
    //Checking if skb is from untrusted source using SKB_GSO_DODGY
	
    if (list_skb && !list_skb->head_frag && skb_headlen(list_skb) &&
	    (skb_shinfo(head_skb)->gso_type & SKB_GSO_DODGY)) {
		/* gso_size is untrusted, and we have a frag_list with a linear
		 * non head_frag head.
		 *
		 * (we assume checking the first list_skb member suffices;
		 * i.e if either of the list_skb members have non head_frag
		 * head, then the first one has too).
		 *
		 * If head_skb's headlen does not fit requested gso_size, it
		 * means that the frag_list members do NOT terminate on exact
		 * gso_size boundaries. Hence we cannot perform skb_frag_t page
		 * sharing. Therefore we must fallback to copying the frag_list
		 * skbs; we do so by disabling SG.
		 */
		if (mss != GSO_BY_FRAGS && mss != skb_headlen(head_skb))
			features &= ~NETIF_F_SG;
	}
    
    //===============================================================

    
    //[Maike]   Pushing back the data pointer by doffset to point to the MAC header beginning
    //          Apparently in this function, the entire packet including headers is to be treated as data
	__skb_push(head_skb, doffset);

    //[Maike]   Getting Network Protocol
	proto = skb_network_protocol(head_skb, NULL);
	

    if (unlikely(!proto))
		return ERR_PTR(-EINVAL);
    
    //[Maike]   Setting Scatter/Gather IO to true if it is in the features
	sg = !!(features & NETIF_F_SG);


    //[Maike]   Setting CSUM to true if it is in the features, given the protocol
	csum = !!can_checksum_protocol(features, proto);



    //===[Maike]===========================================================
    //This is probably some sort of setup for the type of segmentation 
    //that has to be performed
    //
    //I assume this, because if mss == GSO_BY_FRAGS, 
    //the function - apparently - is told to fragment using 
    //the "current fragmentation"
    
	if (sg && csum && (mss != GSO_BY_FRAGS))  {

        //[Maike]   entering this if if features does not include GSO partial
		if (!(features & NETIF_F_GSO_PARTIAL)) {
			struct sk_buff *iter;
			unsigned int frag_len;
        
            //[Maike]   If there is no fragment list, or gso is not supported, go to normal segmentation
			if (!list_skb ||
			    !net_gso_ok(features, skb_shinfo(head_skb)->gso_type))
				goto normal;

			/* If we get here then all the required
			 * GSO features except frag_list are supported.
			 * Try to split the SKB to multiple GSO SKBs
			 * with no frag_list.
			 * Currently we can do that only when the buffers don't
			 * have a linear part and all the buffers except
			 * the last are of the same length.
			 */

            //[Maike]   Save the length of the first skb on the fragment list
			frag_len = list_skb->len;

            
			skb_walk_frags(head_skb, iter) {

                //[Maike]   Go to normal if a fragment that is not the last one has a different length than the first one
				if (frag_len != iter->len && iter->next)
					goto normal;
				//[Maike]   Go to normal if a fragment that is not the head fragment has headers
                if (skb_headlen(iter) && !iter->head_frag)
					goto normal;

				len -= iter->len;
			}

            //[Maike]   If the last fragement's length is different from the first one's go to normal
			if (len != frag_len)
				goto normal;
		}

		/* GSO partial only requires that we trim off any excess that
		 * doesn't fit into an MSS sized block, so take care of that
		 * now.
		 */

        //[Maike]   If we went here straight from the if statement, len == head_skb->len
        //          If we processed the entire if statement len == list_skb->len
		partial_segs = len / mss;
        if (partial_segs > 1)
			mss *= partial_segs;
		else
			partial_segs = 0;
	}

    //===============================================================================



    //[Maike]   All of the upper part was just for partial_segs???

normal:
	headroom = skb_headroom(head_skb);
	pos = skb_headlen(head_skb);

	do {
		struct sk_buff *nskb;
		skb_frag_t *nskb_frag;
		int hsize;
		int size;

        //[Maike]   if we are segmenting by fragments, our length is as long as the first fragment's length
		if (unlikely(mss == GSO_BY_FRAGS)) {
			len = list_skb->len;
		} 
        //[Maike]   else, our length is the length of the head_skb without headers (?)
        else {
			len = head_skb->len - offset;
			if (len > mss)
				len = mss;
		}
        
        //[Maike]   this should always be 0?? offset is defined as the distance between MAC header and data
        //          Is this maybe bigger than 0 if 
		hsize = skb_headlen(head_skb) - offset;

        //[Maike]   This branch is taken if all of the below are true:
        //          1. The distance between MAC header and data is equal to the skb's len minus datalen
        //          2. There are no fragments (i=0 at this point, so only taken if nfrags<=0)
        //          3. The list_skb has a header
        //          4. The header of the skb is equal to len, or scatter/gather is true
        //
        //          Skipping this for now, since not taken with fragments!
		if (hsize <= 0 && i >= nfrags && skb_headlen(list_skb) &&
		    (skb_headlen(list_skb) == len || sg)) {
			BUG_ON(skb_headlen(list_skb) > len);

			i = 0;
			nfrags = skb_shinfo(list_skb)->nr_frags;
			frag = skb_shinfo(list_skb)->frags;
			frag_skb = list_skb;
			pos += skb_headlen(list_skb);

			while (pos < offset + len) {
				BUG_ON(i >= nfrags);

				size = skb_frag_size(frag);
				if (pos + size > offset + len)
					break;

				i++;
				pos += size;
				frag++;
			}

			nskb = skb_clone(list_skb, GFP_ATOMIC);
			list_skb = list_skb->next;

			if (unlikely(!nskb))
				goto err;

			if (unlikely(pskb_trim(nskb, len))) {
				kfree_skb(nskb);
				goto err;
			}

			hsize = skb_end_offset(nskb);
			if (skb_cow_head(nskb, doffset + headroom)) {
				kfree_skb(nskb);
				goto err;
			}

			nskb->truesize += skb_end_offset(nskb) - hsize;
			skb_release_head_state(nskb);
			__skb_push(nskb, doffset);
		}
        //[Maike]   The else part of it 
        else {
            //[Maike]   Keeping hsize in range 0 <= hsize <= len
			if (hsize < 0)
				hsize = 0;
			if (hsize > len || !sg)
				hsize = len;

            //[Maike]   Creating new skbuff [TODO] What is the reasoning behind the size?
			nskb = __alloc_skb(hsize + doffset + headroom,
					   GFP_ATOMIC, skb_alloc_rx_flag(head_skb),
					   NUMA_NO_NODE);

			if (unlikely(!nskb))
				goto err;
            
            //[Maike]   Reserving headroom space and setting space for data to doffset
			skb_reserve(nskb, headroom);
			__skb_put(nskb, doffset);
		}
        
        //[Maike]   If first round, set newly created skb as beginning of segs, else append it to the end
		if (segs)
			tail->next = nskb;
		else
			segs = nskb;
		tail = nskb;


        //[Maike]   Duplicate header from head skb to new skb
		__copy_skb_header(nskb, head_skb);

        //[Maike]   All of these are probably used for filling the new skb
		skb_headers_offset_update(nskb, skb_headroom(nskb) - headroom);
		skb_reset_mac_len(nskb);
    
        //[Maike]   Copying the data from head_skb, including tunnel header if existing, into the new skb's data field  
		skb_copy_from_linear_data_offset(head_skb, -tnl_hlen,
						 nskb->data - tnl_hlen,
						 doffset + tnl_hlen);


        //[Maike]   Skip ahead if... I really don't know what len is supposed to even mean at this point
		if (nskb->len == len + doffset)
			goto perform_csum_check;

		if (!sg) {
			if (!csum) {
				if (!nskb->remcsum_offload)
					nskb->ip_summed = CHECKSUM_NONE;
				SKB_GSO_CB(nskb)->csum =
					skb_copy_and_csum_bits(head_skb, offset,
							       skb_put(nskb,
								       len),
							       len);
				SKB_GSO_CB(nskb)->csum_start =
					skb_headroom(nskb) + doffset;
			} else {
				skb_copy_bits(head_skb, offset,
					      skb_put(nskb, len),
					      len);
			}
			continue;
		}

		nskb_frag = skb_shinfo(nskb)->frags;

		skb_copy_from_linear_data_offset(head_skb, offset,
						 skb_put(nskb, hsize), hsize);

		skb_shinfo(nskb)->flags |= skb_shinfo(head_skb)->flags &
					   SKBFL_SHARED_FRAG;

		if (skb_orphan_frags(frag_skb, GFP_ATOMIC) ||
		    skb_zerocopy_clone(nskb, frag_skb, GFP_ATOMIC))
			goto err;

		while (pos < offset + len) {
			if (i >= nfrags) {
				i = 0;
				nfrags = skb_shinfo(list_skb)->nr_frags;
				frag = skb_shinfo(list_skb)->frags;
				frag_skb = list_skb;
				if (!skb_headlen(list_skb)) {
					BUG_ON(!nfrags);
				} else {
					BUG_ON(!list_skb->head_frag);

					/* to make room for head_frag. */
					i--;
					frag--;
				}
				if (skb_orphan_frags(frag_skb, GFP_ATOMIC) ||
				    skb_zerocopy_clone(nskb, frag_skb,
						       GFP_ATOMIC))
					goto err;

				list_skb = list_skb->next;
			}

			if (unlikely(skb_shinfo(nskb)->nr_frags >=
				     MAX_SKB_FRAGS)) {
				net_warn_ratelimited(
					"skb_segment: too many frags: %u %u\n",
					pos, mss);
				err = -EINVAL;
				goto err;
			}

			*nskb_frag = (i < 0) ? skb_head_frag_to_page_desc(frag_skb) : *frag;
			__skb_frag_ref(nskb_frag);
			size = skb_frag_size(nskb_frag);

			if (pos < offset) {
				skb_frag_off_add(nskb_frag, offset - pos);
				skb_frag_size_sub(nskb_frag, offset - pos);
			}

			skb_shinfo(nskb)->nr_frags++;

			if (pos + size <= offset + len) {
				i++;
				frag++;
				pos += size;
			} else {
				skb_frag_size_sub(nskb_frag, pos + size - (offset + len));
				goto skip_fraglist;
			}

			nskb_frag++;
		}

skip_fraglist:
		nskb->data_len = len - hsize;
		nskb->len += nskb->data_len;
		nskb->truesize += nskb->data_len;

perform_csum_check:
		if (!csum) {
			if (skb_has_shared_frag(nskb) &&
			    __skb_linearize(nskb))
				goto err;

			if (!nskb->remcsum_offload)
				nskb->ip_summed = CHECKSUM_NONE;
			SKB_GSO_CB(nskb)->csum =
				skb_checksum(nskb, doffset,
					     nskb->len - doffset, 0);
			SKB_GSO_CB(nskb)->csum_start =
				skb_headroom(nskb) + doffset;
		}
	} while ((offset += len) < head_skb->len);

	/* Some callers want to get the end of the list.
	 * Put it in segs->prev to avoid walking the list.
	 * (see validate_xmit_skb_list() for example)
	 */
	segs->prev = tail;

	if (partial_segs) {
		struct sk_buff *iter;
		int type = skb_shinfo(head_skb)->gso_type;
		unsigned short gso_size = skb_shinfo(head_skb)->gso_size;

		/* Update type to add partial and then remove dodgy if set */
		type |= (features & NETIF_F_GSO_PARTIAL) / NETIF_F_GSO_PARTIAL * SKB_GSO_PARTIAL;
		type &= ~SKB_GSO_DODGY;

		/* Update GSO info and prepare to start updating headers on
		 * our way back down the stack of protocols.
		 */
		for (iter = segs; iter; iter = iter->next) {
			skb_shinfo(iter)->gso_size = gso_size;
			skb_shinfo(iter)->gso_segs = partial_segs;
			skb_shinfo(iter)->gso_type = type;
			SKB_GSO_CB(iter)->data_offset = skb_headroom(iter) + doffset;
		}

		if (tail->len - doffset <= gso_size)
			skb_shinfo(tail)->gso_size = 0;
		else if (tail != segs)
			skb_shinfo(tail)->gso_segs = DIV_ROUND_UP(tail->len - doffset, gso_size);
	}

	/* Following permits correct backpressure, for protocols
	 * using skb_set_owner_w().
	 * Idea is to tranfert ownership from head_skb to last segment.
	 */
	if (head_skb->destructor == sock_wfree) {
		swap(tail->truesize, head_skb->truesize);
		swap(tail->destructor, head_skb->destructor);
		swap(tail->sk, head_skb->sk);
	}
	return segs;

err:
	kfree_skb_list(segs);
	return ERR_PTR(err);
}
EXPORT_SYMBOL_GPL(skb_segment);

```