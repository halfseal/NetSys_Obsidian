---
creator: 홍권
type: Hands-On
created: 2022-07-26
---
## The experiment I was going to reproduce:

Measure how (packet drop) performance varies between system using DPDK, XDP, and Linux, with different # of cores.

## My Plan was:

1. Start with benchmarking the system(w/o any changes) with TRex traffic generator.
2. Replace the Linux network stack with DPDK or XDP, then run the same benchmark.
3. Do the same benchmarks with different # of cores.
4. Done!

## The paper’s benchmark setup

1. Hardware
    1. Intel Xeon E5-1650 v4 CPU running at 3.60GHz, support DDIO
    2. two Mellanox ConnectX-5 Ex VPI dual-port 100G NIC
2. Software
    1. Linux kernel version: 4.18
    2. TRex packet generator to produce the test traffic

[https://github.com/tohojo/xdp-paper](https://github.com/tohojo/xdp-paper)

---

## Question #1: How the traffic generator works?

The paper’s benchmark setup says it used **two** NIC.

### My assumption

1. Both NICs connected to a single machine
2. Use NIC #1 to send generated packets(to NIC #2)
3. and NIC #2 to receive the packets(from the NIC #1)

→ loopback?

Is it correct…?

So I used

TRex’s default configuration as follows:

```bash
- version: 2
  interfaces: ['65:00.0', '65:00.1']
  port_info:
      **- ip: 1.1.1.1
        default_gw: 2.2.2.2
      - ip: 2.2.2.2
        default_gw: 1.1.1.1**
```

For interface 1, assuming loopback to its dual interface 2. Putting IP 1.1.1.1, default gw 2.2.2.2

## Question #2: What kind of traffic should I generate…?

What kind of traffic can I generate?

How the result varies with the different kind of traffic?

## Question #3: How similar my setup should be?

There aren’t many options for me.

First option:

1. **Hardware**: out of interest at the begining… Not affordable :)

→ CPU with DDIO is enough maybe?

→ Need 100G NIC but only for reproducing, 10G would be enough

1. **Software**

→ Linux kernel version was specified, but not the distribution

→ Found it is not that hard to change Linux kernel but… should I match the kernel version?

→ I guess the distro is not important so just used Ubuntu 20.04 LTS

→ Tried to use TRex traffic generator

## Question #4: NICs not working properly…

lspci | grep Ethernet

```bash
00:1f.6 Ethernet controller: Intel Corporation Ethernet Connection (2) I219-V
**65:00.0 Ethernet controller: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection (rev 01)
65:00.1 Ethernet controller: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection (rev 01)**
```

sudo lshw -class network -short

```bash
H/W path         Device          Class          Description
===========================================================
/0/100/1f.6      eno1            network        Ethernet Connection (2) I219-V
**/0/101/0                         network        82599ES 10-Gigabit SFI/SFP+ Network Connection
/0/101/0.1                       network        82599ES 10-Gigabit SFI/SFP+ Network Connection**
```

ip link show

```bash
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
link/ether 04:42:1a:0c:ea:94 brd ff:ff:ff:ff:ff:ff
altname enp0s31f6
5: teredo: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1280 qdisc fq_codel state UNKNOWN mode DEFAULT group default qlen 500
link/none
6: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
```

→ ip/ifconfig does not show the 10G NICs

So I tried to (compile and)install latest ixgbe driver.

[https://www.xmodulo.com/download-install-ixgbe-driver-ubuntu-debian.html](https://www.xmodulo.com/download-install-ixgbe-driver-ubuntu-debian.html)

It was successful but still not working as follows:

```bash
gwonhong@b1:/opt/trex/v2.99$ sudo ./t-rex-64 -c 4 -i
Starting Scapy server..... Scapy server is started
The ports are bound/configured.
Starting  TRex v2.99 please wait  ...
 set driver name net_ixgbe
 driver capability  : TCP_UDP_OFFLOAD  TSO  LRO
 set dpdk queues mode to DROP_QUE_FILTER
 Number of ports found: 2
zmq publisher at: tcp://*:4500
 wait 2 sec ..
 WARNING : there is no link on one of the ports, interactive mode can continue
port : 0
------------
link         :  link : Link Up - speed 10000 Mbps - full-duplex
promiscuous  : 0
port : 1
------------
link         :  **Link Down**
promiscuous  : 0
 number of ports         : 2
 max cores for 2 ports   : 4
```

Actually, only one of the port works. Maybe the port configuration is wrong…?

```bash
gwonhong@b1:/opt/trex/v2.99$ ./trex-console

Using 'python3' as Python interpeter

Connecting to RPC server on localhost:4501                   [SUCCESS]

Connecting to publisher server on localhost:4500             [SUCCESS]

Acquiring ports [0, 1]:                                      [SUCCESS]

***** Warning - Port 0 destination is unresolved ***
*** Warning - Port 1 destination is unresolved *****

Server Info:

Server version:   v2.99 @ STL
Server mode:      Stateless
Server CPU:       4 x Intel(R) Core(TM) i9-10900X CPU @ 3.70GHz
Ports count:      2 x 10Gbps @ 82599ES 10-Gigabit SFI/SFP+ Network Connection

-=TRex Console v3.0=-

trex>start -f /home/gwonhong/xdp-paper/benchmarks/udp_for_benchmarks.py -t packet_len=64,stream_count=8 --port 0 1m
pps

**start - Port 0 dest MAC is invalid and there are streams without explicit dest MAC.**
```