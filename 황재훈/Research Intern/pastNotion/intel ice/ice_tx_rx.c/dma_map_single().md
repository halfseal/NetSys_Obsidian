  

```JavaScript
static inline dma_addr_t dma_map_single_attrs(struct device *dev, void ptr,
size_t size, enum dma_data_direction dir, unsigned long attrs)
{
/ DMA must never operate on areas that might be remapped. */
if (dev_WARN_ONCE(dev, is_vmalloc_addr(ptr),
"rejecting DMA map of vmalloc memory\n"))
return DMA_MAPPING_ERROR;
debug_dma_map_single(dev, ptr, size);
return dma_map_page_attrs(dev, virt_to_page(ptr), offset_in_page(ptr),
size, dir, attrs);
}

static inline dma_addr_t dma_map_single_attrs(struct device *dev, void ptr,
size_t size, enum dma_data_direction dir, unsigned long attrs)
{
/ DMA must never operate on areas that might be remapped. */
if (dev_WARN_ONCE(dev, is_vmalloc_addr(ptr),
"rejecting DMA map of vmalloc memory\n"))
return DMA_MAPPING_ERROR;
debug_dma_map_single(dev, ptr, size);
return dma_map_page_attrs(dev, virt_to_page(ptr), offset_in_page(ptr),
size, dir, attrs);
}
```

- ==`struct device *dev`====: DMA를 실행할 device==
- ==`void *ptr`====: mapping이 될 mem buffer로의 pointer==
- ==`size_t size`====: mapping 될 buffer의 크기==
- ==`unsigned long attrs`====: ???==

  

  

```JavaScript

return dma_map_page_attrs(dev, virt_to_page(ptr), offset_in_page(ptr),
size, dir, attrs);

static inline dma_addr_t dma_map_page_attrs(struct device *dev,
		struct page *page, size_t offset, size_t size,
		enum dma_data_direction dir, unsigned long attrs)
{
	return DMA_MAPPING_ERROR;
}
```

==`**dma_map_page_attrs**`==

: DMA를 위한 mem buffer mapping을 한다

- ==**Parameters**====:==
    - ==`dev`====: DMA를 실행할 device==
    - ==`virt_to_page(ptr)`====: virtual address in ptr을 struct page * 로 바꾼다. 이 ptr이 있는 page 를 의미한다??==
    - ==`offset_in_page(ptr)`====: ??? ptr이 존재하는 page의 위치를 찾기 위한 offset?==
    - ==`size`====: mapping 될 buffer 의 크기==
    - ==`dir`====: DMA이 될 방향 (====`DMA_TO_DEVICE`====,== ==`DMA_FROM_DEVICE`====, etc.).==
    - ==`attrs`====: Additional attributes for the DMA mapping???==
- ==**Return Value**====: device가 버퍼에 접근할 때 사용할 dma addr를 리턴한다==