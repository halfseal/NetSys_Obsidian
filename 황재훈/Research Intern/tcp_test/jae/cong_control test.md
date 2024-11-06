__CUBIC mtu 1500__
cubic (1 connection) mtu 1500
![[Pasted image 20241106211743.png]]

cubic (10 connection) mtu 1500
![[Pasted image 20241106212648.png]]

cubic (30 connection) mtu 1500
![[Pasted image 20241106212221.png]]

__CUBIC mtu 100__
cubic (1 connection) mtu 100
iperf3 -c 40.0.0.3 -t 30 > cubic_1con_100mtu
![[Pasted image 20241106214810.png]]
cubic (10 connection) mtu 100
iperf3 -c 40.0.0.3 -t 30 -P 10 > cubic_10con_100mtu
![[Pasted image 20241106215005.png]]

cubic (30 connection) mtu 100
iperf3 -c 40.0.0.3 -t 30 -P 30 > cubic_30con_100mtu
![[Pasted image 20241106215036.png]]

__BBR mtu 100__
bbr (1 connection) mtu 100
iperf3 -c 40.0.0.3 -t 30 > bbr_1con_100mtu 
![[Pasted image 20241106215820.png]]

bbr (10 connection) mtu 100
iperf3 -c 40.0.0.3 -t 30 -P 10 > bbr_10con_100mtu  
![[Pasted image 20241106215954.png]]

bbr (30 connection) mtu 100
iperf3 -c 40.0.0.3 -t 30 -P 30 > bbr_30con_100mtu   
![[Pasted image 20241106220012.png]]

__BBR mtu 1500__
bbr (1 connection) mtu 1500
iperf3 -c 40.0.0.3 -t 30 > bbr_1con_1500mtu
![[Pasted image 20241106220820.png]]

bbr (10 connection) mtu 1500
iperf3 -c 40.0.0.3 -t 30 -P 10 > bbr_10con_1500mtu
![[Pasted image 20241106220852.png]]

bbr (30 connection) mtu 1500
iperf3 -c 40.0.0.3 -t 30 -P 30 > bbr_30con_1500mtu
![[Pasted image 20241106220917.png]]