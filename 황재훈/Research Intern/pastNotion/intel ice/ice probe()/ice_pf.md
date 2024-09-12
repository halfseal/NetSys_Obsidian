```JavaScript
struct ice_pf {
	struct pci_dev *pdev;

	struct devlink_region *nvm_region;
	struct devlink_region *sram_region;
	struct devlink_region *devcaps_region;

	/* devlink port data */
	struct devlink_port devlink_port;

	/* OS reserved IRQ details */
	struct msix_entry *msix_entries;
	struct ice_irq_tracker irq_tracker;
	/* First MSIX vector used by SR-IOV VFs. Calculated by subtracting the
	 * number of MSIX vectors needed for all SR-IOV VFs from the number of
	 * MSIX vectors allowed on this PF.
	 */
	u16 sriov_base_vector;
	unsigned long *sriov_irq_bm;	/* bitmap to track irq usage */
	u16 sriov_irq_size;		/* size of the irq_bm bitmap */

	u16 ctrl_vsi_idx;		/* control VSI index in pf->vsi array */

	struct ice_vsi **vsi;		/* VSIs created by the driver */
	struct ice_vsi_stats **vsi_stats;
	struct ice_sw *first_sw;	/* first switch created by firmware */
	u16 eswitch_mode;		/* current mode of eswitch */
	struct dentry *ice_debugfs_pf;
	struct dentry *ice_debugfs_pf_fwlog;
	/* keep track of all the dentrys for FW log modules */
	struct dentry **ice_debugfs_pf_fwlog_modules;
	struct ice_vfs vfs;
	DECLARE_BITMAP(features, ICE_F_MAX);
	DECLARE_BITMAP(state, ICE_STATE_NBITS);
	DECLARE_BITMAP(flags, ICE_PF_FLAGS_NBITS);
	DECLARE_BITMAP(misc_thread, ICE_MISC_THREAD_NBITS);
	unsigned long *avail_txqs;	/* bitmap to track PF Tx queue usage */
	unsigned long *avail_rxqs;	/* bitmap to track PF Rx queue usage */
	unsigned long serv_tmr_period;
	unsigned long serv_tmr_prev;
	struct timer_list serv_tmr;
	struct work_struct serv_task;
	struct mutex avail_q_mutex;	/* protects access to avail_[rx|tx]qs */
	struct mutex sw_mutex;		/* lock for protecting VSI alloc flow */
	struct mutex tc_mutex;		/* lock to protect TC changes */
	struct mutex adev_mutex;	/* lock to protect aux device access */
	struct mutex lag_mutex;		/* protect ice_lag struct in PF */
	u32 msg_enable;
	struct ice_ptp ptp;
	struct gnss_serial *gnss_serial;
	struct gnss_device *gnss_dev;
	u16 num_rdma_msix;		/* Total MSIX vectors for RDMA driver */
	u16 rdma_base_vector;

	/* spinlock to protect the AdminQ wait list */
	spinlock_t aq_wait_lock;
	struct hlist_head aq_wait_list;
	wait_queue_head_t aq_wait_queue;
	bool fw_emp_reset_disabled;

	wait_queue_head_t reset_wait_queue;

	u32 hw_csum_rx_error;
	u32 hw_rx_eipe_error;
	u32 oicr_err_reg;
	struct msi_map oicr_irq;	/* Other interrupt cause MSIX vector */
	struct msi_map ll_ts_irq;	/* LL_TS interrupt MSIX vector */
	u16 max_pf_txqs;	/* Total Tx queues PF wide */
	u16 max_pf_rxqs;	/* Total Rx queues PF wide */
	u16 num_lan_msix;	/* Total MSIX vectors for base driver */
	u16 num_lan_tx;		/* num LAN Tx queues setup */
	u16 num_lan_rx;		/* num LAN Rx queues setup */
	u16 next_vsi;		/* Next free slot in pf->vsi[] - 0-based! */
	u16 num_alloc_vsi;
	u16 corer_count;	/* Core reset count */
	u16 globr_count;	/* Global reset count */
	u16 empr_count;		/* EMP reset count */
	u16 pfr_count;		/* PF reset count */

	u8 wol_ena : 1;		/* software state of WoL */
	u32 wakeup_reason;	/* last wakeup reason */
	struct ice_hw_port_stats stats;
	struct ice_hw_port_stats stats_prev;
	struct ice_hw hw;
	u8 stat_prev_loaded:1; /* has previous stats been loaded */
	u8 rdma_mode;
	u16 dcbx_cap;
	u32 tx_timeout_count;
	unsigned long tx_timeout_last_recovery;
	u32 tx_timeout_recovery_level;
	char int_name[ICE_INT_NAME_STR_LEN];
	char int_name_ll_ts[ICE_INT_NAME_STR_LEN];
	struct auxiliary_device *adev;
	int aux_idx;
	u32 sw_int_count;
	/* count of tc_flower filters specific to channel (aka where filter
	 * action is "hw_tc <tc_num>")
	 */
	u16 num_dmac_chnl_fltrs;
	struct hlist_head tc_flower_fltr_list;

	u64 supported_rxdids;

	__le64 nvm_phy_type_lo; /* NVM PHY type low */
	__le64 nvm_phy_type_hi; /* NVM PHY type high */
	struct ice_link_default_override_tlv link_dflt_override;
	struct ice_lag *lag; /* Link Aggregation information */

	struct ice_eswitch eswitch;
	struct ice_esw_br_port *br_port;

\#define ICE_INVALID_AGG_NODE_ID		0
\#define ICE_PF_AGG_NODE_ID_START	1
\#define ICE_MAX_PF_AGG_NODES		32
	struct ice_agg_node pf_agg_node[ICE_MAX_PF_AGG_NODES];
\#define ICE_VF_AGG_NODE_ID_START	65
\#define ICE_MAX_VF_AGG_NODES		32
	struct ice_agg_node vf_agg_node[ICE_MAX_VF_AGG_NODES];
	struct ice_dplls dplls;
	struct device *hwmon_dev;
};
```