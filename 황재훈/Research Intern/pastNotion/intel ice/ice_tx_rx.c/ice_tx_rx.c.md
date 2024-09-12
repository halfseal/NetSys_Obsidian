[[ice_tx_map()]]

네트워크에서 transmit descriptor를 생성하는 함수

`struct ice_tx_desc`

- hw에서 사용되는 자료구조, packet process transmisson에 사용된다
- 메타 데이터를 보유한다 (pointer to buffer, length, control field?)
- descriptor 는 메모리의 버퍼와 일대일 대응된다

  

[[dma_map_single()]]

  

ice_txrx.c 내부의 함수

1. 변수 초기값 설정
2. ==VLAN 관리?? (virtual network, physical netw → multiple virtaul netw)==
3. skb data mapping 하기 (DMA를 위해)
4. 반복문
    - DMA error 찾기
    - descriptor 업데이트, data 처리
    - 다음 fragment 준비
5. final descriptor 설정하기
6. mem write을 완료하고 hw에 알리기?
7. DMA error 고치기 (==mapping 초기화, ring state update==)?

  

parameter

- tx_ring : buffer를 전송할 ring
    - ice_tx_ring
- first :
- off

variable

- ice_tx_desc
- ice_tx_buf
- sk_buff
- skb_frag_t
- dma_addr_t

```Plain
line 1661
tx_desc = ICE_TX_DESC(tx_ring, i);
```

사용할 descriptor에 해당되는 tx_ring을 할당한다.

  

```JavaScript
	line 1668
	dma = dma_map_single(tx_ring->dev, skb->data, size, DMA_TO_DEVICE)
```