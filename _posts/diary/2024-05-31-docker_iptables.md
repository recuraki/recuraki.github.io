---
layout: post
title:  "2024/05/31日記: dockerとiptable"
date:   2024-05-31T00:00:00+09:00
author: Akira Kanai
categories: diary
cover:  "/assets/instacode.png"
---

# 2024/05/31日記: dockerとiptable
1年に1回くらい同じことではまっているのでメモです。
https://christina04.hatenablog.com/entry/iptables-outline
などのiptablesのテーブルの参照順を頭に入れておくとよいです。

Dockerの-pオプションで公開したサービスのアクセス制限をかけたいのにできない！の謎に迫ります。
```
# グローバルアドレスを持つサーバで8080へのアクセスをREJECTしたつもり
docker run --name test_nginx -d -p 8080:80 nginx
sudo iptables -A INPUT -p tcp --dport 8080 -j REJECT
# でも家のPCからアクセスできる
>> curl  -s 64.227.143.111:8080 | grep title
<title>Welcome to nginx!</title>
```

## 結論
- Dockerは`-p`を使ってサービスをpublishするとDNATが使われます。iptablesの評価順の関係で、普通のプロセスに有効な`filterのINPUT`にルールを入れるiptablesやufwでアクセス制御できません。
- `DOCKER-USER` chainにルールを記載することでアクセス制御が実現できます。この際、`-I`などでDOCKER-USERの既存のルールよりも前にルールを挿入します。
```
※この時、dportはコンテナ側のポート番号である必要があります。
sudo iptables -I DOCKER-USER  -p tcp -m tcp -s 8.8.8.8/32 --dport 80 -j REJECT --reject-with icmp-port-unreachable
```

## 予備知識(docker起動時のI/F)
- dockerを使うとdocker0というI/Fがホスト上に作成される。このインタフェースはブリッジとして動作し、IPアドレスを持つ。
```
kanai@ubuntu-s-1vcpu-1gb-blr1-01:~$ brctl show | grep dock
docker0         8000.02420becbfec       no
kanai@ubuntu-s-1vcpu-1gb-blr1-01:~$ ip addr show docker0
4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:0b:ec:bf:ec brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
```
- I/Fをもつコンテナを通常起動すると、ホストのdocker0のブリッジに接続されたコンテナのI/Fが作成され、アドレスを持つ。
```
# 特に-pをつけずにnginxを実行した例
kanai@ubuntu-s-1vcpu-1gb-blr1-01:~$ docker run --name test_nginx -d nginx
kanai@ubuntu-s-1vcpu-1gb-blr1-01:~$ brctl show | grep dock
docker0         8000.02420becbfec       no              veth6deea54
kanai@ubuntu-s-1vcpu-1gb-blr1-01:~$ docker inspect test_nginx
...
            "Networks": {
                "bridge": {
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.2",
以上は-pをつけても同様である。
```

## 予備知識(iptables)

- iptablesはパケットを評価するLinuxの仕組みであり、条件に一致したパケットに対して`TARGET`の処理をおこなう。ACCEPTとDROPは特殊な意味を持つ`TARGET`でそのテーブルの処理を終えて次に進む、あるいは、パケットを拒否する。
- iptablesは複数のテーブルを持つ。今回は`filter`と`nat`が登場する。以下では`今回のシナリオに関係ある部分のみ`を説明する。
  - パケットが着信するとまず`natのPREROUTING`が評価されてDNATの対象であるかが決まる。DNATとはルータにおいて条件に一致したパケットのDestinationのIPアドレス and/or ポート番号を書き換えてパケット転送(Packet Forward)する機能である。
  - パケットがDNATの対象である場合パケットはルータの内部I/F側に転送されるためIP転送(FORWARD)される。この際は`filterのFORWARD`テーブルが評価される。パケットは自宛でないので`filterのINPUT`では評価されない。
  - パケットがDNATの対象でない場合、そのパケットは自宛であろうから(そうでなければ破棄する)、`filterのINPUT`で評価されてホストOS側に渡される。
- iptablesはあるテーブルに対して`chain`を設定できる。chainとはサブルーチンのようなものであり、可読性を高めることができる。chainの中では上記のほかにRETURN TARGETがあり次のchainに結果を委ねる。
- iptablesはすべてのテーブルでACCEPTかDROPが定まらない場合は、そのテーブルにiptablesの`-P`で示されたデフォルトの振る舞いでACCEPTかDROPを決める。

## dockerの振る舞いとそれによりコンテナに届くパケットの流れ

- dockerの-pオプションとはなにか？`-p <outerport>:<innerport>`を実行するとホスト側の`<outerport>`に着信したパケットは、そのコンテナの`<innerport>`に転送される。iptablesのDNAT機能を用いてこれを実装する。ホスト側のプロセスでLISTENするのでは`ない`ことに注意する。つまり、netstat(ss)では見えない。
- dockerはiptablesにより2つのルールを設定する。
  - 1つ目のルールは`natのPREROUTING`に対して、`ホスト側のインタフェースに着信した<outerport>宛のパケットをコンテナIPアドレス:<innerport>にDNATする`である。このルールではouter側のインタフェースdocker0は指定されないが、`172.17.0.2/16`などのdocker0と同じネットワーク上の宛先が指定されるのでdocker0 I/Fへの転送が行われる。例を示す。
```
-A DOCKER ! -i docker0 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
```
  - 2つめのルールは`filterのFORWARDテーブル`に対して`docker0インタフェースに着信した<innerport>宛のパケットをACCEPTする`である。(このルールの含まれる`DOCKER chain`がFORWARDのかなり早い段階で評価される)。例を示す。
```
-A DOCKER -d 172.17.0.2/32 ! -i docker0 -o docker0 -p tcp -m tcp --dport 80 -j ACCEPT
```
- 変換された`コンテナのIPアドレス`とはホストのOS自身には付与されていないIPアドレスである。そのため自宛のパケットへの評価である`filterのINPUT`は評価されない。しかし、パケット転送であるため`filterのFORWARD`に評価される。

## "一般的な"iptablesのfilterやufwによるアクセス制御
iptablesやufwを使うとホスト側のソケットを用いたプログラムへのアクセス制御が実現できます。例えば、
```
例1: sudo iptables -A INPUT -p tcp --dport 8000 -j REJECT
例2: sudo ufw deny 8000/tcp
```
のように実現できます。ところが、dockerではこれらのアクセス制御が`一切効きません`。
- 例1について、テーブルを設定していませんがiptablesの暗黙のテーブルは`filter`です。上記のコマンドは`filterのINPUT`への追加(`-A`)であるためDNATされるパケットには一切効きません。
- 例2について、ufwとはiptablesのwrapperです。以下のルールを`filter`に追加しますがこれは`filterのINPUT`のchainであるのでDNATされるパケットには一切効きません。
```
-A ufw-user-input -p tcp -m tcp --dport 8000 -j DROP
```
- 例2: 補足`sudo ufw route deny 8080/tcp`とすると以下のように、ufwのFORWARD上のtableのchainにルールが追加されるのですがDockerによってDNATされるパケットには一切効きません。なぜなら`filterのFORWARD`の`DOCKER chain`によってこのパケットをACCEPTするルールが存在するためです。
```
-A ufw-user-forward -p tcp -m tcp --dport 8080 -j DROP
```

## 参考: pオプションへのlistenアドレス追加
ホストからのローカルアクセスのみでいい場合`-p <addr>:<outerport>:<innerport>`を使って`-p 127.0.0.1:<outerport>:<innerport>`という書き方ができます。この場合は何が起きているのでしょうか？`natのPREROUTING`へ条件が追加されます。
```
-A DOCKER ! -i docker0 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
 → <addr>=127.0.0.1を指定した場合
-A DOCKER -d 127.0.0.1/32 ! -i docker0 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
```
この書き方は127.0.0.1宛のパケットのみをDNATするというルールであるので細かなアクセス制御はできません。

## どうすればよい？
dockerにより追加される`filterのFORWARD`のルールは以下の順です。
```
-A FORWARD -j DOCKER-USER
-A FORWARD -j DOCKER-ISOLATION-STAGE-1
-A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -o docker0 -j DOCKER 
-A FORWARD -i docker0 ! -o docker0 -j ACCEPT
-A FORWARD -i docker0 -o docker0 -j ACCEPT
...
-A DOCKER -d 172.17.0.2/32 ! -i docker0 -o docker0 -p tcp -m tcp --dport 80 -j ACCEPT
-A DOCKER-USER -j RETURN
```
`DOCKER chain`に含まれるルールがコンテナへの通信をACCEPTしていますがこのルールより前に`DOCKER-USER chain`が呼ばれています。ここにアクセス制御したいルールを書けばよいです。今回であれば、
```
sudo iptables -I DOCKER-USER  -p tcp -m tcp -s 8.8.8.8/32 --dport 80 -j REJECT --reject-with icmp-port-unreachable
```
なとどすれば特定ホストからのアクセスのみを落とすことができます。この時に、`-I`オプションなどを使ってDOCKER-USERに最初から含まれる`RETURN`よりも前にルールを追加することを忘れないでください。

## おまけ
そもそも、`natのPREROUTING`でフィルタすれば？...というアイデアは`nat`テーブル内でこれが実現できないのでダメです。
```
$ sudo iptables -t nat -I DOCKER ! -i docker0 -p tcp -m tcp --dport 8080 -j DROP
The "nat" table is not intended for filtering, the use of DROP is therefore inhibited.
```

### Appendix

### 背景
```
python3 -m http.server 8000
sudo iptables -A INPUT -p tcp --dport 8000 -j REJECT
とすると
>> curl http://64.227.143.111:8000
curl: (7) Failed to connect to 64.227.143.111 port 8000 after 132 ms: Couldn't connect to server
# sudo iptables -D INPUT -p tcp --dport 8000 -j REJECT
```

```
sudo ufw allow 22/tcp
sudo ufw allow 8000/tcp
# sudo ufw delete allow 8000/tcp
sudo ufw enable
# sudo ufw status verbose
kanai@akanai-mbp14 ~ exectime:296ms
>> curl -s http://64.227.143.111:8080 | grep title
... timeout ...
```

とできるのに

```
docker run --name test_nginx -d -p 8080:80 nginx
sudo iptables -A INPUT -p tcp --dport 8080 -j REJECT
としても
kanai@akanai-mbp14 ~ exectime:296ms
>> curl -s http://64.227.143.111:8080 | grep title
<title>Welcome to nginx!</title>
だし、
sudo ufw enable
# > default deny
>> curl http://64.227.143.111:8080 -s | grep title
<title>Welcome to nginx!</title>
```

## 準備: 環境作り
dockerのホストが欲しかったのでDigitalOceanでVPSを借りました。
```
ssh root@<vpshost>
sudo apt-get update
sudo apt -y install docker.io
useradd kanai
sudo usermod -aG docker kanai
sudo usermod -aG sudo kanai
su kanai
ssh-import-id-gh recuraki
抜けて
ssh <vpshost>
```

## iptable
- inについて、PREROUTING nat -> (routing) -> INPUT filter
- outについて、OUTPUT nat -> OUTPUT filter -> (routing) -> POSTROUTING nat

## log

```
$ docker run --name test_nginx -d -p 8080:80 nginx
kanai@ubuntu-s-1vcpu-1gb-blr1-01:~$ sudo iptables -S | grep -v ufw | cat -n
     1	-P INPUT ACCEPT
     2	-P FORWARD ACCEPT
     3	-P OUTPUT ACCEPT
     4	-N DOCKER
     5	-N DOCKER-ISOLATION-STAGE-1
     6	-N DOCKER-ISOLATION-STAGE-2
     7	-N DOCKER-USER
     8	-A INPUT -p tcp -m tcp --dport 8088 -j REJECT --reject-with icmp-port-unreachable
     9	-A FORWARD -j DOCKER-USER
    10	-A FORWARD -j DOCKER-ISOLATION-STAGE-1
    11	-A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
    12	-A FORWARD -o docker0 -j DOCKER
    13	-A FORWARD -i docker0 ! -o docker0 -j ACCEPT
    14	-A FORWARD -i docker0 -o docker0 -j ACCEPT
    15	-A DOCKER -d 172.17.0.2/32 ! -i docker0 -o docker0 -p tcp -m tcp --dport 80 -j ACCEPT
    16	-A DOCKER-ISOLATION-STAGE-1 -i docker0 ! -o docker0 -j DOCKER-ISOLATION-STAGE-2
    17	-A DOCKER-ISOLATION-STAGE-1 -j RETURN
    18	-A DOCKER-ISOLATION-STAGE-2 -o docker0 -j DROP
    19	-A DOCKER-ISOLATION-STAGE-2 -j RETURN
    20	-A DOCKER-USER -j RETURN
kanai@ubuntu-s-1vcpu-1gb-blr1-01:~$ sudo iptables -S -t nat| grep -v ufw
kanai@ubuntu-s-1vcpu-1gb-blr1-01:~$ sudo iptables -S -t nat| grep -v ufw | cat -n
     1	-P PREROUTING ACCEPT
     2	-P INPUT ACCEPT
     3	-P OUTPUT ACCEPT
     4	-P POSTROUTING ACCEPT
     5	-N DOCKER
     6	-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
     7	-A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER
     8	-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
     9	-A POSTROUTING -s 172.17.0.2/32 -d 172.17.0.2/32 -p tcp -m tcp --dport 80 -j MASQUERADE
    10	-A DOCKER -i docker0 -j RETURN
    11	-A DOCKER ! -i docker0 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
```

- nat `     6	-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER`が凶悪な場所におり、全ての自宛てをdockerに飛ばす。dst-type LOCALは自分のホストが持っているアドレスと考えて良い。
- nat 11の`11	-A DOCKER ! -i docker0 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80`はinがdocker0でないtcp/8080をdocker側の80にマップする。

## table形式

```
kanai@ubuntu-s-1vcpu-1gb-blr1-01:~$ sudo iptables -L -v -n  --line-numbers
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy DROP 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0
2        0     0 DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0
3        0     0 ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
4        0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0
5        0     0 ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0
6        0     0 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination

Chain DOCKER (1 references)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 ACCEPT     tcp  --  !docker0 docker0  0.0.0.0/0            172.17.0.2           tcp dpt:80

Chain DOCKER-ISOLATION-STAGE-1 (1 references)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 DOCKER-ISOLATION-STAGE-2  all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0
2        0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain DOCKER-ISOLATION-STAGE-2 (1 references)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 DROP       all  --  *      docker0  0.0.0.0/0            0.0.0.0/0
2        0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain DOCKER-USER (1 references)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0
kanai@ubuntu-s-1vcpu-1gb-blr1-01:~$ sudo iptables -L -v -n -t nat --line-numbers
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1       78  4032 DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0
2        0     0 MASQUERADE  tcp  --  *      *       172.17.0.2           172.17.0.2           tcp dpt:80

Chain DOCKER (2 references)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0
2        0     0 DNAT       tcp  --  !docker0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:172.17.0.2:80
```

