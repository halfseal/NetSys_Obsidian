---
locate: include/linux/device.h
---
### Field

---

```C
	struct kobject kobj;
	struct device		*parent;
	struct device_private	*p;

	const char		*init_name; /* initial name of the device */
	const struct device_type *type;

	const struct bus_type	*bus;	/* type of bus device is on */
	struct device_driver *driver;	/* which driver has allocated this device */
	void	*platform_data;	/* Platform specific data, device core doesn't touch it */
	void	*driver_data;	/* Driver data, set and get with dev_set_drvdata/dev_get_drvdata */
	struct mutex	mutex;	/* mutex to synchronize calls to its driver.*/

	struct dev_links_info	links;
	struct dev_pm_info	power;
	struct dev_pm_domain	*pm_domain;
	
	struct dev_msi_info	msi;
	u64		*dma_mask;	/* dma mask (if dma'able device) */
	u64		coherent_dma_mask; /* Like dma_mask, but for alloc_coherent mappings as not all hardware supports 64 bit addresses for consistent allocations such descriptors. */
	u64		bus_dma_limit;	/* upstream dma constraint */
	const struct bus_dma_region *dma_range_map;

	struct device_dma_parameters *dma_parms;

	struct list_head	dma_pools;	/* dma pools (if dma'ble) */
	dev_t			devt;	/* dev_t, creates the sysfs "dev" */
	u32			id;	/* device instance */

	spinlock_t		devres_lock;
	struct list_head	devres_head;

	const struct class	*class;
	const struct attribute_group **groups;	/* optional groups */

	void	(*release)(struct device *dev);
	struct iommu_group	*iommu_group;
	struct dev_iommu	*iommu;

	struct device_physical_location *physical_location;

	enum device_removable	removable;

	bool			offline_disabled:1;
	bool			offline:1;
	bool			of_node_reused:1;
	bool			state_synced:1;
	bool			can_match:1;
```

  

### Description

---

- parent - 부모 device를 가르키는 포인터, 그 것이 attached된 device를 가르키며, 대부분 버스거나 호스트 컨트롤러를 가르킴. Null 이라면 최상위 device임.
- p - device의 private한 내용을 담고 있음
- kobj - 다른 클래스들이 상속되는 것과 관련된 최상위 추상 클래스
- init-name - device의 초기 이름
- type - device의 타입과 이와 연관된 정보를 가지고 있음
- mutex - 그 드라이버와 동기화하는 호출을 하기 위해 사용되는 mutex
- bus - bus type의 device가 여기에 있음
- driver - 어떤 드라이버가 여기에 할당 되어있는지 나타냄