---
creator: 황준용
type: Hands-On
created: 2022-08-16
---

<<[[DPDK vs XDP|previous]]>>

## DPDK vs XDP

Server Side:

```bash
brian11hwang@jooyoung iperf -s
------------------------------------------------------------
Server listening on TCP port 5001
TCP window size:  128 KByte (default)
------------------------------------------------------------
[  4] local 10.0.1.4 port 5001 connected with 10.0.1.3 port 54558
[ ID] Interval       Transfer     Bandwidth
[  4]  0.0-10.0 sec  11.0 GBytes  9.40 Gbits/sec
```

Client Side:

```bash
brian11hwang@b1 sudo iperf -c 10.0.1.4
------------------------------------------------------------
Client connecting to 10.0.1.4, TCP port 5001
TCP window size: 2.52 MByte (default)
------------------------------------------------------------
[  3] local 10.0.1.3 port 54558 connected with 10.0.1.4 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-10.0 sec  11.0 GBytes  9.42 Gbits/sec
```

_using UDP_

It is important for the Server side to also listen in UDP Mode!

Server Side:

```bash
brian11hwang@b1 iperf -s -u
------------------------------------------------------------
Server listening on UDP port 5001
Receiving 1470 byte datagrams
UDP buffer size:  208 KByte (default)
------------------------------------------------------------
[  3] local 10.0.1.3 port 5001 connected with 10.0.1.4 port 50905
[ ID] Interval       Transfer     Bandwidth        Jitter   Lost/Total Datagrams
[  3]  0.0-10.0 sec  1.25 MBytes  1.05 Mbits/sec   0.006 ms    0/  892 (0%)
[  4] local 10.0.1.3 port 5001 connected with 10.0.1.4 port 57524
[  4]  0.0-10.0 sec  1.25 MBytes  1.05 Mbits/sec   0.003 ms    0/  892 (0%)
```

Client Side:

```bash
brian11hwang@jooyoung sudo iperf -c 10.0.1.3 -B 10.0.1.4 -u
------------------------------------------------------------
Client connecting to 10.0.1.3, UDP port 5001
Binding to local address 10.0.1.4
Sending 1470 byte datagrams, IPG target: 11215.21 us (kalman adjust)
UDP buffer size:  208 KByte (default)
------------------------------------------------------------
[  3] local 10.0.1.4 port 57524 connected with 10.0.1.3 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-10.0 sec  1.25 MBytes  1.05 Mbits/sec
[  3] Sent 892 datagrams
[  3] Server Report:
[  3]  0.0-10.0 sec  1.25 MBytes  1.05 Mbits/sec   0.002 ms    0/  892 (0%)
```

## Firewall

### Simple IPTables experiment I: ICMP BLock

Ping Disable

ICMP ping before

![https://velog.velcdn.com/images/brian11hwang/post/da930ac5-a220-4281-82f0-3138009e6e1f/image.png](https://velog.velcdn.com/images/brian11hwang/post/da930ac5-a220-4281-82f0-3138009e6e1f/image.png)

```
IPTABLES -N ICMP
IPTABLES -A INPUT -p icmp -j ICMP
IPTALBLES -A ICMP -p icmp -icmp-type 8 -j DROP
```

Add a policy to iptables that does not respond to ICMP messages.

- IPTABLES -N ICMP: Adds a new chain called ICMP to the table.
- IPTABLES-A INPUT-picmp-j ICMP: Adds a policy that forwards ICMP messages to the ICMP chain when they are received.
- IPTALBLES-A ICMP-picmp-icmp-type 8-j DROP: Type 8 of ICMP messages (icmp echo request)Block messages.

When ICMP File comes(INPUT), redirected to 'ICMP'

![https://velog.velcdn.com/images/brian11hwang/post/2e9150c5-1ab8-4d62-b86b-7453f8b24740/image.png](https://velog.velcdn.com/images/brian11hwang/post/2e9150c5-1ab8-4d62-b86b-7453f8b24740/image.png)

With ICMP type 8 (icmp echo request), DROP Package

![https://velog.velcdn.com/images/brian11hwang/post/09e66a1b-9a40-4bed-ae9f-e1d3bc25e816/image.png](https://velog.velcdn.com/images/brian11hwang/post/09e66a1b-9a40-4bed-ae9f-e1d3bc25e816/image.png)

We can see that no response to Ping happened.

![https://velog.velcdn.com/images/brian11hwang/post/65e6cc84-57ff-473d-a233-ad13f50a7269/image.png](https://velog.velcdn.com/images/brian11hwang/post/65e6cc84-57ff-473d-a233-ad13f50a7269/image.png)

### Simple IPTables experiment II : UDP Flood Block

### Before:

1. Iperf Log

```
brian11hwang@jooyoung iperf -c 10.0.1.3 -B 10.0.1.4 -u -b 5G
------------------------------------------------------------
Client connecting to 10.0.1.3, UDP port 5001
Binding to local address 10.0.1.4
Sending 1470 byte datagrams, IPG target: 2.19 us (kalman adjust)
UDP buffer size:  208 KByte (default)
------------------------------------------------------------
[  3] local 10.0.1.4 port 52229 connected with 10.0.1.3 port 80
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-10.0 sec  2.86 GBytes  2.46 Gbits/sec
[  3] Sent 2087706 datagrams
[  3] Server Report:
[  3]  0.0-10.0 sec  2.80 GBytes  2.41 Gbits/sec   0.006 ms 41410/2087706 (2%)
```

1. Tshark Log

```
5584414 178.703669944     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
5584415 178.703702400     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
5584416 178.703702416     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
5584417 178.703702431     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
5584418 178.703702445     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
5584419 178.703702459     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
5584420 178.703702473     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
5584421 178.703702488     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
5584422 178.703702502     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
5584423 178.703736543     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
5584424 178.703736557     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
5584425 178.703736572     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
5584426 178.703736587     10.0.1.4 → 10.0.1.3     UDP 1512 58225 → 5001 Len=1470
5584427 178.721185099     10.0.1.3 → 10.0.1.4     UDP 1512 5001 → 58225 Len=1470
```

Iperf Server Periodically sends ACK Although UDP Connection for status check.

### Iptables Setting:

```
IPTABLES -N UDP
IPTABLES -A INPUT -p udp -j UDP
sudo iptables -I UDP -p udp -d 10.0.1.3 --dport 5001 -j DROP
IPTABLES -A UDP -j LOG --log-prefix "UDP FLOOD"
```

(Iperf uses 5001 Port)

### After:

```
brian11hwang@jooyoung iperf -c 10.0.1.3 -B 10.0.1.4 -u -b 5G
------------------------------------------------------------
Client connecting to 10.0.1.3, UDP port 5001
Binding to local address 10.0.1.4
Sending 1470 byte datagrams, IPG target: 2.19 us (kalman adjust)
UDP buffer size:  208 KByte (default)
------------------------------------------------------------
[  3] local 10.0.1.4 port 56652 connected with 10.0.1.3 port 5001
[  3] WARNING: did not receive ack of last datagram after 10 tries.
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-10.0 sec  6.25 GBytes  5.37 Gbits/sec
[  3] Sent 4565229 datagrams
```

However, after Cannot Recieve Ack because Iptables drop all packet. Thus Iperf does not know the need to send ACK.