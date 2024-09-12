---
locate: drivers/net/ethernet/intel/ice/ice.h
---
### Field

---

```C
	struct ice_vsi *vsi;
	struct ice_repr *repr;
	/* indirect block callbacks on registered higher level devices
	 * (e.g. tunnel devices)
	 *
	 * tc_indr_block_cb_priv_list is used to look up indirect callback
	 * private data
	 */
	struct list_head tc_indr_block_priv_list;
```

  

### Description

---

net device의 private한 정보에 대한 구조체