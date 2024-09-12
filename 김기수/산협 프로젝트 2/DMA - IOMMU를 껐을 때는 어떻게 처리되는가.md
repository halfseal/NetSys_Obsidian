iommu를 끄게 된다면 device가 직접 physical address에 접근하게 된다.

따라서 IOMMU를 처리하는 오버헤드가 줄어들어 성능이 향상될 수 있다.

그러나 메모리를 보호할 수 없으며 32bit device가 더 큰 메모리 공간을 사용할 수 있는 가능성을 없앤다. → 메모리 주소 공간의 효율성이 저하된다.

또한 device간의 격리가 어려워진다. → 서로의 메모리 주소에 접근할 수 있게 된다.

  

IOMMU를 끄더라도 dma는 이와는 별개로 작동하는데, virtual address가 가르키는 physical address로 접근을 해야하기 때문이다. 즉, 특정 context에서 CPU가 새로운 메모리를 할당하게 되고, 이는 virtual address가 반환되게 된다. 이러한 virtual address가 가르키는 physical address를 device가 가지게 하는 것이 IOMMU를 끄는 것이고, 만약 켜져있다면, IOMMU를 통해 변환된 주소가 해당 physical address가 되도록 bus address를 만들어 이를 반환하게 될 것이다.

결국 둘은 별개가 될 것이고, core에서 사용할 virtual address를 바탕으로 해당하는 physical address 혹은 bus addresss를 device가 알고, 사용할 수 있도록 하는 기술이 DMA이다.

  

디바이스마다 버스