---
layout: post
title:  "【未完】iptables + pythonする"
date:   2014-10-22T00:00:00+09:00
author: Akira Kanai
categories: diary
cover:  "/assets/instacode.png"
---

# 20141022: 【未完】iptables + pythonする

## 背景

先週からの流行はSSLv3.0であるので、Mallory(マロリー)の気持ちになってみる。
BEASTはCBCかつ、AES共通鍵暗号などでならなければならない。
実際に能動的な攻撃者がプロトコルのバージョンを落とせるか考察する。

## 作戦

Queueでインジェクション、改ざんする。


## SSLサーバをたてる

http://pokotsun.mydns.jp/?p=775

などを参考にすればよい

```
 sudo su
 a2enmod ssl
 a2ensite default-ssl
 cd /etc/apache2/
 mkdir ssl
 cd ssl
 opennssl genrsa -des3 1024 > server.key
 > いったん適当に入れる
 mv server.key server.key.old
 openssl rsa -in server.key.old -out server.key
 openssl req -utf8 -new -key server.key -out server.csr
 openssl x509 -in server.csr -out server.pem -req -signkey server.key -days 3650
 chmod 400 ./server.*
 service apache2 restart
```

```
 apt-get install build-essential python-dev libnetfilter-queue-dev
 easy_install pip
 pip install NetfilterQueue

 iptables -I INPUT -p tcp --dport 443 -j NFQUEUE --queue-num 1
 # iptables -F
 # でフラッシュ
```

http://d.hatena.ne.jp/silphire/20081221/1229828624

http://netfilter.org/projects/libnetfilter_queue/doxygen/

をみること。
