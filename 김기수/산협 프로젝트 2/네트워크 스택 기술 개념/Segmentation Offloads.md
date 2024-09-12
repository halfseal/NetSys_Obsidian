[https://docs.kernel.org/networking/segmentation-offloads.html](https://docs.kernel.org/networking/segmentation-offloads.html)

  

### TSO - TCP Segmentation Offload

---

TCP Segmentation은 device가 하나의 커다란 frame을 skb_shinfo()→gso_size를 참고하여 여러 개의 frame들로 쪼개는 것을 말한다.

SKB_GSO_TCPV4, SKB_GSO_TCPV6 등의 비트가 켜져 있을 때 요청되며, gso_type과 gso_size가 specific 되어야 한다.

새로운 segment들을 정의해야 하므로 CheckSum offload 또한 지원해야 한다.

또한, skbuff에서 3계층과 4계층 헤더의 오프셋 정보 또한 주어져야 device의 driver가 이를 바탕으로 처리할 수 있을 것이다.

IP 헤더의 Identification field에서, 해당 부분을 자연스럽게 증가시킬지, 고정시킬지도 옵션을 통해 정할 수 있다.

  

### GSO - Generic Segmentation Offload

---

GSO는 순수한 소프트웨어 오프로드인데, device driver가 수행할 수 없는 경우들을 다루게 된다.

skb_shinfo()→gso_size를 통해 제공 된 MSS 크기를 바탕으로 주어진 sk_buff를 여러 개의 sk_buff로 분할시킨다.

다른 어떠한 Hardware Segmentation Offload든지 활성화 하기전에, GSO에 해당하는 software offload가 필요하다.

그러지 않는다면, frame이 Device간에 재라우팅 되었을 때, 전송되지 못할 가능성이 있기 때문이다.

  

### GRO - Generic Receive Offload

---

GRO는 GSO와 상호보완적이다. 이상적인 상황에서 GRO로 조립된 프레임이면 GSO로 분할 가능하고, GSO로 분할된 프레임들은 GRO로 조립될 수 있어야 한다.

한 가지 예외 상황은 IPv4에서 DF bit이 설정된 IP header의 Identification field이다.

만약 IPv4의 ID 필드가 연속적으로 증가하지 않는다면, 이는 GSO로 분할되어 전송되었고, GRO로 조립될 프레임이라는 것임을 알릴 것이다.