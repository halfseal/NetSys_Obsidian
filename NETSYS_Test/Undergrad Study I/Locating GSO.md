---
creator: 교수님
type: Hands-On
created: 2022-06-21
---
**Highly probably wrong**

skb_gso_segment() in ip_finish_output_gso() in __ip_finish_output() in ip_finish_output() in ip_output() at net/ipv4/ip_output.c

```c
static int __ip_finish_output(struct net *net, struct sock *sk, struct sk_buff *skb)
{
  unsigned int mtu;

#if defined(CONFIG_NETFILTER) && defined(CONFIG_XFRM)
  /* Policy lookup after SNAT yielded a new policy */
  if (skb_dst(skb)->xfrm) {
    IPCB(skb)->flags |= IPSKB_REROUTED;
    return dst_output(net, sk, skb);
  }
#endif
  mtu = ip_skb_dst_mtu(sk, skb);
  if (skb_is_gso(skb))
    return **ip_finish_output_gso**(net, sk, skb, mtu);

  if (skb->len > mtu || IPCB(skb)->frag_max_size)
    return ip_fragment(net, sk, skb, mtu, ip_finish_output2);

  return ip_finish_output2(net, sk, skb);
}
```

```c
static int ip_finish_output_gso(struct net *net, struct sock *sk,
        struct sk_buff *skb, unsigned int mtu)
{
  netdev_features_t features;
  struct sk_buff *segs;
  int ret = 0;

  /* common case: seglen is <= mtu
   */
  if (skb_gso_validate_network_len(skb, mtu))
    return ip_finish_output2(net, sk, skb);

  /* Slowpath -  GSO segment length exceeds the egress MTU.
   *
   * This can happen in several cases:
   *  - Forwarding of a TCP GRO skb, when DF flag is not set.
   *  - Forwarding of an skb that arrived on a virtualization interface
   *    (virtio-net/vhost/tap) with TSO/GSO size set by other network
   *    stack.
   *  - Local GSO skb transmitted on an NETIF_F_TSO tunnel stacked over an
   *    interface with a smaller MTU.
   *  - Arriving GRO skb (or GSO skb in a virtualized environment) that is
   *    bridged to a NETIF_F_TSO tunnel stacked over an interface with an
   *    insufficent MTU.
   */
  features = netif_skb_features(skb);
  BUILD_BUG_ON(sizeof(*IPCB(skb)) > SKB_SGO_CB_OFFSET);
  segs = **skb_gso_segment**(skb, features & ~NETIF_F_GSO_MASK);
  if (IS_ERR_OR_NULL(segs)) {
    kfree_skb(skb);
    return -ENOMEM;
  }

  consume_skb(skb);

  do {
    struct sk_buff *nskb = segs->next;
    int err;

    skb_mark_not_on_list(segs);
    err = ip_fragment(net, sk, segs, mtu, ip_finish_output2);

    if (err && ret == 0)
      ret = err;
    segs = nskb;
  } while (segs);

  return ret;
}
```

skb_gso_segment in include/linux/netdevice.h

__skb_gso_segment() in net/core/dev.c

```c
/**
 *  __skb_gso_segment - Perform segmentation on skb.
 *  @skb: buffer to segment
 *  @features: features for the output path (see dev->features)
 *  @tx_path: whether it is called in TX path
 *
 *  This function segments the given skb and returns a list of segments.
 *
 *  It may return NULL if the skb requires no segmentation.  This is
 *  only possible when GSO is used for verifying header integrity.
 *
 *  Segmentation preserves SKB_SGO_CB_OFFSET bytes of previous skb cb.
 */
struct sk_buff *__skb_gso_segment(struct sk_buff *skb,
          netdev_features_t features, bool tx_path)
{
  struct sk_buff *segs;

  if (unlikely(skb_needs_check(skb, tx_path))) {
    int err;

    /* We're going to init ->check field in TCP or UDP header */
    err = skb_cow_head(skb, 0);
    if (err < 0)
      return ERR_PTR(err);
  }

  /* Only report GSO partial support if it will enable us to
   * support segmentation on this frame without needing additional
   * work.
   */
  if (features & NETIF_F_GSO_PARTIAL) {
    netdev_features_t partial_features = NETIF_F_GSO_ROBUST;
    struct net_device *dev = skb->dev;

    partial_features |= dev->features & dev->gso_partial_features;
    if (!skb_gso_ok(skb, features | partial_features))
      features &= ~NETIF_F_GSO_PARTIAL;
  }

  BUILD_BUG_ON(SKB_SGO_CB_OFFSET +
         sizeof(*SKB_GSO_CB(skb)) > sizeof(skb->cb));

  SKB_GSO_CB(skb)->mac_offset = skb_headroom(skb);
  SKB_GSO_CB(skb)->encap_level = 0;

  skb_reset_mac_header(skb);
  skb_reset_mac_len(skb);

  segs = **skb_mac_gso_segment**(skb, features);

  if (unlikely(skb_needs_check(skb, tx_path) && !IS_ERR(segs)))
    skb_warn_bad_offload(skb);

  return segs;
}
EXPORT_SYMBOL(__skb_gso_segment);
```

skb_mac_gso_segment() in the same file (net/core/dev.c)

```c

/**
 *  dev_add_offload - register offload handlers
 *  @po: protocol offload declaration
 *
 *  Add protocol offload handlers to the networking stack. The passed
 *  &proto_offload is linked into kernel lists and may not be freed until
 *  it has been removed from the kernel lists.
 *
 *  This call does not sleep therefore it can not
 *  guarantee all CPU's that are in middle of receiving packets
 *  will see the new offload handlers (until the next received packet).
 */
void **dev_add_offload**(struct packet_offload *po)
{
  struct packet_offload *elem;

  spin_lock(&offload_lock);
  list_for_each_entry(elem, &**offload_base**, list) {
    if (po->priority < elem->priority)
      break;
  }
  list_add_rcu(&po->list, elem->list.prev);
  spin_unlock(&offload_lock);
}
EXPORT_SYMBOL(dev_add_offload);

/**
 *  skb_mac_gso_segment - mac layer segmentation handler.
 *  @skb: buffer to segment
 *  @features: features for the output path (see dev->features)
 */
struct sk_buff *skb_mac_gso_segment(struct sk_buff *skb,
            netdev_features_t features)
{
  struct sk_buff *segs = ERR_PTR(-EPROTONOSUPPORT);
  struct packet_offload *ptype;
  int vlan_depth = skb->mac_len;
  __be16 type = skb_network_protocol(skb, &vlan_depth);

  if (unlikely(!type))
    return ERR_PTR(-EINVAL);

  __skb_pull(skb, vlan_depth);

  rcu_read_lock();
	**// NEED to check the types. ptype? but callbacks are in struct net_offload**
  list_for_each_entry_rcu(ptype, &**offload_base**, list) {
    if (ptype->type == type && ptype->callbacks.gso_segment) {
      **segs = ptype->callbacks.gso_segment(skb, features);**
      break;
    }
  }
  rcu_read_unlock();

  __skb_push(skb, skb->data - skb_mac_header(skb));

  return segs;
}
EXPORT_SYMBOL(skb_mac_gso_segment);
```

net/ipv4/af_inet.c

```c
static int __init ipv4_offload_init(void)
{
  /*
   * Add offloads
   */
  if (udpv4_offload_init() < 0)
    pr_crit("%s: Cannot add UDP protocol offload\\n", __func__);
  if (tcpv4_offload_init() < 0)
    pr_crit("%s: Cannot add TCP protocol offload\\n", __func__);
  if (ipip_offload_init() < 0)
    pr_crit("%s: Cannot add IPIP protocol offload\\n", __func__);

  **dev_add_offload**(&ip_packet_offload);
  return 0;
}

/*
 *  IP protocol layer initialiser
 */

static struct packet_offload ip_packet_offload __read_mostly = {
  .type = cpu_to_be16(ETH_P_IP),
  .callbacks = {
    .gso_segment = inet_gso_segment,
    .gro_receive = inet_gro_receive,
    .gro_complete = inet_gro_complete,
  },
};

struct sk_buff *inet_gso_segment(struct sk_buff *skb,
         netdev_features_t features)
{
...

ops = rcu_dereference(inet_offloads[proto]);
  if (likely(ops && ops->callbacks.gso_segment))
    segs = ops->callbacks.**gso_segment**(skb, features); // This must be the TCP GSO
...

// L3 GSO is done here!
}
```

net/core/dev.c

```c
/**
 *  dev_add_offload - register offload handlers
 *  @po: protocol offload declaration
 *
 *  Add protocol offload handlers to the networking stack. The passed
 *  &proto_offload is linked into kernel lists and may not be freed until
 *  it has been removed from the kernel lists.
 *
 *  This call does not sleep therefore it can not
 *  guarantee all CPU's that are in middle of receiving packets
 *  will see the new offload handlers (until the next received packet).
 */
void dev_add_offload(struct **packet_offload** *po)
{
  struct packet_offload *elem;

  spin_lock(&offload_lock);
  list_for_each_entry(elem, &offload_base, list) {
    if (po->priority < elem->priority)
      break;
  }
  list_add_rcu(&po->list, elem->list.prev);
  spin_unlock(&offload_lock);
}
EXPORT_SYMBOL(dev_add_offload);
```

TODO: Where are the followings refered? Maybe in a function inet_gso_segment from af_inet.c

net/ipv4/tcp_offload.c

```c
struct sk_buff *tcp_gso_segment(struct sk_buff *skb,
        netdev_features_t features)
{
	// DO SOMETHING HERE!! (Related to seq, etc.)
	...
	segs = skb_segment(skb, features); // NEED to check what's going on in this func.
  ...

}

static struct sk_buff *tcp4_gso_segment(struct sk_buff *skb,
          netdev_features_t features)
{
  if (!(skb_shinfo(skb)->gso_type & SKB_GSO_TCPV4))
    return ERR_PTR(-EINVAL);

  if (!pskb_may_pull(skb, sizeof(struct tcphdr)))
    return ERR_PTR(-EINVAL);

  if (unlikely(skb->ip_summed != CHECKSUM_PARTIAL)) {
    const struct iphdr *iph = ip_hdr(skb);
    struct tcphdr *th = tcp_hdr(skb);

    /* Set up checksum pseudo header, usually expect stack to
     * have done this already.
     */

    th->check = 0;
    skb->ip_summed = CHECKSUM_PARTIAL;
    __tcp_v4_send_check(skb, iph->saddr, iph->daddr);
  }

  return tcp_gso_segment(skb, features);
}

static const struct net_offload tcpv4_offload = {
  .callbacks = {
    .gso_segment  = tcp4_gso_segment,
    .gro_receive  = tcp4_gro_receive,
    .gro_complete = tcp4_gro_complete,
  },
};
```