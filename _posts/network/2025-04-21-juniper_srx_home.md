---
layout: post
title:  "ユーザセグメント"
date:   2025-04-21T00:00:00+09:00
author: Akira Kanai
categories: network
cover:  "/assets/instacode.png"
---

家向けSRXコンフィグ
===

## 基本設定

```
 # これは最初に必要
 set system root-authentication set system host-name srx-home
 # タイムゾーン設定
 set system time-zone Asia/Tokyo
 # タイムアウトを設定するユーザ向け
 set system login class super-user-local idle-timeout 1800
 set system login class super-user-local permissions all
 # ユーザを作る
 set system login user kanai uid 1000
 set system login user kanai class super-user-local
 set system login user kanai authentication plain
 # このあたりはご自由に(特にtelnet)
 set system services ssh
 set system services telnet
 set system services netconf ssh
 # 主にlocal向けのsyslog設定
 set system syslog archive size 10m
 set system syslog archive files 5
 set system syslog user * any emergency
 set system syslog user * authorization info
 set system syslog file messages any notice
 set system syslog file messages authorization info
 set system syslog file interactive-commands interactive-commands any
 set system syslog time-format
 # syslogを外部に出すときのsource addr
 set system syslog source-address 192.168.100.253
 # rollbackできるConfigの数
 set system max-configurations-on-flash 49
 set system max-configuration-rollbacks 49
 # NTP
 set system ntp server 210.173.160.57
 set system ntp server 210.173.160.27
 set system ntp server 210.173.160.87
 set system ntp source-address 192.168.101.252
 # netflow向け
 set forwarding-options sampling input rate 8192
 # SNMP
 set snmp community public authorization read-only
 set snmp community public clients 192.168.100.0/24
 set snmp trap-options source-address lo0
```

## ブロードバンドルータ設定
ここでは、port0をpppoeにつかい、port7をmgmtに使います。

```
# ユーザセグメント
 set interfaces fe-0/0/2 unit 0 family ethernet-switching port-mode access
 set interfaces fe-0/0/2 unit 0 family ethernet-switching vlan members v100
 set interfaces fe-0/0/3 unit 0 family ethernet-switching vlan members v100
 set interfaces fe-0/0/4 unit 0 family ethernet-switching vlan members v100
 set interfaces fe-0/0/5 unit 0 family ethernet-switching vlan members v100
 set interfaces fe-0/0/6 unit 0 family ethernet-switching vlan members v100
# ユーザ向けセグメント設定
 set vlans v100 vlan-id 100
 set vlans v100 l3-interface vlan.100
 set interfaces vlan unit 100 family inet address 192.168.1.1/24
 # mgmtセグメント
 set interfaces fe-0/0/7 description mgmt
 set interfaces fe-0/0/7 unit 0 family inet address 192.168.101.252/24
# ここをunderlayI/Fとして指定する。
set interfaces fe-0/0/0 description "pppoe uplink"
set interfaces fe-0/0/0 unit 0 encapsulation ppp-over-ether
# overlayのPPの設定
 set interfaces pp0 unit 0 ppp-options chap default-chap-secret ""
 set interfaces pp0 unit 0 ppp-options chap local-name "a@ocn.ne.jp"
 set interfaces pp0 unit 0 ppp-options chap passive
 set interfaces pp0 unit 0 pppoe-options underlying-interface fe-0/0/0.0
 set interfaces pp0 unit 0 pppoe-options auto-reconnect 10
 set interfaces pp0 unit 0 pppoe-options client
 set interfaces pp0 unit 0 family inet mtu 1454
 set interfaces pp0 unit 0 family inet negotiate-address
```

## ルータとしての基本設定
```
 set interfaces lo0 unit 0 family inet address 127.0.0.1/32
 set interfaces lo0 unit 0 family inet address 192.168.255.253/32
 set interfaces lo0 unit 0 family inet6 address fd00::253/128
 # router用の設定
 set routing-options router-id 192.168.255.253
 set routing-options autonomous-system 65000
 # RA用の設定
 set protocols router-advertisement traceoptions file ra.log
 set protocols router-advertisement traceoptions flag all
 # BGP用の基本設定
 set protocols bgp traceoptions file bgp.log
 set protocols bgp traceoptions flag open
 set protocols bgp hold-time 180
 set protocols bgp group iBGP type internal
 set protocols bgp group iBGP family inet unicast prefix-limit maximum 100
# route limitのteardown設定
 set protocols bgp group iBGP family inet unicast prefix-limit teardown idle-timeout forever
 set protocols bgp group iBGP local-as 65000
 # ospfv2,v3,lldp周りの最低限の設定
 set protocols ospf area 0.0.0.0 interface lo0.0 passive
 set protocols ospf area 0.0.0.0 interface lo0.0 metric 1
 set protocols ospf3 area 0.0.0.0 interface lo0.0 passive
 set protocols ospf3 area 0.0.0.0 interface lo0.0 metric 1
 # 以下は経路はくときのテスト用
 set routing-options rib inet6.0 static route fd00::ffff/128 discard
 set routing-options static route 255.0.0.0/32 discard

 # lldp
 set protocols lldp interface all
 set protocols lldp-med interface all
```

## ホストフィルタ
これは、SRX自身へのアクセスを制限するものです。

```
 # telnetはアドレス制限にしています
 set firewall family inet filter telnet-access term telnet-permit from source-address 192.168.100.0/24
 set firewall family inet filter telnet-access term telnet-permit from source-address 192.168.200.0/24
 set firewall family inet filter telnet-access term telnet-permit from protocol tcp
 set firewall family inet filter telnet-access term telnet-permit from destination-port telnet
 set firewall family inet filter telnet-access term telnet-permit from destination-port ssh
 set firewall family inet filter telnet-access term telnet-permit then accept
 set firewall family inet filter telnet-access term telnet-deny from protocol tcp
 set firewall family inet filter telnet-access term telnet-deny from destination-port telnet
 set firewall family inet filter telnet-access term telnet-deny from destination-port ssh
 set firewall family inet filter telnet-access term telnet-deny then discard

 # BGPに関しては定義されているneighbor単位でのacceptにします
 # これによって、bgpを用いたtcp syn attackを防ぎます
 set policy-options prefix-list bgp-peers apply-path "protocols bgp group <*> neighbor <*>;"
 set firewall family inet filter bgp-access term bgp-permit from prefix-list bgp-peers
 set firewall family inet filter bgp-access term bgp-permit from protocol tcp
 set firewall family inet filter bgp-access term bgp-permit from port 179
 set firewall family inet filter bgp-access term bgp-permit then accept
 set firewall family inet filter bgp-access term bgp-deny from protocol tcp
 set firewall family inet filter bgp-access term bgp-deny from port 179
 set firewall family inet filter bgp-access term bgp-deny then discard

 # それ以外に関しては一度すべてをpassするようにしています
 set firewall family any filter permit-all term permit-all then accept
```


##  ルーティングインスタンス
このネットワークでは、mgmtとlanのセグメントは完全に分離します。
mgmtをRIできる方法もありますが、ntpやDNSなどがRI上にあると、JUNOS 17以下ではうまく動かないので、
ユーザセグメントをRIとして切ることにします。

```
 set routing-instances lan instance-type virtual-router
 set routing-instances lan interface fe-0/0/0.0
 # pppoeはlanの出口なので、同じVLANに入れておきます
 set routing-instances lan interface pp0.0
 # vlan100はユーザ用VLAN
 set routing-instances lan interface vlan.100
 # default gateはpppoeに向けます
 set routing-instances lan routing-options static route 0.0.0.0/0 next-hop pp0.0
```

## ユーザ向けRIのDHCPd

RI上でDHCPを上げるには、system dhcpではなく、access poolで設定しないといけません（多分）

```
 set routing-instances lan system services dhcp-local-server group pool_vlan_100 interface vlan.100
 set routing-instances lan access address-assignment pool pool_vlan_100 family inet network 192.168.1.0/24
 set routing-instances lan access address-assignment pool pool_vlan_100 family inet range dhcp low 192.168.1.100
 set routing-instances lan access address-assignment pool pool_vlan_100 family inet range dhcp high 192.168.1.199
 set routing-instances lan access address-assignment pool pool_vlan_100 family inet dhcp-attributes maximum-lease-time 300
 set routing-instances lan access address-assignment pool pool_vlan_100 family inet dhcp-attributes name-server 192.168.1.1
 set routing-instances lan access address-assignment pool pool_vlan_100 family inet dhcp-attributes router 192.168.1.1
```

## mgmt向けrouting

mgmtで必要な経路を書きます。
基本的にMGMTはこのSRXで外部にroutingしません。

```
 # mgmt内のntp routing
 set routing-options rib inet.0 static route 210.173.160.57/32 next-hop 192.168.101.1
 set routing-options rib inet.0 static route 210.173.160.27/32 next-hop 192.168.101.1
 set routing-options rib inet.0 static route 210.173.160.87/32 next-hop 192.168.101.1
 set routing-options rib inet.0 static route 8.8.8.8/32 next-hop 192.168.101.1
```


## DHCP on VRF
```
# 以下は古い試行なので無視
 set system services dhcp router 192.168.1.1
 set system services dhcp pool 192.168.1.0/24 address-range low 192.168.1.2
 set system services dhcp pool 192.168.1.0/24 address-range high 192.168.1.254
 set system services dhcp propagate-settings fe-0/0/0.0
```

## NAT

普通のNAT設定です

```
 set security nat source rule-set trust-to-untrust from zone trust
 set security nat source rule-set trust-to-untrust to zone untrust
 set security nat source rule-set trust-to-untrust rule source-nat-rule match source-address 0.0.0.0/0
 set security nat source rule-set trust-to-untrust rule source-nat-rule then source-nat interface
```

## zone間ポリシ

明示的なpermit

```
 set security policies from-zone trust to-zone untrust policy trust-to-untrust match source-address any
 set security policies from-zone trust to-zone untrust policy trust-to-untrust match destination-address any
 set security policies from-zone trust to-zone untrust policy trust-to-untrust match application any
 set security policies from-zone trust to-zone untrust policy trust-to-untrust then permit
 set security policies default-policy deny-all
```

## zone設定
```
 # 家では便利性からTrustからのすべて受け取る
 set security zones security-zone trust host-inbound-traffic system-services all
 set security zones security-zone trust host-inbound-traffic protocols all
 set security zones security-zone trust interfaces vlan.100
 set security zones security-zone untrust screen untrust-screen
 set security zones security-zone untrust interfaces fe-0/0/0.0 host-inbound-traffic system-services dhcp
 set security zones security-zone untrust interfaces fe-0/0/1.0 host-inbound-traffic system-services dhcpv6
 set security zones security-zone untrust interfaces pp0.0
 # mgmt
 set security zones security-zone mgmt host-inbound-traffic system-services all
 set security zones security-zone mgmt host-inbound-traffic protocols all
 set security zones security-zone mgmt interfaces fe-0/0/7.0
```


# SRX de MAC RADIUS認証

## インストール
```
sudo apt-get instll freeradius
clients.conf
client 192.168.101.0/24{
        secret      = secret
}

service freeradius reload

"radiusd.conf"
auth = yes
        auth_badpass = yes
        auth_goodpass = yes


eap.conf
                peap {
                        use_tunneled_reply = yes
00247ffffff Auth-type:=EAP, User-Password := "00247effffff"
        Tunnel-Type = VLAN, Tunnel-Medium-Type = IEEE-802, Tunnel-Private-Group-ID = "v200"
```
のように書く。

### SRX側の設定
https://www.juniper.net/documentation/en_US/junos/topics/example/802-1x-pnac-ex-series-connecting-server-configuring.html
```
# radiusサーバの設定
set access radius-server 192.168.101.22 secret public
set access profile raspi-radius authentication-order radius
set access profile raspi-radius radius authentication-server 192.168.101.22
# インタフェースにそのプロファイルでの認証を紐づける
set protocols dot1x authenticator interface fe-0/0/6.0 mac-radius restrict
set protocols dot1x authenticator authentication-profile-name raspi-radius

set protocols dot1x traceoptions flag all
set protocols dot1x traceoptions file _dot1x
```

### 認証の確認
```
show vlans
show dot1x interface
> fe-0/0/6.0    Authenticator  Authenticated   00:24:7E:16:31:1E    00247e16311e
```


### 注意
```
set interfaces fe-0/0/6 unit 0 family ethernet-switching vlan members v100
```
とあってもRADIUSからの応答で上書きしてしまうので注意！
