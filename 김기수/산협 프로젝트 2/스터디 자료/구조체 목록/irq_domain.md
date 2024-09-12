### Field

---

```C
	struct list_head		link;
	const char			*name;
	const struct irq_domain_ops	*ops;
	void				*host_data;
	unsigned int			flags;
	unsigned int			mapcount;
	struct mutex			mutex;
	struct irq_domain		*root;

	/* Optional data */
	struct fwnode_handle		*fwnode;
	enum irq_domain_bus_token	bus_token;
	struct irq_domain_chip_generic	*gc;
	struct device			*dev;
	struct device			*pm_dev;
```

  

### Description

---