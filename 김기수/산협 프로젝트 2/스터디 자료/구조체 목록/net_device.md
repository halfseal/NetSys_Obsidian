---
locate: include/linux/netdevice.h
---
### Field

---

```C
__cacheline_group_begin(net_device_read_tx);
	unsigned long long	priv_flags;
	const struct net_device_ops *netdev_ops;
	const struct header_ops *header_ops;
	struct netdev_queue	*_tx;
	netdev_features_t	gso_partial_features;
	unsigned int		real_num_tx_queues;
	unsigned int		gso_max_size;
	unsigned int		gso_ipv4_max_size;
	u16			gso_max_segs;
	s16			num_tc;
	unsigned int		mtu;
	unsigned short		needed_headroom;
	struct netdev_tc_txq	tc_to_txq[TC_MAX_QUEUE];
	__cacheline_group_end(net_device_read_tx);

	/* TXRX read-mostly hotpath */
	__cacheline_group_begin(net_device_read_txrx);
	union {
		struct pcpu_lstats __percpu		*lstats;
		struct pcpu_sw_netstats __percpu	*tstats;
		struct pcpu_dstats __percpu		*dstats;
	};
	unsigned long		state;
	unsigned int		flags;
	unsigned short		hard_header_len;
	netdev_features_t	features;
	struct inet6_dev __rcu	*ip6_ptr;
	__cacheline_group_end(net_device_read_txrx);

	/* RX read-mostly hotpath */
	__cacheline_group_begin(net_device_read_rx);
	struct bpf_prog __rcu	*xdp_prog;
	struct list_head	ptype_specific;
	int			ifindex;
	unsigned int		real_num_rx_queues;
	struct netdev_rx_queue	*_rx;
	unsigned long		gro_flush_timeout;
	int			napi_defer_hard_irqs;
	unsigned int		gro_max_size;
	unsigned int		gro_ipv4_max_size;
	rx_handler_func_t __rcu	*rx_handler;
	void __rcu		*rx_handler_data;
	possible_net_t			nd_net;
	unsigned int		min_mtu;
	unsigned int		max_mtu;
	unsigned short		type;
	unsigned char		min_header_len;
	unsigned char		name_assign_type;
	unsigned int		operstate;
	unsigned char		link_mode;

	unsigned char		if_port;
	unsigned char		dma;
	struct device		dev;
	const struct attribute_group *sysfs_groups[4];
	const struct attribute_group *sysfs_rx_queue_group;

	const struct rtnl_link_ops *rtnl_link_ops;

	const struct netdev_stat_ops *stat_ops;

	/* Interface address info. */
	unsigned char		perm_addr[MAX_ADDR_LEN];
	unsigned char		addr_assign_type;
	unsigned char		addr_len;
	unsigned char		upper_level;
	unsigned char		lower_level;

	unsigned short		neigh_priv_len;
	unsigned short          dev_id;
	unsigned short          dev_port;
	unsigned short		padded;

	spinlock_t		addr_list_lock;
	int			irq;

	struct netdev_hw_addr_list	uc;
	struct netdev_hw_addr_list	mc;
	struct netdev_hw_addr_list	dev_addrs;
	struct phy_device	*phydev;
	struct sfp_bus		*sfp_bus;
	struct lock_class_key	*qdisc_tx_busylock;
	bool			proto_down;
	unsigned		wol_enabled:1;
	unsigned		threaded:1;

	struct list_head	net_notifier_list;
```

  

### Description

---

network와 관련된 device….