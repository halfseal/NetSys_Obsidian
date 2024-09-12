---
creator: 김명수
type: Hands-On
created: 2022-05-10
---
<<[[GSO, TSO enable|previous]] | [[GSO, TSO cntd. 2|next]]>>

-W2 : GSO, TSO의 정확한 차이? + GSO가 결국 NIC이 아니라 kernel단에서 실행된다면 GSO로 얻을 수 있는 이득이 무엇인가?

-W2 : core reading line by line
![[Untitled(25).png]]


[https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/performance_tuning_guide/network-nic-offloads](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/performance_tuning_guide/network-nic-offloads)

- GSO의 목적은 segmentation수행을 최대한 늦추는 것.
- GSO자체는 모든 NIC이 TSO를 지원할 수 없다는 문제를 해결하기 위해 존재하는 것은 맞지만 GSO가 TSO를 대체하거나 그 두 가지가 완전히 대립되는 개념이 아님
- GSO에 generic이란 이름이 붙은 이유가 해당 NIC에서 TSO가능여부와 상관없이 gso_size()크기의 여러 data로 segment를 수행하기 때문이라 생각됨.

## Case 1. GSO, TSO unable
![[Untitled(26).png]]


## Case2. GSO, TSO enable
![[Untitled(27).png]]


- gso_size = 64KB

## Case3. GSO enable, TSO unable
![[Untitled(28).png]]


- case3그림에서 2가지 가능성이 존재함
- [ ] -1. TSO가 불가능한 host에서는 애초에 gso_size 1500B로 설정이 되어 있어서 NIC에 default MTU(1500B) size로 전달
- [ ] -2. 설정된 gso_size로 segment를 수행한 뒤 NIC이 TSO가 불가능 하다는 사실을 알았을 때 다시 MTU(1500B)로 segment를 수행해서 NIC에게 전달
- 성능적인 면을 고려했을 때 scenario1이 더 적합해 보이지만 정확한 매커니즘은 코드를 통해 확인 해봐야함.
- 만약 scenario2의 방식으로gso가 동작한다면 TSO 지원이 불가능한 host에서는 오히려 case1보다 성능이 안 좋아질 것으로 예상(모든 segment작업이 kernel에서 이루어질 뿐만 아니라 segment횟수도 더 많다.)

code...

[gsotso.pptx](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/93e9b829-6df2-4cdb-bf4f-eb9d0ef7e88e/gsotso.pptx)