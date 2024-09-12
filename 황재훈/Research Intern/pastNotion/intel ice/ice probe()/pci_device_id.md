```JavaScript
struct pci_device_id {
	__u32 vendor, device;		/* Vendor and device ID or PCI_ANY_ID*/
	__u32 subvendor, subdevice;	/* Subsystem ID's or PCI_ANY_ID */
	__u32 class, class_mask;	/* (class,subclass,prog-if) triplet */
	kernel_ulong_t driver_data;	/* Data private to the driver */
	__u32 override_only;
};
```