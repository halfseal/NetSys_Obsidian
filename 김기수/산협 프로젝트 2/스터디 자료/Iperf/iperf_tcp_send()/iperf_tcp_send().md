src/iperf_tcp.c

만약 제로카피면 Nsendfile()로 하고, 아니라면 Nwrite()으로 하게 됨.

![[Untitled 9.png|Untitled 9.png]]

중간 중간에 최대 크기만큼 전송하지 못하는 경우가 있음.

  

---

또한, loopback으로 하였을 때, 최대 blocksize는 (1024 * 1024)Byte였고, 이때 100Gbps를 달성하는 것을 보았음.

![[Untitled 1 4.png|Untitled 1 4.png]]

![[Untitled 2 4.png|Untitled 2 4.png]]

사용한 옵션

블럭 사이즈를 기본으로 하니 약 45Gbps로 감소하게 됨. ( 128* 1024)

zero copy 옵션을 비활성화 하면 매번 달라지게 되나 대부분 감소하는 경향을 보였음. (차이 없음 ~ 30Gbps)

mac 에서 네이티브로 돌렸을 때는 160Gbps까지 올라갔었음.

  

---

![[Untitled 3 2.png|Untitled 3 2.png]]

perf로 iperf 성능측정을 해보았지만, 아직까지 어떻게 해석하는지 몰라서 일단 데이터만 저장. 커널 함수들만 측정해서 그런지 iperf내의 함수 심볼은 보이지 않음. 추가 공부 필요.

  

---

앞으로 해야 할 일 - loopback과 normal 각각 transmit의 메커니즘에서 차이가 나는 것을 확인해 봐야 할 필요가 있음. 왜 100Gbps 이상이 나오는지 해석해야 함.

  

파일에서 읽어오는 것이 문제라고 보았고, 1048576크기의 문자 배열을 선언하여 시도하여 보았으나 중간에 프로그램이 멈추었다. 또한 찍힌 시간도 크게 다르지 않았다.

  

루프백은 L3에서 루프백인터페이스를 통해 다시 자신에게 전송되는 형식이다. 즉, 루프백과 일반적인 동신의 차이점은 일단 2계층에서 잡아먹는 성능 문제와 프로그램 자체적인 오버헤드이다.

  

위의 perf 그래프는 실제 호출 스택과 실행시간이 아니라 정리된 실행시간이다. 각 칸마다 마우스를 올려보면 ~ samples라고 뜨는데, 즉 perf 자체 기능으로는 함수 스택을 시간 순으로 그대로 쫓아가는 것이 아니라 아주 짧은 주기로 샘플링을 하여 얼마나 많이 이 샘플링에 포함되어 있느냐로 %를 계산하는 것이다. 따라서 호출 횟수나 이러한 정보들은 포함되어 있지 않은 것이다. 이 것을 깨닫게 된 것은 uftrace를 사용하기 위해 -pg 옵션을 추가함으로써, 본 perf에서도 해당하는 함수의 이름이 뜨게 된 것인데, 기존에는 [iperf]라고만 떴던 함수의 이름이 구체적으로 Nwrite이라고 떴었고, 위의 그래프는 프로그램 전체에서 측정한 것이므로 Nwrite이 한 번만 호출되었다는 것은 어불성설이다. 따라서 위와 같이 생각해볼 수 있다는 것이다.

  

![[Untitled 4 2.png|Untitled 4 2.png]]

uftrace로 userspace에서 실행되는 함수의 시간을 확인 할 수 있었음. iperf_send_mt()를 계속 실행하게 되는데, 이때 각 함수는 tmp파일을 읽어서 한 번 전송하는 것으로 끝나게 됨. 중간에 iperf_check_throttle()함수의 경우 사전에 옵션으로 설정한 throttle을 넘어서는지 체크하는 함수이지만, 이는 전체 실행시간 중 무시할만 하다고 할 수 있음.(보통 42ns)

  

parallel 옵션을 주었을 때, 몇개이든 상관없이 360~370Gbps 정도의 성능을 보여줌. 단일 flow에서 170정도였던 것과 비교해 보았을 때, 이를 해결하는 것을 우선적으로 하면 될 것 같음.

- 터미널
    
    ```Plain
    user@users  ~/network/iperf   master ±  ./src/iperf3 -c 127.0.0.1 -Z -t 5 -l 1048576 -P 6 -N
    Connecting to host 127.0.0.1, port 5201
    [  5] local 127.0.0.1 port 50050 connected to 127.0.0.1 port 5201
    [  7] local 127.0.0.1 port 50056 connected to 127.0.0.1 port 5201
    [  9] local 127.0.0.1 port 50060 connected to 127.0.0.1 port 5201
    [ 11] local 127.0.0.1 port 50064 connected to 127.0.0.1 port 5201
    [ 13] local 127.0.0.1 port 50074 connected to 127.0.0.1 port 5201
    [ 15] local 127.0.0.1 port 50084 connected to 127.0.0.1 port 5201
    [ ID] Interval           Transfer     Bitrate         Retr  Cwnd
    [  5]   0.00-1.00   sec  8.76 GBytes  75.2 Gbits/sec    5   8.12 MBytes
    [  7]   0.00-1.00   sec  6.01 GBytes  51.6 Gbits/sec    7   8.12 MBytes
    [  9]   0.00-1.00   sec  8.59 GBytes  73.8 Gbits/sec   15   8.12 MBytes
    [ 11]   0.00-1.00   sec  6.67 GBytes  57.2 Gbits/sec   29   4.12 MBytes
    [ 13]   0.00-1.00   sec  6.78 GBytes  58.2 Gbits/sec   28   1.94 MBytes
    [ 15]   0.00-1.00   sec  4.81 GBytes  41.3 Gbits/sec   30   2.25 MBytes
    [SUM]   0.00-1.00   sec  41.6 GBytes   357 Gbits/sec  114
    
    ---
    
    [  5]   1.00-2.00   sec  8.31 GBytes  71.4 Gbits/sec   17   8.12 MBytes
    [  7]   1.00-2.00   sec  6.10 GBytes  52.4 Gbits/sec   15   3.93 MBytes
    [  9]   1.00-2.00   sec  7.31 GBytes  62.8 Gbits/sec   19   8.12 MBytes
    [ 11]   1.00-2.00   sec  9.01 GBytes  77.4 Gbits/sec    7   4.12 MBytes
    [ 13]   1.00-2.00   sec  7.16 GBytes  61.5 Gbits/sec   91   1.62 MBytes
    [ 15]   1.00-2.00   sec  7.76 GBytes  66.7 Gbits/sec   15   2.25 MBytes
    [SUM]   1.00-2.00   sec  45.6 GBytes   392 Gbits/sec  164
    
    ---
    
    [  5]   2.00-3.00   sec  8.83 GBytes  75.8 Gbits/sec   15   8.12 MBytes
    [  7]   2.00-3.00   sec  5.68 GBytes  48.8 Gbits/sec    1   3.93 MBytes
    [  9]   2.00-3.00   sec  5.81 GBytes  49.9 Gbits/sec   95   3.93 MBytes
    [ 11]   2.00-3.00   sec  8.94 GBytes  76.8 Gbits/sec   23   4.12 MBytes
    [ 13]   2.00-3.00   sec  6.53 GBytes  56.1 Gbits/sec   61   2.12 MBytes
    [ 15]   2.00-3.00   sec  6.61 GBytes  56.8 Gbits/sec   66   1.87 MBytes
    [SUM]   2.00-3.00   sec  42.4 GBytes   364 Gbits/sec  261
    
    ---
    
    [  5]   3.00-4.00   sec  8.18 GBytes  70.3 Gbits/sec   23   8.12 MBytes
    [  7]   3.00-4.00   sec  8.63 GBytes  74.1 Gbits/sec   12   3.93 MBytes
    [  9]   3.00-4.00   sec  7.11 GBytes  61.1 Gbits/sec   45   2.06 MBytes
    [ 11]   3.00-4.00   sec  8.25 GBytes  70.9 Gbits/sec   18   4.12 MBytes
    [ 13]   3.00-4.00   sec  7.20 GBytes  61.9 Gbits/sec   11   2.31 MBytes
    [ 15]   3.00-4.00   sec  6.85 GBytes  58.8 Gbits/sec   44   2.75 MBytes
    [SUM]   3.00-4.00   sec  46.2 GBytes   397 Gbits/sec  153
    
    ---
    
    [  5]   4.00-5.00   sec  7.54 GBytes  64.7 Gbits/sec   15   8.12 MBytes
    [  7]   4.00-5.00   sec  7.08 GBytes  60.8 Gbits/sec   17   3.93 MBytes
    [  9]   4.00-5.00   sec  6.71 GBytes  57.6 Gbits/sec  224   1.19 MBytes
    [ 11]   4.00-5.00   sec  8.72 GBytes  74.8 Gbits/sec   23   3.62 MBytes
    [ 13]   4.00-5.00   sec  7.33 GBytes  62.9 Gbits/sec   82   2.56 MBytes
    [ 15]   4.00-5.00   sec  6.82 GBytes  58.5 Gbits/sec  101   1.56 MBytes
    [SUM]   4.00-5.00   sec  44.2 GBytes   379 Gbits/sec  462
    
    ---
    
    [ ID] Interval           Transfer     Bitrate         Retr
    [  5]   0.00-5.00   sec  41.6 GBytes  71.5 Gbits/sec   75             sender
    [  5]   0.00-5.00   sec  41.6 GBytes  71.5 Gbits/sec                  receiver
    [  7]   0.00-5.00   sec  33.5 GBytes  57.5 Gbits/sec   52             sender
    [  7]   0.00-5.00   sec  33.5 GBytes  57.5 Gbits/sec                  receiver
    [  9]   0.00-5.00   sec  35.5 GBytes  61.0 Gbits/sec  398             sender
    [  9]   0.00-5.00   sec  35.5 GBytes  61.0 Gbits/sec                  receiver
    [ 11]   0.00-5.00   sec  41.6 GBytes  71.4 Gbits/sec  100             sender
    [ 11]   0.00-5.00   sec  41.6 GBytes  71.4 Gbits/sec                  receiver
    [ 13]   0.00-5.00   sec  35.0 GBytes  60.1 Gbits/sec  273             sender
    [ 13]   0.00-5.00   sec  35.0 GBytes  60.1 Gbits/sec                  receiver
    [ 15]   0.00-5.00   sec  32.9 GBytes  56.4 Gbits/sec  256             sender
    [ 15]   0.00-5.00   sec  32.9 GBytes  56.4 Gbits/sec                  receiver
    [SUM]   0.00-5.00   sec   220 GBytes   378 Gbits/sec  1154             sender
    [SUM]   0.00-5.00   sec   220 GBytes   378 Gbits/sec                  receiver
    
    iperf Done.
    ```
    

[https://seongsee.tistory.com/10](https://seongsee.tistory.com/10)