```JavaScript
static void
ice_tx_map(struct ice_tx_ring *tx_ring, struct ice_tx_buf *first,
	   struct ice_tx_offload_params *off)
{
	u64 td_offset, td_tag, td_cmd;
	u16 i = tx_ring->next_to_use;
	unsigned int data_len, size;
	struct ice_tx_desc *tx_desc;
	struct ice_tx_buf *tx_buf;
	struct sk_buff *skb;
	skb_frag_t *frag;
	dma_addr_t dma;
	bool kick;

	td_tag = off->td_l2tag1;
	td_cmd = off->td_cmd;
	td_offset = off->td_offset;
	skb = first->skb;

	data_len = skb->data_len;
	size = skb_headlen(skb);

	tx_desc = ICE_TX_DESC(tx_ring, i);

	if (first->tx_flags & ICE_TX_FLAGS_HW_VLAN) {
		td_cmd |= (u64)ICE_TX_DESC_CMD_IL2TAG1;
		td_tag = first->vid;
	}

	dma = dma_map_single(tx_ring->dev, skb->data, size, DMA_TO_DEVICE);

	tx_buf = first;

	for (frag = &skb_shinfo(skb)->frags[0];; frag++) {
		unsigned int max_data = ICE_MAX_DATA_PER_TXD_ALIGNED;

		if (dma_mapping_error(tx_ring->dev, dma))
			goto dma_error;

		/* record length, and DMA address */
		dma_unmap_len_set(tx_buf, len, size);
		dma_unmap_addr_set(tx_buf, dma, dma);

		/* align size to end of page */
		max_data += -dma & (ICE_MAX_READ_REQ_SIZE - 1);
		tx_desc->buf_addr = cpu_to_le64(dma);

		/* account for data chunks larger than the hardware
		 * can handle
		 */
		while (unlikely(size > ICE_MAX_DATA_PER_TXD)) {
			tx_desc->cmd_type_offset_bsz =
				ice_build_ctob(td_cmd, td_offset, max_data,
					       td_tag);

			tx_desc++;
			i++;

			if (i == tx_ring->count) {
				tx_desc = ICE_TX_DESC(tx_ring, 0);
				i = 0;
			}

			dma += max_data;
			size -= max_data;

			max_data = ICE_MAX_DATA_PER_TXD_ALIGNED;
			tx_desc->buf_addr = cpu_to_le64(dma);
		}

		if (likely(!data_len))
			break;

		tx_desc->cmd_type_offset_bsz = ice_build_ctob(td_cmd, td_offset,
							      size, td_tag);

		tx_desc++;
		i++;

		if (i == tx_ring->count) {
			tx_desc = ICE_TX_DESC(tx_ring, 0);
			i = 0;
		}

		size = skb_frag_size(frag);
		data_len -= size;

		dma = skb_frag_dma_map(tx_ring->dev, frag, 0, size,
				       DMA_TO_DEVICE);

		tx_buf = &tx_ring->tx_buf[i];
		tx_buf->type = ICE_TX_BUF_FRAG;
	}

	/* record SW timestamp if HW timestamp is not available */
	skb_tx_timestamp(first->skb);

	i++;
	if (i == tx_ring->count)
		i = 0;

	/* write last descriptor with RS and EOP bits */
	td_cmd |= (u64)ICE_TXD_LAST_DESC_CMD;
	tx_desc->cmd_type_offset_bsz =
			ice_build_ctob(td_cmd, td_offset, size, td_tag);

	/* Force memory writes to complete before letting h/w know there
	 * are new descriptors to fetch.
	 *
	 * We also use this memory barrier to make certain all of the
	 * status bits have been updated before next_to_watch is written.
	 */
	wmb();

	/* set next_to_watch value indicating a packet is present */
	first->next_to_watch = tx_desc;

	tx_ring->next_to_use = i;

	ice_maybe_stop_tx(tx_ring, DESC_NEEDED);

	/* notify HW of packet */
	kick = __netdev_tx_sent_queue(txring_txq(tx_ring), first->bytecount,
				      netdev_xmit_more());
	if (kick)
		/* notify HW of packet */
		writel(i, tx_ring->tail);

	return;

dma_error:
	/* clear DMA mappings for failed tx_buf map */
	for (;;) {
		tx_buf = &tx_ring->tx_buf[i];
		ice_unmap_and_free_tx_buf(tx_ring, tx_buf);
		if (tx_buf == first)
			break;
		if (i == 0)
			i = tx_ring->count;
		i--;
	}

	tx_ring->next_to_use = i;
}
```