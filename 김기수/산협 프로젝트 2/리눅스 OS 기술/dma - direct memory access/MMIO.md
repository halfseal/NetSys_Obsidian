memory-mapped input/output

host가 device의 레지스터와 통신하기 위해 사용되는 메모리 주소 공간.

메모리 read / write와 같은 방식으로 device와 상호작용할 수 있게 됨.

physical address와 device의 register간의 매핑을 하여 명령어 수준에서 host가 device의 register에 접근 가능. device의 driver가 이를 관리하게 됨.