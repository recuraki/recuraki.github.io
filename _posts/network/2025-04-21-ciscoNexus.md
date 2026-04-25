---
layout: post
title:  "Cisco Nexus"
date:   2025-04-21T00:00:00+09:00
author: Akira Kanai
categories: network
cover:  "/assets/instacode.png"
---

# Cisco Nexus

## dual-stage commit
[10.xからサポート](https://www.cisco.com/c/ja_jp/td/docs/dcn/nx-os/nexus9000/102x/configuration/system-management/cisco-nexus-9000-series-nx-os-system-management-configuration-guide-102x/m-two-stage-configuration-commit.html#concept_nql_fby_z4b)されたらしい。やったね。

### 基本的な使い方
```
cml# configure dual-stage
Enter configuration commands, one per line. End with CNTL/Z.
cml(config-dual-stage)# int lo 123
cml(config-dual-stage-if)# desc hoge
cml(config-dual-stage-if)# exit
cml(config-dual-stage)# show configuration
! Cached configuration
!
interface loopback123
 description hoge
cml(config-dual-stage)# commit
Verification Succeeded.

Proceeding to apply configuration. This might take a while depending on amount of configuration in buffer.
Please avoid other configuration changes during this time.
Configuration committed by user '' using Commit ID : 1000000002
cml(config-dual-stage)# show configuration
! Cached configuration
cml(config-dual-stage)# end
```

### commit confirmed (commitしないと戻す)
```
cml# configure dual-stage
cml(config-dual-stage)# no int lo 124
cml(config-dual-stage)# commit confirmed ?
  <30-65535>  Seconds until rollback unless there is a confirming commit
cml(config-dual-stage)# commit confirmed 30
Verification Succeeded.

Proceeding to apply configuration. This might take a while depending on amount of configuration in buffer.
Please avoid other configuration changes during this time.
Configuration committed by user 'kanai' using Commit ID : cml
cml(config-dual-stage)#
(30秒経過)
cml(config-dual-stage)# Confirm commit Timer expired, triggering rollback commit
Rollback in progress
cml(config-dual-stage)#
Configuration committed by rollback using Commit ID : 1000000005
```
confirmはcommitすればいい
```
cml(config-dual-stage)# commit confirmed 30
Verification Succeeded.
Proceeding to apply configuration. This might take a while depending on amount of configuration in buffer.
Please avoid other configuration changes during this time.
Configuration committed by user 'kanai' using Commit ID : 1000000006
cml(config-dual-stage)# commit
cml(config-dual-stage)#
cml(config-dual-stage)# show configuration
! Cached configuration
```

### commitされたdiffを取る
`from xxx to yyy`のdiffは出来ず、そのcommitで変更されたdiffだけが表示される。
```
cml# show configuration commit changes 1000000002
*** /bootflash/.dual-stage/1000000002.tmp       Fri Mar 22 12:36:28 2024
--- /bootflash/.dual-stage/1000000002   Fri Mar 22 12:36:31 2024
***************
*** 740,745 ****
--- 740,749 ----
    ip address 10.254.0.2/32

+ interface loopback123
+   description hoge
+
```

### rollbackする
そのcommitが実施されたところまで戻る。
```
cml# rollback configuration to 1000000003
Rolling back to commitID :1000000003
ADVISORY: Rollback operation started... CTRL-C is disabled.
Modifying running configuration from another VSH terminal in parallel
is not recommended, as this may lead to Rollback failure.
```

### candidateを捨てる
```
cml(config-dual-stage)# int lo 125
cml(config-dual-stage-if)# desc hoge
cml(config-dual-stage-if)# exit
cml(config-dual-stage)# abort
cml#
```

### commitできないconfigがあった時は全体が止まる
例えば、以下のように`router bgp 200`が入らない場合は`int lo 126`は無視される？＞される。

```
cml(config-dual-stage)# show configuration
! Cached configuration
!
interface loopback126
 description desc
!
router bgp 200
cml(config-dual-stage)# commit
Failed. err=bgp instance is already running; Tag is 100
commit verification failed
```

## Firmware Upgrade

### Upgrade方法
[ここ](https://www.cisco.com/c/en/us/support/switches/nexus-9000-series-switches/products-installation-guides-list.html)


### Upgrade Pathの確認
[ここからできる](https://www.cisco.com/c/dam/en/us/td/docs/dcn/tools/nexus-9k3k-issu-matrix/index.html)

# トラシュー
## mirror
```
no monitor session 1 
monitor session 1 
  shut
  source interface port-channel2 rx
  destination interface sup-eth0
  no shut
ethanalyzer local interface inband mirror display-filter icmp  limit-captured-frames 100

```

## packet forward trace
```
show troubleshoot l2 mac 0000.0000.0001 vlan 1250
```

## deepdive
```
show mac address-table address
show system internal l2fm l2dbg macdb address
show system internal l2fm info agedb 
```