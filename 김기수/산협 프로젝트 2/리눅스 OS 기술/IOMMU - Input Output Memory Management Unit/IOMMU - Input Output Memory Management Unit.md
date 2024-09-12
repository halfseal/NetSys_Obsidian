- IOMMU가 제공하는 기능
    1. device가 접근할 수 없는 physical address space를 접근할 수 있도록 address space를 다시 매핑하는 기능 ⇒ ex) 32-Bit DMA를 64-Bit physical address로. 이말은 device가 32비트 메모리 주소 체계를 가지고 있어 DMA 또한 32비트 체계만을 가지고 있지만, IOMMU를 통해 64비트 메모리 주소로 매핑이 가능하므로, 더 넓은 대역을 쓸 수 있다. 즉, 42억까지의 주소가 아닌 100억 번째의 physical memory address를 접근할 수 있도록 할 수 있다.
    2. page level에서 scatter-gather에 대한 것을 지원함으로써, device가 자체적으로 하지 않아도 된다. 우선 scatter-gather란 다양한 곳에서 쓰이는 용어인데, 비슷한 기능이 여러번 호출되는 경우 이를 묶어서 한번에 처리하여 한번에 가져오거나 한번에 묶어서 보내는 어감을 가지고 있다. IOMMU의 scatter-gather는 비연속적인 physical address를 연속적인 virtual address로 매핑할 수 있으므로, device가 이를 다루기 위해 들여야 할 품이 줄어들 것이다.
    3. 함부로 다른 physical 메모리를 접근하는것을 막아 시스템을 보호를 제공할 수 있고, 매핑되지 않은 주소영역을 접근하려고 한다면 faulting을 할 것이다.

![[Untitled 7.png|Untitled 7.png]]

교수님께서 말씀하신 writel() 함수의 경우 ice_txrx.c 안에 ice_tx_map() 함수 내부에서 호출되고 있었다. 여기서 좀 더 뻗어나가고 싶지만 아직 Rx 측을 끝내지 못하여 확인하지 못하였다.

```C
\#ifndef writel
\#define writel writel
static inline void writel(u32 value, volatile void __iomem *addr)
{
	log_write_mmio(value, 32, addr, _THIS_IP_, _RET_IP_);
	__io_bw();
	__raw_writel((u32 __force)__cpu_to_le32(value), addr);
	__io_aw();
	log_post_write_mmio(value, 32, addr, _THIS_IP_, _RET_IP_);
}
\#endif
```