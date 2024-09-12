---
creator: 홍권
type: Hands-On
created: 2022-08-09
---
# Troubleshooting #1
### Problem

Can’t being between two 10G NICs

### First try

gave IP address of 10.0.0.x to two 10G NICs

but still can’t ping each other.

checked arp table, and found no MAC address at 10G NICs.

manually entered each others’ MAC address.

then, ping from jooyoung → b1 is possible, but not the opposite.

### Troubleshooting

monitored packets of 10G NIC at jooyoung server

while ping from b1 to jooyoung server

ping request from b1(O)

ping response to b1(X)

by checking another port, found the ‘ping response to b1’!

since two port using the same subnet 10.0.0.X, routing table had two entries for 10.0.0.0, and the first port had higher priority, so ping response to 10.0.0.3(b1 server) was sent to another port.
![[Untitled(121).png]]


```bash
gwonhong@jooyoung:~$ route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         gate179.skku.ac 0.0.0.0         UG    0      0        0 eno1
**10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 enp3s0f0
10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 enp3s0f1**
115.145.179.0   0.0.0.0         255.255.255.0   U     0      0        0 eno1
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0
```

*enp3s0**f1** is the port we want to send/receive packets

So we changed **f0** to 10.0.1.X then became able to ping each other!

But f0 port was connected to another server → better not to change anything

So tried to change another port(f1) and b1 server’s ip to 10.0.1.X instead of f0.

But then it failed.

b1 server kept sending packets to 10.10.1.6 instead of 10.0.0.X.

Searched for the information of 10.10.1.6…

and found it was **Bogon IP Address**

**What is a Bogon IP Address?**

Some IP addresses and IP ranges are reserved for special use,

…such as for local or private networks, and should not appear on the public internet. These reserved ranges, along with other IP ranges that haven’t yet been allocated and therefore also shouldn’t appear on the public internet are sometimes known as bogons.

**Private Address Space**

The Internet Assigned Numbers Authority (IANA) has reserved the following three blocks of the IP address space for private internets:

```
    **10.0.0.0        -   10.255.255.255  (10/8 prefix)**
    172.16.0.0      -   172.31.255.255  (172.16/12 prefix)
    192.168.0.0     -   192.168.255.255 (192.168/16 prefix)
```

Actually, the routing table was not updated for 10.0.0.X, so it was being sent to the outside, but since it’s within private address space, it was just dropped at somewhere in the network(I guess).

```bash
gwonhong@b1:~$ route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         gate179.skku.ac 0.0.0.0         UG    100    0        0 eno1
115.145.179.0   0.0.0.0         255.255.255.0   U     100    0        0 eno1
link-local      0.0.0.0         255.255.0.0     U     1000   0        0 eno1
#no entry for 10.0.0.0!
```

### Conclusion

changed jooyoung’s port(connected with b1) to **10.0.1**.4/24,

and b1’s port(connected with jooyoung) to **10.0.1**.3/24,

reverted jooyoung’s another port to original IP(10.0.0.3/24)

then everything works fine!



# Troubleshooting #2
### Problem

Jooyoung server unreachable with ssh, after enabling UFW(Uncomplicated FireWall, Ubuntu’s built-in firewall) and reboot.

### Plan

1. Need to check if the server is successfully booting or not
2. If it’s booting well, then need to check NICs’ status
3. If NICs are okay, need to check if ssh port(22) is blocked by ufw or whatever

**#1 check if server is booting well**

→ connected monitor and keyboard to check what was going on

❌ it freezes at loading screen…!
![[Untitled(122).png]]


🔍 disabled gpu driver at boot(with nomodeset option)

✅ able to see the tty with login prompt

**#2 check NICs status**

→ logged in with my account, checked NIC status with “ip a” command

❌ ip addresses not assigned to every NICs

🔍 checked netplan config file(/etc/netplan/….yaml)

❓ looked OK so tried “netplan apply” to apply the configuration, then got following error

```bash
/etc/netplan/00-installer-config.yaml:11:1: Invalid YAML: tabs are not allowed for indent:
        enp3s0f0:
^
```

✅ replaced tabs to 8 spaces, then everything got fine

**#3 added ufw firewall exception rule for ssh port(22)**

```bash
sudo ufw allow ssh
```

Then finally able to ssh to jooyoung server


# Simple Benchmark Using iperf
## #1 default settings(TCP, MSS=1500 bytes)

### **jooyoung → b1**

**client side(jooyoung)**

```bash
gwonhong@jooyoung:~$ iperf -c 115.145.179.174
------------------------------------------------------------
Client connecting to 115.145.179.174, TCP port 5001
TCP window size:  620 KByte (default)
------------------------------------------------------------
[  3] local 115.145.179.191 port 37418 connected with 115.145.179.174 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-10.0 sec  1.09 GBytes   936 Mbits/sec
```

**server side(b1)**

```bash
gwonhong@b1:~$ iperf -s
------------------------------------------------------------
Server listening on TCP port 5001
TCP window size:  128 KByte (default)
------------------------------------------------------------
[  4] local 115.145.179.174 port 5001 connected with 115.145.179.191 port 37418
[ ID] Interval       Transfer     Bandwidth
[  4]  0.0-10.0 sec  1.09 GBytes   934 Mbits/sec
```

since they are 10G NICs, the utilization is calculated as about 10%.

### **b1 → jooyoung**

**client side(b1)**

```bash
gwonhong@b1:~$ iperf -c 115.145.179.191
------------------------------------------------------------
Client connecting to 115.145.179.191, TCP port 5001
TCP window size:  527 KByte (default)
------------------------------------------------------------
[  3] local 115.145.179.174 port 57886 connected with 115.145.179.191 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-10.0 sec   862 MBytes   723 Mbits/sec
```

**server side(jooyoung)**

```bash
gwonhong@jooyoung:~$ iperf -s
------------------------------------------------------------
Server listening on TCP port 5001
TCP window size:  128 KByte (default)
------------------------------------------------------------
[  4] local 115.145.179.191 port 5001 connected with 115.145.179.174 port 57886
[ ID] Interval       Transfer     Bandwidth
[  4]  0.0-10.0 sec   862 MBytes   722 Mbits/sec
```

### ❓Questions

1. Why the bandwidths in each other direction are different significantly?
2. Why the bandwidth captured at **client side** is slightly **larger than** bandwidth captured at **server side**? Got similar results with the repeated benchmarks, and in opposite direction.

## #2 MSS=64 bytes

### **jooyoung → b1**

**client side(jooyoung)**

```bash
gwonhong@jooyoung:~$ iperf -c 115.145.179.174 -M 64 -m
WARNING: **attempt to set TCP maxmimum segment size to 64 failed.**
Setting the MSS may not be implemented on this OS.
------------------------------------------------------------
Client connecting to 115.145.179.174, TCP port 5001
TCP window size:  442 KByte (default)
------------------------------------------------------------
[  3] local 115.145.179.191 port 37454 connected with 115.145.179.174 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-10.0 sec  1.09 GBytes   936 Mbits/sec
[  3] MSS size 1448 bytes (MTU 1500 bytes, ethernet)
```

→ setting MSS to 64 bytes has failed.

So tried with MSS=1000 bytes

```bash
gwonhong@jooyoung:~$ iperf -c 115.145.179.174 -M 1000 -m
WARNING: attempt to set TCP maximum segment size to 1000, **but got 536**
------------------------------------------------------------
Client connecting to 115.145.179.174, TCP port 5001
TCP window size:  580 KByte (default)
------------------------------------------------------------
[  3] local 115.145.179.191 port 37458 connected with 115.145.179.174 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-10.0 sec  1.06 GBytes   912 Mbits/sec
[  3] MSS size 988 bytes (MTU 1028 bytes, unknown interface)
```

→ still unable to set MSS to the specific size(1000 bytes)

### ❓Questions

1. Why is it unable to set MSS to specific size?