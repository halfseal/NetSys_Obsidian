
#### 참고한 곳
---
https://www.kernel.org/doc/html/v5.1/networking/bpf_flow_dissector.html


#### 본문
---
##### overview
---
Flow dissector는 패킷의 메타데이터를 파싱하는 루틴이다.
이러한 Flow dissector는 다양한 네트워크 서브 시스템에서 사용하는데, RFS, flow hash 등등이 있다.

BPF flow dissector는 BPF verifier의 이득 모두 얻기 위해서 BPF context에서 실행되는 C-기반의 flow dissector를 재작성하는 것을 시도하는 것이다.

