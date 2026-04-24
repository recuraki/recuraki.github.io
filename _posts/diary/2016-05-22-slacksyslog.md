---
layout: post
title:  "20160522日記: fluentd + slackでsshのログイン状態を監視"
date:   2016-05-22T00:00:00+09:00
author: Akira Kanai
categories: diary
cover:  "/assets/instacode.png"
---

# 20160522日記: fluentd + slackでsshのログイン状態を監視

いまどき、chatopsが叫ばれているが、chatからの命令を受け取るだけではなく、
chatに状態を表示したい。
chatは即時通達の手段であって、目的ではない。同時にfluentd + kibanaでログ解析ができるようにする。

## syslog_ngのインストール

syslogを使う以上、syslogdは必要となる。
今回はauth_logを用いるので必然性はないが、今後のために入れておく。
これを行うことで、hostベースやファシリティベースでのログファイルの分離が
可能になり、以後の解析や連携に役立つ。

```
 sudo aptitude install syslog-ng
 sudo vi /etc/syslog-ng/syslog-ng.conf
 > 以下のようにinternal()のあとにudp追加。
 source s_src { unix-dgram("/dev/log"); internal(); udp();
             file("/proc/kmsg" program_override("kernel"));
 };
```

次に、Configの読み込みを行う。
```
 sudo service syslog-ng restart
 logger -n 127.0.0.1 -p local1.debug hoge
 tail /var/log/debug
```
として、メッセージが残っていればOK

## Javaのインストール

Fluentdなどに必要なのでいれる。
```
 sudo apt-get install ruby software-properties-common
 sudo add-apt-repository -y ppa:webupd8team/java
 sudo apt-get update
 sudo apt-get -y install oracle-java8-installer
```

## パッケージのインストール 5.0
```
 wget -qO - https://packages.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
 echo "deb https://packages.elastic.co/elasticsearch/5.x/debian stable main" | sudo tee -a /etc/apt/sources.list.d/elasticsearch-5.x.list
 sudo apt-get -y update && sudo apt-get -y install elasticsearch
 sudo update-rc.d elasticsearch defaults 95 10
 sudo service elasticsearch start
 # kibana
 cd /tmp
 wget https://download.elastic.co/kibana/kibana/kibana-5.0.0-alpha4-amd64.deb
 dpkg -i kibana-5.0.0-alpha4-amd64.deb
 sudo update-rc.d kibana defaults 95 10
 sudo /etc/init.d/kibana restart
 # logstash
 wget https://download.elastic.co/logstash/logstash/packages/debian/logstash-5.0.0-alpha4.deb
dpkg -i logstash-5.0.0-alpha4.deb
```

### パッケージのインストール
```
 # ElasticSearchのrepoとか
 wget -qO - https://packages.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
 echo "deb https://packages.elastic.co/elasticsearch/2.x/debian stable main" | sudo tee -a /etc/apt/sources.list.d/elasticsearch-2.x.list
 sudo apt-get -y update && sudo apt-get -y install elasticsearch
 # td-agentいれる
 curl -L https://td-toolbelt.herokuapp.com/sh/install-ubuntu-trusty-td-agent2.sh | sh
 sudo /opt/td-agent/embedded/bin/fluent-gem install fluent-plugin-elasticsearch
 # kibanaいれる
 echo "deb http://packages.elastic.co/kibana/4.5/debian stable main" | sudo tee -a /etc/apt/sources.list
 sudo apt-get update && sudo apt-get install kibana
 sudo update-rc.d kibana defaults 95 10
 sudo update-rc.d elasticsearch defaults 95 10
 sudo service elasticsearch start
 sudo  /opt/td-agent/embedded/bin/fluent-gem install fluent-plugin-slack
 sudo  /opt/td-agent/embedded/bin/fluent-gem install fluent-plugin-grep
 sudo  /opt/td-agent/embedded/bin/fluent-gem install fluent-plugin-copy
 sudo /etc/init.d/kibana restart
 # http://192.168.78.2:5601/　などにアクセスしてkibanaが見えることを確認
```

## SSHのログ用のConfig
```
 sudo vi /etc/td-agent/td-agent.conf
```

で以下のように書く
```
 sudo chmod 777 /var/log/auth.log
 sudo /etc/init.d/td-agent  configtest
 sudo /etc/init.d/td-agent  restart
 # sudo usermod -G adm,td-agent td-agent
```

## Fluentdのテスト(試験用)
```
 sudo apt-get install python-pip
 pip install fluent-logger
```

```
 from fluent import sender
 from fluent import event
 sender.setup('slack.test', host='localhost', port=24224)
 event.Event('follow', {
     'message': 'hogehoge',
     'to': 'userB'
     })
```
