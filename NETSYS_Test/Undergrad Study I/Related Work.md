---
creator: 김명수
type: Paper Review
created: 2022-11-14
---
# Related work

- Dynamic thread block launch

[https://ieeexplore.ieee.org/document/7284092](https://ieeexplore.ieee.org/document/7284092)

- application에서 data를 잘 분배하여mapping하는 것이 좋은 접근 방법이지만 현재(2015)는 device-side에서의 kernel launch의 cost로 인해 효율적이지 못하다.
    
- 논문 저자들은 기존의 비효율적인 새로운 kernel launch대신 cost가 더 적은 thread block launch로 접근
    
- 기존 DP의 특징
    
    - High kernel density
        
        - 너무 많은 kernel이 launch되어서 overhead증가
    - Low compute intensity
        
        - 새로운 kernel이 매우 작은 thread들(평균40)만 가지고 있어서 효율적이지 못하다.
    - workload similarity
        
    - Low concurrency & scheduling efficiency
        
        - 커널이 많이 생성되어도 동시에 실행될 수 있는 커널의 수가 HWQ로 제한되어서 효율적이지 못함
        - 13 SM에서 1개 SM당 64개 warp 실행가능 할 경우
        - 예를 들어 2개의 warp를 가지고 있는 32개의 kernel ⇒ 동시에 실행되는 warp가 64개 ⇒ 극단적으로 1개의 SM만 사용할 수도 있음 ??
        - low utilization
- device kernel의 thread가 여러 thread block을 생성할 수 있음
    

## **🧐**

- HWQ로 인해서 max kernel exe가 제한되고 무분별한 kernel launch가 overhead라면 현재 실행중인 kernel의 수를 child kernel launch의 metric으로 사용하면 어떨까??
    - max concurent kernel로 실행되는 순간에서 얻는 이득과 kernel launch를 줄임으로써 생기는 이득을 비교해야함