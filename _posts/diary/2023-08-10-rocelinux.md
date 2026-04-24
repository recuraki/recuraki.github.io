---
layout: post
title:  "20230810 VMでRoCEv2Pingをする"
date:   2023-08-10T00:00:00+09:00
author: Akira Kanai
categories: diary
cover:  "/assets/instacode.png"
---

# 20230810 VMでRoCEv2Pingをする
--

## 前提

- おなじセグメント内でいるLinuxがVMWareなどで立ち上がっている。(同じセグメントは必須要件ではない)
- NW I/Fはens34とする

## 共通

```
IF=ens34
sudo apt -y install infiniband-diags ibverbs-utils infiniband-diags 
sudo apt -y install perftest rdma-core wireshark  python3-pip libibverbs-dev
sudo apt -y install rdmacm-utils tshark
rdma link show
 > なにもない
ls /sys/class/infiniband/
 > なにもない
sudo rdma link add rxe0 type rxe netdev $IF
rdma link show
 > link rxe0/1 state ACTIVE physical_state LINK_UP netdev ens34
ls /sys/class/infiniband/
 > rxe0

sudo modprobe rdma_ucm 
sudo modprobe ib_uverbs
sudo modprobe ib_iser 
sudo modprobe ib_umad
sudo modprobe ib_ipoib

```

## 状態の確認
### ibstat
```
kanai@ubuntu1:~$ ibstat
 > ここで、LIDが0のままだがよい。
 > IBの場合DiscoverされるがRoCEの場合LIDはつかない。
CA 'rxe0'
        CA type:
        Number of ports: 1
        Firmware version:
        Hardware version:
        Node GUID: 0x020c29fffe9122a4
        System image GUID: 0x020c29fffe9122a4
        Port 1:
                State: Active
                Physical state: LinkUp
                Rate: 10 (FDR10)
                Base lid: 0
                LMC: 0
                SM lid: 0
                Capability mask: 0x00010000
                Port GUID: 0x020c29fffe9122a4
                Link layer: Ethernet
```
### ibv_devinfo  -v
```
ibv_devinfo  -v
---
hca_id:	rxe0
	num_comp_vectors:		2
		port:	1
			GID[  0]:		fe80::20c:29ff:fe46:4c04, RoCE v2
			GID[  1]:		::ffff:192.168.244.128, RoCE v2
            GID[  4]:               fe80::6ddb:b1ea:ec80:a442, RoCE v2            
# ここで、ipv6のついているモノがあるが、 ip addr showなどでどれが使われているのかを見ておく
# 私の環境ではGID0のアドレスが ip addr showでens34についていなかった...
---
```

## ping

### rping(IPv4)
```
kanai@ubuntu1:~$ rping -s -a 192.168.101.174 -C 10 -v
 > -aでLISTENを指定する必必要はない。rping -sでもOK
 server ping data: rdma-ping-0: ABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqr
 server DISCONNECT EVENT...
kanai@ubuntu2:~$ rping -c -a 192.168.101.174 -C 2 -v
 ping data: rdma-ping-0: ABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqr
 ping data: rdma-ping-1: BCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrs 

tcpdump
 14:53:14.201551 IP 192.168.101.147.55410 > 192.168.101.174.4791: UDP, length 280
 14:53:14.201572 IP 192.168.101.147.64713 > 192.168.101.174.4791: UDP, length 32
```

### rping(IPv6)
```
kanai@ubuntu1:~$ rping -s -a ::0
 > -a で::0を指定しないといけない
 >         -s              server side.  To bind to any address with IPv6 use -a ::0
kanai@ubuntu2:~$ rping -c -a fe80::6ddb:b1ea:ec80:a442%ens34 -C 2 -v

tcpdump

14:52:31.957600 IP6 fe80::9f3c:546c:7e0e:29ca.49984 > fe80::6ddb:b1ea:ec80:a442.4791: UDP, length 32
14:52:31.957623 IP6 fe80::6ddb:b1ea:ec80:a442.49984 > fe80::9f3c:546c:7e0e:29ca.4791: UDP, length 20

```

### IP Verb ping(IPv4)
```
Server
kanai@ubuntu1:~$ sudo  ibv_rc_pingpong -d rxe0 -g 1
  local address:  LID 0x0000, QPN 0x000014, PSN 0xb86d37, GID ::ffff:192.168.101.174
  remote address: LID 0x0000, QPN 0x000014, PSN 0x71370e, GID ::ffff:192.168.101.147
Client
ibv_rc_pingpong -d rxe0 -g 1  192.168.101.174 
  local address:  LID 0x0000, QPN 0x000014, PSN 0x71370e, GID ::ffff:192.168.101.147
  remote address: LID 0x0000, QPN 0x000014, PSN 0xb86d37, GID ::ffff:192.168.101.174

Server (tcpdump)
sudo tcpdump -ni ens34
 14:33:20.876929 IP 192.168.101.174.49593 > 192.168.101.147.4791: UDP, length 20
 14:33:20.876943 IP 192.168.101.174.49593 > 192.168.101.147.4791: UDP, length 1040 

tcpdump 
 14:47:58.174977 IP 192.168.101.147.57730 > 192.168.101.174.4791: UDP, length 32
 14:47:58.174990 IP 192.168.101.174.57730 > 192.168.101.147.4791: UDP, length 20

IPv4で通信されている
```

### IP Verb ping(IPv6)
ここで興味深いのは以下でClient側がv4でtargetを指定しているものの、IPv6で通信されていること
```
Server
kanai@ubuntu1:~$ sudo  ibv_rc_pingpong -d rxe0 -g 4
  local address:  LID 0x0000, QPN 0x00001b, PSN 0x600c60, GID fe80::6ddb:b1ea:ec80:a442
  remote address: LID 0x0000, QPN 0x00001b, PSN 0x5fe13c, GID fe80::9f3c:546c:7e0e:29ca
kanai@ubuntu2:~$ ibv_rc_pingpong -d rxe0 -g 4  192.168.101.174
  local address:  LID 0x0000, QPN 0x00001b, PSN 0x5fe13c, GID fe80::9f3c:546c:7e0e:29ca
  remote address: LID 0x0000, QPN 0x00001b, PSN 0x600c60, GID fe80::6ddb:b1ea:ec80:a442


Server (tcpdump)
14:39:00.613401 IP6 fe80::9f3c:546c:7e0e:29ca.49936 > fe80::6ddb:b1ea:ec80:a442.4791: UDP, length 1040
14:39:00.613404 IP6 fe80::9f3c:546c:7e0e:29ca.49936 > fe80::6ddb:b1ea:ec80:a442.4791: UDP, length 1040
```
