---
layout: post
title:  "OpenFlow チュートリアル on OSX"
date:   2025-04-21T00:00:00+09:00
author: Akira Kanai
categories: network
cover:  "/assets/instacode.png"
---

# OpenFlow チュートリアル on OSX


TODO:
Xのインストールが必要か調査


`OpenFlow Tutorial <http://archive.openflow.org/wk/index.php/OpenFlow_Tutorial>` 

mininet-2.0.0-113012-amd64-ovf.zip
は解凍して適当なディレクトリにコピーします。
解凍したディレクトリでVMのイメージができるので、気になる人は
適当なディレクトリに移します。

まず、VirtualBoxをインストールします。
クリックでそのまま進めればOKです。

![of_virtualbox_install](/assets/img/source/network/Img/of_virtualbox_install.png)

VirtualBoxを起動します。

![of_virtualbox_init](/assets/img/source/network/Img/of_virtualbox_init.png)

ファイル -> インポート -> アプライアンスを開く
先ほどのファイルをインポートします。(これには1,2分くらいかかります。)

![of_vm_file](/assets/img/source/network/Img/of_vm_file.png)

Finish VM Setupの欄にあるとおり、重要なネットワークインタフェースを作ります。
これは、皆さんのmacとOpenFlowVMを繋ぐ、ローカルネットワークです。

まずmac自身に、mac<->VM用のNICを作ります。VirtualBox自身を選択し、
VirtualBox -> 環境設定 -> ネットワーク -> ホストオンリーネットワーク
で、「＋」を選択すると、図のようにネットワークが作成されるはずです。
（こだわりのある人は適当に設定を変えても大丈夫です）

![of_vm_adpt](/assets/img/source/network/Img/of_vm_adpt.png)

次に、このNICとVMのNICを繋ぎましょう。
作ったVMを選択 -> 上のバーから設定 -> ネットワーク -> アダプター2 
を開き、
ネットワークアダプタを有効化
割り当てで「ホストオンリーアダプター」を選択してください。
名前は先ほど作成したvboxnet0が選択されると思います。

![of_vm_hostbase](/assets/img/source/network/Img/of_vm_hostbase.png)

画面のようなメッセージが出ますが、「フォーカスを取られるよ」というメッセージです。
この後にも「マウスのフォーカスを取られるよ」といわれますが、同様です。
Donot showにしてしまっていいでしょう。
VMから抜けるときは、左のコマンドボタンを押します。

![of_vm_warn](/assets/img/source/network/Img/of_vm_warn.png)

さて、mininet:mininetでログインできます。
まずは、


```
dhclient eth1
ifconfig eth1
```

を入力します。
eth1は先ほど作成したホストオンリーネットワークです。
おそらく192.168.56.101というアドレスが確認できるはずです。
(DHCPの割り当ての最初のアドレスだから。適宜読み替えてください。)

OSXから、


```
LC_ALL=C ssh -X mininet@192.168.56.101     
```

と実行し、"mininet"でログインします。
このVMは、macをNAPTとして外部疎通性を持ちます。
ping 8.8.8.8
など実行すると、外に抜けていけるはずです。
尚、LC_ALL=Cは言語互換性のために「念のため」入れています。
-XはX Forwardingのオプションになります。

さて、これで、皆さんのPC上にOpenFlowの箱庭ができました。
VirtualBoxのウィンドウは小さくしてしまって構いません。

次に、"xterm"と入力します。
別のウィンドウでコンソールが出てきたと思います。

![of_xterm](/assets/img/source/network/Img/of_xterm.png)

これは、OpenFlowVMで動いているVMのコンソールです。
先ほど、sshする際に-Xオプションをつけたので、X Forwardが有効になっています。
そのため、VM上で起動したxterm(最も基本的なコンソールプログラム)
のウィンドウが皆さんのOSXの上に表示されています。

さて、このウィンドウはCtl+Dや"exit"で抜けてしまいましょう。

尚、xterm[RET]とすると、その起動元のプログラムはxtermを終了するまで
何もアクションができません。
複数のxtermを起動したいときは、
xterm &
等と"&"をつけて実行することで、xtermの戻りを待たずに次のコマンドを入力できます。
(つまり、複数、xterm &と打てば、複数のウィンドウを起動できます)

では、チュートリアルを進めます。

## Start Network
まず、SSHした画面(xtermではありません！)から、


```
sudo mn --topo single,3 --mac --switch ovsk --controller remote
```

と入力します。
これで、チュートリアルにあるトポロジが作成されました。

TODO: トポロジの解説

mininetの実行中は仮想的にこれらのホストを生成しています。
(興味のある人はmininet実行中にsshでホストに入り、ifconfigしてみてください)


```
h1 <cmd>
h1 ifconfig
h1 ifconfig | grep inet 
```

でh1のノード上でifconfigを実行できます。"|"についても、このノード上で実行されます。


```
xterm h1 h2
```

と入力することで、仮想的なノードh1,h2のコンソールが出せます。h1のウィンドウで


```
ping 10.0.0.2
```

とすればh2にpingが届く。。。ように思えますが、ここでは成功しません。
なぜなら、まだ、OpenFlowスイッチには何らフローが入っておらず、
h1からスイッチに入力したパケットはスイッチで破棄されてしまうからです！

ここからチュートリアルはスタートです！
この仮想環境で、まずはダムハブを作成し、pingが到達するようにします。
最終的にはスイッチを作成し、pingが到達するようにするのです！

Note:
mininetはLinuxシステム上に仮想インタフェースを作り、
その上で、あたかも違うネットワークインタフェースを持つホストが存在するかのように
振る舞うソフトウエアです。
このため「ごみ」が残った場合、


```
sudo mn -c
```

とすることで、mininetの情報をクリアすることができます。


## dpctl Example Usage

さて、
今のmininetは起動したままで、
ここで、「もう一つ」sshのウィンドウを開きます。
OSXなら、ターミナルのウィンドウを増やし(Cmd+N)


```
LC_ALL=C ssh mininet@192.168.56.101       
```

してください。(こちらではXはいらないので-Xオプションは不要です)

一つめのウィンドウはmininetを起動しており、
xterm h1など、mininet上の操作を行います。
今作成したウィンドウは、OpenFlowスイッチを制御するウィンドウです。

ヒント:
OpenFlowのコードを書いていると複数のVM上のterminalがほしくなるかも知れません。
ある程度、xtermに慣れているなら、
xtermを複数開いておくと良いでしょう。

OpenFlowのコントローラを書く前に、dpctlを学習します。
これは、OpenFlowスイッチの様々な情報が取得できるプログラムです。
showコマンドは、OFスイッチのポート情報などを見ることができます。
showの後には以下のように入力します。


```
$ dpctl show tcp:127.0.0.1:6634
features_reply (xid=0xbfc1b5aa): ver:0x1, dpid:1
n_tables:255, n_buffers:256
features: capabilities:0xc7, actions:0xfff
1(s1-eth1): addr:12:7b:af:e3:8d:37, config: 0, state:0
 current:    10GB-FD COPPER 
2(s1-eth2): addr:06:9b:e9:cf:c5:3b, config: 0, state:0
 current:    10GB-FD COPPER 
3(s1-eth3): addr:1a:07:c4:d8:45:05, config: 0, state:0
 current:    10GB-FD COPPER 
LOCAL(s1): addr:76:af:79:ae:b3:48, config: 0x1, state:0x1
get_config_reply (xid=0x1538fa22): miss_send_len=0
```

これで、VM上で動作しているOpenvswitchの情報を取得できました。
次に、この上に存在するフロー情報を取得してみましょう。


```
mininet@mininet-vm:~$ dpctl dump-flows tcp:127.0.0.1:6634
stats_reply (xid=0x897d2abf): flags=none type=1(flow)
```

おや？特に出力がありません。
これは、現在、「OpenFlowに登録されているフローがない」からです。

では、初めてのフローを手動で作成しましょう。
mininetで以下のように入力します。


```
mininet> h1 ping  -c3 h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
From 10.0.0.1 icmp_seq=1 Destination Host Unreachable
From 10.0.0.1 icmp_seq=2 Destination Host Unreachable
From 10.0.0.1 icmp_seq=3 Destination Host Unreachable
```

先ほどと同じです。pingは帰ってきません。

メモ：
このコマンドは不思議です。
本質的に、このコマンドは、h1上でping -c3 h2を実行します。
ですが、xterm h1 h2でxtermを開いて、h1上から
ping h2としても、h2と入力してもh2の名前を解決できないのに、
なぜh2(10.0.0.2)にpingが実行されているのでしょうか？
その秘密は、mininetのshellにあります。


```
mininet> h1 echo s1 - c0 - h1 - h2 
127.0.0.1 - 127.0.0.1 - 10.0.0.1 - 10.0.0.2
```

なるほど！mininetはh1 <cmd>を実行する際に、<cmd>に含まれる"nodes"の値を
勝手に置換していたんですね！

では、ついに初めてのフローを定義しましょう！


```
dpctl add-flow tcp:127.0.0.1:6634 in_port=1,actions=output:2
dpctl add-flow tcp:127.0.0.1:6634 in_port=2,actions=output:1
dpctl dump-flows tcp:127.0.0.1:6634
stats_reply (xid=0x6493079): flags=none type=1(flow)
cookie=0, duration_sec=21s, duration_nsec=836000000s, table_id=0, priority=32768, n_packets=0, n_bytes=0, idle_timeout=60,hard_timeout=0,in_port=1,actions=output:2
cookie=0, duration_sec=21s, duration_nsec=199000000s, table_id=0, priority=32768, n_packets=0, n_bytes=0, idle_timeout=60,hard_timeout=0,in_port=2,actions=output:1
```

おや！先ほどはなにも表示されなｋった、dump-flowで出力が現れました！
この後、「すぐに」＝60秒以内に mininetのウィンドウでさっきのコマンドを
再試行します。


```
mininet> h1 ping  -c3 h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_req=1 ttl=64 time=0.145 ms
64 bytes from 10.0.0.2: icmp_req=2 ttl=64 time=0.060 ms
64 bytes from 10.0.0.2: icmp_req=3 ttl=64 time=0.046 ms
```

やった！pingが通りました！
しかし、しばらくするとpingが通らなくなってしまいます。
これは、フローを追加する際には、生存時間が決められており、
デフォルトでは60secとなっているからです。
先ほどのdump-flowの結果を見ると、"idle_timeout=60"になっていることが分かります。

これは、


```
dpctl add-flow tcp:127.0.0.1:6634 in_port=1,idle_timeout=120,actions=output:2
```

のようにすれば生存時間を延ばすことができます。

## Start Wireshark

次に、このVMにはwiresharkが含まれているので、その使い方を説明します。
また、最近のwiresharkでは、標準でof用のフィルタが含まれているので
その使い方も学びます。

openflow用に開いているウィンドウでそのまま


```
sudo wireshark &
```

と入力します。(筆者の環境では、起動時に若干エラーが出ましたが、特に問題ありませんでした。)

![of_wireshark](/assets/img/source/network/Img/of_wireshark.png)

起動後、Caputureでlo0を選択し、Startします。
画面上部のFilterに"of"と入力し、applyします。

![of_wireshark_2](/assets/img/source/network/Img/of_wireshark_2.png)

次に、Wiresharkを起動したホスト上で、


```
controller ptcp:
```

と入力してください。
このコマンドは、ローカルホスト上で、「簡単なスイッチ」を扱うコントローラです。

実行してしばらくすると、wireshark上にパケットが見えます。
この状態で、mininetから、
pingを実行します。すると、応答が帰ってくると共に、
wiresharkにいくつかのパケットが見えます。


```
mininet> h1 ping  -c3 h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_req=1 ttl=64 time=2.08 ms
64 bytes from 10.0.0.2: icmp_req=2 ttl=64 time=0.248 ms
64 bytes from 10.0.0.2: icmp_req=3 ttl=64 time=0.046 ms
```

OFP+ICMPとOFP+ARPのパケットが見えましたか？
そうです。h1とh2の通信の一部がここに見えます。

![of_wireshark_3](/assets/img/source/network/Img/of_wireshark_3.png)

ところが、これらについて、
pingを3回試行したのに、OFP+ICMPが2パケット＝一発分しか見えないことに気がつきましたか？

そうです。OpenFlowのコントローラは、初めてこれらのパケットが見えたとき、
フローを学習し、OpenFlowスイッチにそのフローをインストールします。
二発目移行のフローはスイッチにフローがインストールされているため、
コントローラまで上がらないのです。



```
メモ:
sudo mn --topo single,3 --mac --switch ovsk --controller remote
mininet> iperf
*** Iperf: testing TCP bandwidth between h1 and h3
waiting for iperf to start up...*** Results: ['2.16 Gbits/sec', '2.16 Gbits/sec']
sudo mn --topo single,3 --mac --controller remote --switch user
の比較はうまくいかなかったんでパス。
```

## Controller Choice: POX (Python)
ここまででOFスイッチを作成し、
既存のコントローラを使って、pingが届くようになりました。
では、いよいよ、PythonのOFコントローラ実装POXを使って、
スイッチを作っていきます。

まず、環境の初期化を行います。
今、mininetを開いているディレクトリで以下の通り入力し、
新しく、mininetを作り直します。


  #もしログインしている場合は抜ける
  mininet> exit
  sudo killall controller
  $ sudo mn -c
  $ sudo mn --topo single,3 --mac --switch ovsk --controller remote

次に、コントローラを開いていたウィンドウで以下の通り入力します。


  # このVMには既にPOXのレポジトリが存在します
  cd /home/mininet/pox
  # 最新版のpoxをpullします
  git pull
  ./pox.py log.level --DEBUG samples.of_tutorial
  > ...
  > DEBUG:openflow.of_01:Listening for connections on 0.0.0.0:6633
  > Ready.
  > POX> INFO:openflow.of_01:[Con 1/1] Connected to 00-00-00-00-00-01
  > DEBUG:samples.of_tutorial:Controlling [Con 1/1]

さて、これで、poxを起動しました。
pox.pyの後の引数について説明します。
log.levelはpox/log/level.pyを「読み込み」ます。
このファイルはデバッグメッセージを出力するモジュールで、
この後の引数--DEBUGがデバッグレベルを示します。
ここではおまじないだと覚えておけばOKです。
つぎに、samples.of_tutorialは、
pox/samples/of_tutorial.pyというファイルを読み込みます。
これが、コントローラを記述するコードです。
これからこのファイルを触っていきます。

さて、今、mininet上のOFスイッチはハブとして動作しています。
これを確認しましょう。

mininetでh1,h2,h3のxtermを開き、それぞれのウィンドウで次の通り入力します。


```
mininet> xterm h1 h2 h3
# h2のウィンドウ
tcpdump -XX -n -i h2-eth0
# h3のウィンドウ
tcpdump -XX -n -i h3-eth0
```

これらが実行できたら、h1のウィンドウで次のように入力してください。


 # h1 -> h2へのping
 ping -c3 10.0.0.2

h2,h3両方のウィンドウでicmpのメッセージが見えましたか？
いま、h1->h2の通信がh3にも見えています。
まさに、ハブとして動作しています！
ちなみに、


 # h1 -> 存在しないホストにping
 ping -c1 10.0.0.5

すると、arpが見えます。なるほど。ARPも正しくすべてのノードに転送されているようです。
それではxtermを「すべて閉じて」ください。

ではいったい、今、どのくらいのスループットが出るのでしょう？


```
mininet> pingall
mininet> iperf
*** Iperf: testing TCP bandwidth between h1 and h3
*** Results: ['5.46 Mbits/sec', '5.44 Mbits/sec']
```

遅い！どうして？
すべてのパケットがコントローラに上がってきているのです！
TODO: ここ追加（フローがはいってるんじゃないよってこと)


```
cp pox/samples/of_tutorial.py pox/samples/of_tutorial.py.orig
```

