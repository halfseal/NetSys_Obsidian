---
creator: 홍권
type: Glossary
created: 2022-07-12
---
# Why do we need Segmentation Offloads?

[http://www.packetinside.com/2013/02/mtu-1500.html](http://www.packetinside.com/2013/02/mtu-1500.html)

10G 네트워크 링크를 사용하는 경우 1500 바이트 MTU 로 계산하면 대략 초당 800,000 packet를 처리해야 한다. 큰 CPU 오버헤드로 이어짐.

이런 오버헤드를 줄이기 위해 MTU를 늘려보자.

→ MTU를 [점보프레임](http://www.packetinside.com/2012/03/jumbo-frame.html)인 9000 바이트로 늘리면?

→ 인터넷에 연결되어 있는 모든 구간이 저 MTU 값을 사용하지 못한다. 보통의 경우는 1500 바이트 이내일 것이다. 내부 네트워크라면 모르지만 인터넷 구간에서는 현실적으로 사용이 힘들어진다.

그래서 네트워크 어뎁터에서 채용하기 시작한 것이 TSO 이다.

TSO 호환 가능 NIC는 커널상에서 MTU보다 큰 패킷데이터를 받고, 이를 외부로 전송할 때 NIC에서 데이터를 재-세그먼트 하여 MTU에 맞게 보내게 된다.

# Segmentation Offloads & Details

[https://www.kernel.org/doc/html/latest/networking/segmentation-offloads.html](https://www.kernel.org/doc/html/latest/networking/segmentation-offloads.html)

[https://blogs.gnome.org/markmc/2008/05/28/checksums-scatter-gather-io-and-segmentation-offload/](https://blogs.gnome.org/markmc/2008/05/28/checksums-scatter-gather-io-and-segmentation-offload/)

### **TSO(TCP Segmentation Offload)**

> TSO is the ability of some network devices to take a frame and break it down to smaller (i.e. MTU) sized frames before transmitting. This is done by breaking the TCP payload into segments and using the same IP header with each of the segments.

- support one of two types in terms of the **IP ID**.
    
    - IP header’s ID field & IP fragmentation
        
        [CellStream - The Purpose of the IP ID Field Demystified](https://www.cellstream.com/reference-reading/tipsandtricks/314-the-purpose-of-the-ip-id-field-demystified)
        ![[Untitled(63).png]]
        
        → IPv4 header. Identification field = used for IP packet fragmentation & reassembly
        
        ### When fragmentation is needed?
        
        IPv4’s maximum datagram size = 2^16(size of Total Length field) = 65535 bytes
        
        Ethernet’s maximum transmission unit size(MTU) = 1500 bytes
        
        → Usually, IP protocol stack set the packet’s maximum size to fit Ethernet’s MTU(1500 bytes) to prevent fragmentaiton.
        
        → But still IP packet can go into some different interface with smaller MTU.
        
        Example)
        
        IP packet’s size = 4000 bytes(=3980 payload + 20 header)
        
        MTU = 1500 bytes

| Order | Payload size         | Header size | “More” flag | Offset(8 bytes unit) |
| ----- | -------------------- | ----------- | ----------- | -------------------- |
| 1     | 1480                 | 20          | 1           | 0                    |
| 2     | 1480                 | 20          | 1           | 185                  |
| 3     | 1020(3980-1480-1480) | 20          | 0           | 370                  |

        
        ![[Untitled(64).png]]
        
        ![[Untitled(65).png]]
        
        
    
    1. default behavior is to increment the IP ID with every segment.
    2. If the GSO type SKB_GSO_TCP_FIXEDID is specified then we will not increment the IP ID(all segments will use the same IP ID).
    
    - If a device has NETIF_F_TSO_MANGLEID set then the IP ID can be ignored when performing TSO → either a. or b. based on driver preference.

[short history of GSO](https://wiki.geant.org/pages/releaseview.action?pageId=121340568)

### **GSO(Generic Segmentation Offload)**

> GSO is a generalisation of TSO in the kernel. The idea is that you **delay segmenting a packet until the last possible moment**.

1. If the device does support TSO, the unsegmented skb would be passed to the driver.
2. In the case where a device doesn’t support TSO, this would be just before passing the skb to the driver.
    - See **dev_hard_start_xmit()** for where **dev_gso_segment()** is used to segment a frame before passing to the driver in the case where the device does not support GSO(TSO를 오타낸 게 아닐까?).
        ![[Untitled(66).png]]
        
        그런데 dev_gso_segment()를 발견할 수 없었다.
        
        TODO: Why newer version doesn’t have dev_gso_segment()? 대체된걸까? 혹은 불필요해진걸까?
        
        2008년 글이라 해당 년도의 커널 버전 확인,
        ![[Untitled(67).png]]
        
        [v2.6의 /net/core/dev.c](https://github.com/torvalds/linux/blob/v2.6.39/net/core/dev.c)를 확인해봤다.
        
        ```c
        int dev_hard_start_xmit(struct sk_buff *skb, struct net_device *dev,
        			struct netdev_queue *txq)
        {
        	const struct net_device_ops *ops = dev->netdev_ops;
        	int rc = NETDEV_TX_OK;
        
        	if (likely(!skb->next)) {
        		u32 features;
        
        		/*
        		 * If device doesn't need skb->dst, release it right now while
        		 * its hot in this cpu cache
        		 */
        		if (dev->priv_flags & IFF_XMIT_DST_RELEASE)
        			skb_dst_drop(skb);
        
        		if (!list_empty(&ptype_all))
        			dev_queue_xmit_nit(skb, dev);
        
        		skb_orphan_try(skb);
        
        		features = netif_skb_features(skb);
        
        		if (vlan_tx_tag_present(skb) &&
        		    !(features & NETIF_F_HW_VLAN_TX)) {
        			skb = __vlan_put_tag(skb, vlan_tx_tag_get(skb));
        			if (unlikely(!skb))
        				goto out;
        
        			skb->vlan_tci = 0;
        		}
        
        		if (netif_needs_gso(skb, features)) {
        			if (unlikely(**dev_gso_segment**(skb, features)))
        				goto out_kfree_skb;
        			if (skb->next)
        				goto gso;
        		} else {
        			if (skb_needs_linearize(skb, features) &&
        			    __skb_linearize(skb))
        				goto out_kfree_skb;
        
        			/* If packet is not checksummed and device does not
        			 * support checksumming for this protocol, complete
        			 * checksumming here.
        			 */
        			if (skb->ip_summed == CHECKSUM_PARTIAL) {
        				skb_set_transport_header(skb,
        					skb_checksum_start_offset(skb));
        				if (!(features & NETIF_F_ALL_CSUM) &&
        				     skb_checksum_help(skb))
        					goto out_kfree_skb;
        			}
        		}
        
        		rc = ops->ndo_start_xmit(skb, dev);
        		trace_net_dev_xmit(skb, rc);
        		if (rc == NETDEV_TX_OK)
        			txq_trans_update(txq);
        		return rc;
        	}
        
        gso:
        	do {
        		struct sk_buff *nskb = skb->next;
        
        		skb->next = nskb->next;
        		nskb->next = NULL;
        
        		/*
        		 * If device doesn't need nskb->dst, release it right now while
        		 * its hot in this cpu cache
        		 */
        		if (dev->priv_flags & IFF_XMIT_DST_RELEASE)
        			skb_dst_drop(nskb);
        
        		rc = ops->ndo_start_xmit(nskb, dev);
        		trace_net_dev_xmit(nskb, rc);
        		if (unlikely(rc != NETDEV_TX_OK)) {
        			if (rc & ~NETDEV_TX_MASK)
        				goto out_kfree_gso_skb;
        			nskb->next = skb->next;
        			skb->next = nskb;
        			return rc;
        		}
        		txq_trans_update(txq);
        		if (unlikely(netif_tx_queue_stopped(txq) && skb->next))
        			return NETDEV_TX_BUSY;
        	} while (skb->next);
        
        out_kfree_gso_skb:
        	if (likely(skb->next == NULL))
        		skb->destructor = DEV_GSO_CB(skb)->destructor;
        out_kfree_skb:
        	kfree_skb(skb);
        out:
        	return rc;
        }
        ```
        
        dev_gso_segment()를 확인할 수 있었다.
        
        ```c
        /**
         *	dev_gso_segment - Perform emulated hardware segmentation on skb.
         *	@skb: buffer to segment
         *	@features: device features as applicable to this skb
         *
         *	This function segments the given skb and stores the list of segments
         *	in skb->next.
         */
        static int dev_gso_segment(struct sk_buff *skb, int features)
        {
        	struct sk_buff *segs;
        
        	segs = skb_gso_segment(skb, features);
        
        	/* Verifying header integrity only. */
        	if (!segs)
        		return 0;
        
        	if (IS_ERR(segs))
        		return PTR_ERR(segs);
        
        	skb->next = segs;
        	DEV_GSO_CB(skb)->destructor = skb->destructor;
        	skb->destructor = dev_gso_skb_destructor;
        
        	return 0;
        }
        
        /*
         * Try to orphan skb early, right before transmission by the device.
         * We cannot orphan skb if tx timestamp is requested or the sk-reference
         * is needed on driver level for other reasons, e.g. see net/can/raw.c
         */
        static inline void skb_orphan_try(struct sk_buff *skb)
        {
        	struct sock *sk = skb->sk;
        
        	if (sk && !skb_shinfo(skb)->tx_flags) {
        		/* skb_tx_hash() wont be able to get sk.
        		 * We copy sk_hash into skb->rxhash
        		 */
        		if (!skb->rxhash)
        			skb->rxhash = sk->sk_hash;
        		skb_orphan(skb);
        	}
        }
        ```
        
        skb_gso_segment()
        
        ```c
        /**
         *	skb_gso_segment - Perform segmentation on skb.
         *	@skb: buffer to segment
         *	@features: features for the output path (see dev->features)
         *
         *	This function segments the given skb and returns a list of segments.
         *
         *	It may return NULL if the skb requires no segmentation.  This is
         *	only possible when GSO is used for verifying header integrity.
         */
        
        struct sk_buff *skb_gso_segment(struct sk_buff *skb, u32 features)
        {
        	struct sk_buff *segs = ERR_PTR(-EPROTONOSUPPORT);
        	struct packet_type *ptype;
        	__be16 type = skb->protocol;
        	int vlan_depth = ETH_HLEN;
        	int err;
        
        	while (type == htons(ETH_P_8021Q)) {
        		struct vlan_hdr *vh;
        
        		if (unlikely(!pskb_may_pull(skb, vlan_depth + VLAN_HLEN)))
        			return ERR_PTR(-EINVAL);
        
        		vh = (struct vlan_hdr *)(skb->data + vlan_depth);
        		type = vh->h_vlan_encapsulated_proto;
        		vlan_depth += VLAN_HLEN;
        	}
        
        	skb_reset_mac_header(skb);
        	skb->mac_len = skb->network_header - skb->mac_header;
        	__skb_pull(skb, skb->mac_len);
        
        	if (unlikely(skb->ip_summed != CHECKSUM_PARTIAL)) {
        		struct net_device *dev = skb->dev;
        		struct ethtool_drvinfo info = {};
        
        		if (dev && dev->ethtool_ops && dev->ethtool_ops->get_drvinfo)
        			dev->ethtool_ops->get_drvinfo(dev, &info);
        
        		WARN(1, "%s: caps=(0x%lx, 0x%lx) len=%d data_len=%d ip_summed=%d\\n",
        		     info.driver, dev ? dev->features : 0L,
        		     skb->sk ? skb->sk->sk_route_caps : 0L,
        		     skb->len, skb->data_len, skb->ip_summed);
        
        		if (skb_header_cloned(skb) &&
        		    (err = pskb_expand_head(skb, 0, 0, GFP_ATOMIC)))
        			return ERR_PTR(err);
        	}
        
        	rcu_read_lock();
        	list_for_each_entry_rcu(ptype,
        			&ptype_base[ntohs(type) & PTYPE_HASH_MASK], list) {
        		if (ptype->type == type && !ptype->dev && ptype->gso_segment) {
        			if (unlikely(skb->ip_summed != CHECKSUM_PARTIAL)) {
        				err = ptype->gso_send_check(skb);
        				segs = ERR_PTR(err);
        				if (err || skb_gso_ok(skb, features))
        					break;
        				__skb_push(skb, (skb->data -
        						 skb_network_header(skb)));
        			}
        			segs = ptype->gso_segment(skb, features);
        			break;
        		}
        	}
        	rcu_read_unlock();
        
        	__skb_push(skb, skb->data - skb_mac_header(skb));
        
        	return segs;
        }
        EXPORT_SYMBOL(skb_gso_segment);
        ```
        

- **a given skbuff** will have its data broken out over multiple skbuffs that have been resized to match the MSS(provided via skb_shinfo()->gso_size)
    
- Before enabling any hardware segmentation offload(e.g. TSO) a corresponding software offload is required in GSO. Otherwise a frame might be re-routed between devices and end up being unable to be transmitted.
    
    → With GSO, it provides a fallback so that **we may enable TSO for a bridge even if some of its constituents do not support TSO**.
    
- [https://wiki.linuxfoundation.org/networking/gso](https://wiki.linuxfoundation.org/networking/gso)
    
    **In the ideal world**, the segmentation would occur inside each NIC driver where they would
    
    rip the super-packet apart and either produce SG(Scatter Gather) lists which are directly fed to the hardware or linearise each segment into pre-allocated memory to be fed to the NIC.
    
    → This would elminate segmented skb's altogether.
    
    - Scatter-Gather DMA
        
        The best scenario is obviously the case where the underlying NIC supports SG. This means that we **simply have to manipulate the SG entries** and **place them into individual skb's before passing them to the driver**.
        
        The worst scenario is where the NIC does not support SG and the user uses write which means that **we have to copy the data twice**.
        
        [Scatter-Gather DMA](https://m.blog.naver.com/sysganda/30169551168)
        
        → 기본적으로 DMA는 연속된 메모리 공간으로의 전송만 지원했는데, SG-DMA는 분산(Scatter)해서 쓰고 CPU가 이를 모아(Gather)서 접근 하는 것이 (한번에)가능하도록 한다.
        
        [NDIS Scatter/Gather DMA - Windows drivers](https://docs.microsoft.com/en-us/windows-hardware/drivers/network/ndis-scatter-gather-dma)
        

### **GRO(Generic Receive Offload)**

- complement to GSO
- any frame assembled by GRO should be segmented to create an identical sequence of frames using GSO
- any sequence of frames segmented by GSO should be able to be reassembled back to the original by GRO
- The only exception to this is **IPv4 ID** in the case that the **DF(Don’t Fragement) bit is set** for a given IP header.
    - [https://stackoverflow.com/a/60730822/16728567](https://stackoverflow.com/a/60730822/16728567), RFC 791 it says:
        
        > If the Don't Fragment flag (DF) bit is set, then internet fragmentation of this datagram is NOT permitted, although it may be discarded. This can be used to prohibit fragmentation in cases where the receiving host does not have sufficient resources to reassemble internet fragments.
        
    - **If the 'DF' bit is set on packets**, a router which normally would fragment **a packet larger than MTU** (and potentially deliver it out of order), **instead will drop the packet.**
        
    - Then, the router is expected to send "ICMP Fragmentation Needed" packet, allowing the **sending host to account for the lower MTU** on the path to the destination host.
        
    - **The sending side will then reduce** **its** estimate of the connection's **Path MTU** (Maximum Transmission Unit) and re-send in smaller segments. This process is called PMTU-D ("Path MTU Discovery").
        

---

# Linux kernel의 TOE support patch 제안(2005)

[Linux and TCP offload engines](https://lwn.net/Articles/148697/)

Cehlsio Communications가 제안한 이 패치([https://lwn.net/Articles/147289/](https://lwn.net/Articles/147289/))는 generic한 “open TOE” framework의 형태로 TOE를 지원함.

그러나 아래와 같은 이유들로 TOE support patch는 kernel에 통합되지 못할 것이라고 함.

1. TOE는 필연적으로 Linux TCP implementation 깊숙히 hook 될 수 밖에 없고 이는 결국 미래에 짐이 된다.
2. 또한 버그나 보안 이슈가 생기면 여태까진 쉽게 고칠 수 있었으나 TOE adapter의 문제라면 고치기 어려워진다.
3. TOE adapter(+S/W 지원)가 지금의 단순한 adapter들을 고속 인터넷(e.g. 10G) 환경에선 능가한다고 해도, 추후에 10G adapter가 보급될 시기엔 별 차이가 안나게 된다. 즉, TOE로 얻는 성능 향상은 일시적이지만, code는 평생 유지보수 되어야 한다.

그래서 결국은?

[Wiki](https://wiki.linuxfoundation.org/networking/toe)

여러 벤더들이 TOE를 kernel에 포함시키려는 시도를 했음에도 불구하고 끝끝내 reject 되었다고 함.

### TOE ≠ TSO or TRO

TOE는 TCP/IP stack을 fully/partially offload하고자 한다면

segmentation offload는 그 중 segmentation만 NIC에 offload하는 것.