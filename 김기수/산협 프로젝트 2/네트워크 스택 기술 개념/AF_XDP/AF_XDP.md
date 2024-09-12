[https://docs.kernel.org/networking/af_xdp.html](https://docs.kernel.org/networking/af_xdp.html)

af_xdp는 초고속 패킷 처리를 위해 최적화된 주소들의 모임이다.

[https://docs.cilium.io/en/latest/bpf/](https://docs.cilium.io/en/latest/bpf/)

[https://www.intel.co.kr/content/www/kr/ko/support/articles/000098553/ethernet-products.html](https://www.intel.co.kr/content/www/kr/ko/support/articles/000098553/ethernet-products.html)

---

XDP는 실행중인 커널에 사용자가 코드를 주입함으로써, 네트워크 스택에 들어가기 전에 실행함으로써 CPU 사용율을 줄일 수 있는 오픈소스 기술이다.

AF_XDP는 그와중에 socket optimized 된 것이다. 따라서 socket API를 활용하여 대역폭을 온전히 활용하여 처리할 수 있다.

위의 두 기술은 BPF / eBPF 위에서 작동중이다.

BPF는 리눅스 커널에서 레지스터 기반의 virtual machine-like construct로, 매우 유연하고 효과적이여서, bytecode를 다양한 hook points에서 안전한 방법으로 실행가능하다.

[https://www.minzkn.com/moniwiki/wiki.php/XDP](https://www.minzkn.com/moniwiki/wiki.php/XDP)

![[Untitled 8.png|Untitled 8.png]]

---

[https://www.slideshare.net/suselab/ixgbe-internals#3](https://www.slideshare.net/suselab/ixgbe-internals#3)