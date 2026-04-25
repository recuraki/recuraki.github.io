---
layout: post
title:  "自宅ネットワークにおけるOpenFlowスイッチ OpenvSwitchの導入"
date:   2025-04-21T00:00:00+09:00
author: Akira Kanai
categories: memo
cover:  "/assets/instacode.png"
---

# 自宅ネットワークにおけるOpenFlowスイッチ OpenvSwitchの導入


![20120313_Home_Openflow](/assets/img/source/memo/Img/20120313_Home_Openflow.jpg)

## モチベーション
日本においてOpenFlowは近年のファブリック技術の発展・普及と
共に注目されている技術の一つである。

一般にOpenFlowに関してはUTMとの連携や帯域の有効活用が
現在よりも、そして、ファブリック技術よりも柔軟に可能と言われている。
しかし、また発展中の技術であること、実装が多くは存在しないことから、
OpenFlow技術自身が広く認知されているとは言い難い。

本稿では、
*自宅における生活のネットワークをOpenFlowで制御できるようにする*
ことを目的とする。

自宅における運用に際して以下の点を留意した。

* 低価格: 個人で常識を疑われない投資コストであること

* 省電力: 暑くならない(電力が高い=熱いではないけれど)

* ファンレス: 夜寝れること

機材は図の通りである。
IX2015は必ずしも必要ない。
ただし、
この場合、Vlan100に相当する部分のルータのみブロードバンドルータに担わせ、
vlan210だけ、他

コントローラはNOXを用いる。
これは、OpenFlow Tutorialなどでも用いられており、
一般に広く用いられているコントローラ実装ためである。

スイッチはOpenFlowをサポートする仮想スイッチソフトウェアの
OpenvSwitchを用いる。
仮想化の

*NEC IX2015*

ファンレスルータ。vlanを食える。トンネルもできる。L3としてしか動かず、vlanが透過できないことだけ問題。

*NetGear GS108Ev2*

ファンレススイッチ。vlanを食える。Windowsからしか管理できないがとにかく安い。7000円以下。

*EPSON NP11*

atomのPC。省電力。小さい。SSDに換装してある。

## Debianの基本的な設定

### VLAN I/Fの作成
まず、Debianに対してvlanを切る。

*必要なpkgのインストール*

 aptitude install vlan

次に、interfacesにvlan I/Fの情報を書き、
networkingをrestartする。

*/etc/network/interfaces*

```
 auto lo
 iface lo inet loopback

 auto eth2.200
 iface eth2.200 inet static
  address 192.168.200.240
  netmask 255.255.255.0
  gateway 192.168.200.254
  dns-nameservers 192.168.200.254

 auto eth2.210
 iface eth2.210 inet manual

 auto eth2.220
 iface eth2.220 inet manual
```

し、


> sudo /etc/init.d/networing restart

する。

restartのタイミングでI/Fがtaggedでないとアクセスできなくなるので注意。

eth2.XXXはeth2を物理的なI/FとしてVLANIDがXXXの
802.1aq taggedの論理I/Fを切ることを意味する。

ifaceのmanualはアドレスは手動でつけられることを意味する。
必要に応じて、I/Fがup/downする際のpre/postスクリプトを記述することも出来る。
今回の場合、Debianではvlan210, vlan220をブリッジインタフェースとして
利用するため、アドレスを付与しない(L3終端しない)ようにした。

## OpenvSwitch(ovs)インストール

まず、Debian上にソフトウェアスイッチであるovsをインストールする。
次に、ovsの初期化と先ほど作成した２つのvlan I/Fの追加を行い、
(OpenFlowスイッチとしてではなく)通常のスイッチとして動作させる。

### ソースからのインストール

(2012/03/01現在)
debianではovsのpkgは存在しない。
sidでは存在するが、ここでは、ソースからコンパイルした。


 wheezyにあげます
```
 sudo aptitude update
 sudo aptitude full-upgrade
```
 # コンパイルなどに必要なパッケージのインストール
```
 sudo apt-get install python-qt4 python-zopeinterface python-twisted-conch pkg-config
 sudo apt-get install build-essential fakeroot
```
 # レポジトリの取得
```
 mkdir ~/src
 cd ~/src
 git clone git://openvswitch.org/openvswitch
 # 導入
 cd openvswitch/
 dpkg-checkbuilddeps
   dpkg-checkbuilddeps: Unmet build dependencies: debhelper (>= 8) python-all (>= 2.6.6-3~)
```
 といわれたので、
```
 sudo apt-get install debhelper python-all
```
 してから
```
 fakeroot debian/rules binary
```
 あるいは
```
 DEB_BUILD_OPTIONS='parallel=8 nocheck' fakeroot debian/rules binary
```
 を実行。後者はunit checkが飛ばされる。
```
 cd ..
 sudo dpkg -i openvswitch-common_1.10.90-1_i386.deb
 sudo apt-get install uuid-runtime
 sudo dpkg -i openvswitch-switch_1.10.90-1_i386.deb
```
 すると、"Module openvswitch not found."
```
 /usr/share/doc/openvswitch-datapath-source/README.Debian
```
 をみろと怒られる。
```
 sudo apt-get install module-assistant
 sudo dpkg -i openvswitch-datapath-source_1.10.90-1_all.deb
 /usr/share/doc/openvswitch-datapath-source/README.Debianを見ます。
 sudo module-assistant auto-install openvswitch-datapath
 sudo dpkg -i openvswitch-switch_1.10.90-1_i386.deb
```

### あたらしいやつ

```
 ovs-vsctl --no-wait init
 # ブリッジの追加
 ovs-vsctl add-br br0
 # 確認 -> br0が見える
 sudo ovs-vsctl show
 # datapath変更
 ovs-vsctl set bridge br0 datapath_type=netdev
 # I/F追加
 sudo ovs-vsctl add-port br0 eth2.210
 sudo ovs-vsctl add-port br0 eth2.220
```

 これで、bridgeにI/Fを追加できました！

### ovsの初期設定(ここまでふるい)(旧設定)


 # ovsの裏で動くDBが格納されるディレクトリとファイルを作成
```
 sudo mkdir mkdir -p /usr/local/etc/openvswitch
 sudo ovsdb-tool create /usr/local/etc/openvswitch/conf.db vswitchd/vswitch.ovsschema
```

 # この時点で、DB Serverが動くので、まずは起動
```
 sudo ovsdb-server --remote=punix:/usr/local/var/run/openvswitch/db.sock \
 --remote=db:Open_vSwitch,manager_options \
  --bootstrap-ca-cert=db:SSL,ca_cert \
  --pidfile --detach
```

 # DBの初期化
```
 ovs-vsctl --no-wait init
```

 # 仮想スイッチを起動し、DBと接続する
 # この時点では、単なる１つのプロセスであり、どのI/Fとも接続されていない
```
 ovs-vswitchd unix:/usr/local/var/run/openvswitch/db.sock --pidfile --detach
```



## POX


```
 cd ~src
 git clone git://github.com/noxrepo/pox.git
 cd
 sudo ovs-vsctl get-controller br0
 sudo ovs-vsctl set-controller br0 tcp:127.0.0.1:6633
 sudo ovs-vsctl get-controller br0
 > tcp:127.0.0.1:6633
 sudo ovs-ofctl dump-flows br0
 > NXST_FLOW reply (xid=0x4):
 ./pox.py forwarding.l2_learning
```

## NOXのインストール
http://blog.bitmeister.jp/?p=2176
を参考にしました。


```
 cd /etc/apt/sources.list.d
 sudo wget http://openflowswitch.org/downloads/debian/nox.listsudo
 apt-get update
 sudo apt-get install nox-dependencies
 cd ~
 git clone git://noxrepo.org/nox
 cd nox
 cd ~/nox/build/src
 sudo vi +1125 /usr/lib/python2.6/dist-packages/twisted/internet/base.py
```

の下に


```python

```

```
 def _handleSigchld(self, signum, frame, _threadSupport=platform.supportsThreads()):
    from twisted.internet.process import reapAllProcesses
    if _threadSupport:
        self.callFromThread(reapAllProcesses)
    else:
        self.callLater(0, reapAllProcesses)
```

## 初めてのOpenFlow Forwarding


```
 $ ./nox_core -v -i ptcp:6633 pyswitch
 別のウィンドウで
 $ sudo ovs-vsctl get-controller br0
 $ sudo ovs-vsctl set-controller br0 tcp:127.0.0.1:6633
 $ sudo ovs-vsctl get-controller br0
 tcp:127.0.0.1:6633
 $ sudo ovs-ofctl dump-flows br0
```
