---
layout: post
title:  "Debianインストールメモ"
date:   2025-04-21T00:00:00+09:00
author: Akira Kanai
categories: unix
cover:  "/assets/instacode.png"
---

## Debianインストールメモ

## はじめにやること

### ユーザアカウント

```
passwd
adduser kanai
...(Input)
Is the information correct? [Y/n] y
sudo visudo
```

以下のconfigのみとする

```
Defaults        env_reset
root    ALL=(ALL) ALL
kanai ALL=(ALL) ALL
```

### sshの鍵を登録する

```
mkdir /home/kanai
mkdir /home/kanai/.ssh
chmod 700 /home/kanai/.ssh
touch /home/kanai/.ssh/authorized_keys
chmod 400 /home/kanai/.ssh/authorized_keys
chown -R kanai:kanai /home/kanai
```

## DTI固有の環境の削除(dtiのときのみ)

```
apt-get purge ajaxterm
sudo vi /etc/ssh/sshd_config
-> Port 22に書き換える
-> PermitRootLogin no
sudo /etc/init.d/ssh restart
```

> **重要:** 現在のsshd sessionを保ったままほかのホストから入れるか確認

## vlanの捜査

```
modprobe 8021q
apt-get install vlan
vconfig add eth0 222    # 222 is vlan number

ifconfig eth0.222 up
ifconfig eth0.222 mtu 1496
ifconfig eth0.222 mtu 1504
ifconfig eth0.222 10.10.10.1 netmask 255.255.255.0
```

## interfaces

```
sudo vi /etc/network/interfaces


iface eth0 inet static
    address 10.5.10.78
    netmask 255.255.255.0
    network 10.5.10.0
    broadcast 10.5.10.255
    gateway 10.5.10.1
    dns-nameservers 8.8.8.8 127.0.0.1
iface eth0 inet6 static
    address 2001:db8::c0ca:1eaf
    netmask 64
    gateway 2001:db8::1ead:ed:beef
```



## iptablesの設定

### iptable scriptの生成

```
cat<<EOF>/etc/init.d/iptables
#!/bin/sh
iptables -P INPUT DROP
iptables -P OUTPUT ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -p tcp -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp --dport 3843 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p udp --sport 53 -d 0/0 --dport 1024: -m state --state ESTABLISHED -j ACCEPT
iptables -A INPUT -p tcp --dport 53 -j ACCEPT
iptables -A INPUT -p udp --dport 53 -j ACCEPT
iptables -A INPUT -p tcp --sport 53 -j ACCEPT
iptables -A INPUT -p udp --sport 53 -j ACCEPT
iptables -A INPUT -p icmp --icmp-type destination-unreachable -j ACCEPT
iptables -A INPUT -p icmp --icmp-type source-quench -j ACCEPT
iptables -A INPUT -p icmp --icmp-type time-exceeded -j ACCEPT
iptables -A INPUT -p icmp --icmp-type parameter-problem -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
EOF
```

### ufw

```
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 53/udp
```

### 動作確認

```
sudo apt-get install apache2 php5
ln -s /etc/apache2/mods-available/userdir.conf /etc/apache2/mods-enabled/userdir.conf
ln -s /etc/apache2/mods-available/userdir.load /etc/apache2/mods-enabled/userdir.load
ln -s /etc/apache2/mods-available/auth_digest.load /etc/apache2/mods-enabled/auth_digest.load
cp -p /etc/apache2/apache2.conf.dpkg-dist /etc/apache2/apache2.conf
cp -p /etc/apache2/envvars.dpkg-dist /etc/apache2/envvars
sudo /etc/init.d/apache2 restart
http://183.181.172.190/
が見えるか確認
cd ~; mkdir public_html; cd public_html; touch index.html
して
http://183.181.172.190/~kanai
が見えるか確認

sudo vi /etc/apache2/sites-enabled/000-default.conf
webrorrtを/home/kanai/public_htmlへ
```

### python wsgi

```
a2enmod wsgi
cat > /etc/apache2/sites-enabled/001-wsgi-test.conf<<EOF
WSGIDaemonProcess  user=nobody group=nogroup threads=10
WSGIScriptReloading On
WSGIScriptAlias /hoge /home/kanai/py.wsgi
EOF

vi /etc/apache2/sites-enabled/000-default.conf
>> <VirtualHost *:80>に以下のように書く
<Directory "/home/nttcom">
  AllowOverride All
  Require all granted
</Directory>

cat > /home/kanai/py.wsgi <<EOF
import sys, os
sys.path.append('/home/kanai')
from wsgitest import app as application
EOF
```


## NS(bind9)

```
sudo apt-get install bind9
ln -s /etc/bind /var/namedb
cd /etc/bind/
rm named.conf.default-zones named.conf.local named.conf.options
vi named.conf
named-checkconf
service bind9 restart
```

### named.conf sample

```
options {
       directory "/var/cache/bind";
       auth-nxdomain no;    # conform to RFC1035
       listen-on-v6 { any; };
};


zone "." {
       type hint;
       file "/etc/bind/db.root";
};
zone "localhost" {
       type master;
       file "/etc/bind/db.local";
};

zone "127.in-addr.arpa" {
       type master;
       file "/etc/bind/db.127";
};

zone "0.in-addr.arpa" {
       type master;
       file "/etc/bind/db.0";
};

zone "255.in-addr.arpa" {
       type master;
       file "/etc/bind/db.255";
};
acl "trust-network" {
 localhost;
 ::1;
 116.197.140.178;
};

zone "hogetan.net" {
 type master;
 file "/etc/bind/zone.hogetan.net";
};
```

## sphinx

```
sudo apt-get install texlive-latex-base
```

## ntp

```
sudo apt-get install ntp
sudo vi /etc/ntp.conf
```

```
server  ntp1.jst.mfeed.ad.jp
server  ntp2.jst.mfeed.ad.jp
server  ntp3.jst.mfeed.ad.jp
fudge   127.127.1.0 stratum 10
driftfile /var/lib/ntp/ntp.drift
logfile /var/log/ntpd.log
authenticate no
# default deny all
restrict default ignore
restrict 45.0.0.0 mask 255.255.0.0 noquery nomodify nopeer notrust notrap
restrict 172.16.0.0 mask 255.255.0.0 noquery nomodify nopeer notrust notrap
restrict 210.173.160.27 noquery nomodify
restrict 210.173.160.57 noquery nomodify
restrict 210.173.160.87 noquery nomodify
restrict 127.0.0.1
```

```
sudo touch /var/lib/ntp/drift
sudo  chown ntp:ntp /var/lib/ntp/drift
sudo service ntp restart
sudo ntpq -p
-> 少し待ちます(reachが377になるまで)
```

## syslog-ng

```
sudo aptitude install syslog-ng
vi /etc/syslog-ng/syslog-ng.conf
internal()のあとにudp追加。
source s_src { unix-dgram("/dev/log"); internal(); udp();
            file("/proc/kmsg" program_override("kernel"));
};

filter f_host_router  { netmask(192.168.100.254/32); };
destination homelog { file("/var/log/homelog" perm(0644)); };
log { source(s_src);   filter(f_host_router); destination(homelog); };

filter f_local1 { facility(local2) ; };
destination l2l3log { file("/var/log/l2l3" perm(0644)); };
log { source(s_src);   filter(f_local1); destination(l2l3log); };

sudo service syslog-ng restart
logger -h 127.0.0.1 -p local1.debug hoge
```

## python

```
# これなにようだっけ？
sudo apt-get install libatlas3gf-base f2c
sudo pip install tweepy
sudo apt-get install python-pip python-setuptools \
  python-dev build-essential libfreetype6-dev libpng-dev python-virtualenv \
  gfortran libblas-dev liblapack-dev g++ tk-dev \
  python-numpy libhdf5-serial-dev
sudo pip install PyYAML
sudo pip install numpy
 -> とおらない
sudo pip install scipy
sudo pip install SymPy netCDF4 nose PIL  matplotlib nltk
sudo easy_install -U distribute
sudo pip install nltk
```

* python + emacs

```
sudo apt-get install python-mode
```

## VLANconfigの基本

```
cat <<EOF>> /etc/sysconfig/network
VLAN=yes
VLAN_NAME_TYPE=VLAN_PLUS_VID_NO_PAD
NETWORKING_IPV6=yes
NOZEROCONF=yes
EOF
```

## T400の設定

```
apt-get install firmware-iwlwifi
apt-get install wicd-cli
apt-get install iw
iwconfig wlan0 mode Managed
iwconfig wlan0 essid beefbeef-home-air
iwconfig wlan0 key bc1
iwlist  wlan0 scanning
wpa_passphrase beefbeef-home-air <password> >> /etc/wpa_supplicant.conf
wpa_supplicant -i wlan0 -c /etc/wpa_supplicant.conf
```

## ブリッジにする

```
apt-get install bridge-utils
```

## int

```
/etc/network/interfaces

auto lo
 iface lo inet loopback

auto eth0.100
iface eth0.100 inet dhcp

auto eth0.500
iface eth0.302 inet static
 address 192.168.5.254
 netmask 255.255.255.0
```

## dhcpd

```
apt-get install isc-dhcp-server
vi /etc/dhcp/dhcpd.conf
/etc/init.d/isc-dhcp-server restart
```

## router化

```
net.ipv4.tcp_syncookies = 1
net.ipv4.ip_forward = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
iptables -t nat -A POSTROUTING -o eth0.100  -j MASQUERADE
```

## bind cache

```
apt-get install bind9
```

## gmailをsmtpサーバとして活用する

relayの設定：このホストを家庭ネットワークのrelayサーバとする場合、mynetworksに追加する

```
sudo vi  /etc/postfix/main.cf
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 192.168.0.0/16
とか。
```

gmailへのSSLトンネル確立

```
sudo apt-get install stunnel
cd /etc/ssl/certs
openssl req -new -x509 -nodes -days 365 -out stunnel.pem -keyout stunnel.pem
chmod 600 stunnel.pem
dd if=/dev/urandom of=temp_file count=2
openssl dhparam -rand temp_file 512 >> stunnel.pem
ln -sf stunnel.pem `openssl x509 -noout -hash < stunnel.pem`.0

debug用コマンド: smtp.gmail.comにアクセスできるかは以下のコマンドで確認
openssl s_client -host smtp.gmail.com -port 465

sudo vi /etc/stunnel/stunnel.conf
; clientを書き換える
client = yes
; Service-level configuration の下を以下だけにする
; 127.0.0.1をlocalhostにするとv6 onlyでlistenする..
[gmailsmtp]
accept  = 127.0.0.1:8465
connect = smtp.gmail.com:465

sudo vi /etc/default/stunnel4
ENABLE=1

sudo service stunnel4 restart
```

次に、postfix側でrelayの設定

```
plain認証のため(postfixの)
sudo apt-get install cyrus-sasl2-dbg
sudo vi  /etc/postfix/main.cf
relayhost = [localhost]:8465
smtp_sasl_auth_enable  = yes
smtp_sasl_password_maps = hash:/etc/postfix/isp_passwd
smtp_sasl_security_options = noanonymous
smtp_sasl_mechanism_filter = cram-md5,digest-md5,plain,login

sudo vi /etc/postfix/isp_passwd
[localhost]:8465 <user>:password> < ここはgmailのアプリケーションパスワードを入れる！(スペースは抜こう

sudo chmod 400 /etc/postfix/isp_passwd
sudo postmap /etc/postfix/isp_passwd
sudo service postfix restart
```

## tftpd

```
# もし入っているなら消す
sudo apt-get remove tftpd
sudo apt-get install tftpd-hpa
sudo vi /etc/default/tftpd-hpa
# ここはよしなに変える
TFTP_DIRECTORY="/tftpboot"
# --createをいれるとファイルが新規に作れる
TFTP_OPTIONS="--secure --create"
# /tftpbootつくって所有者の変更
sudo mkdir /tftpboot/
sudo chown -R tftp /tftpboot/
sudo service tftpd-hpa restart
# 詳細は以下
# https://help.ubuntu.com/community/TFTP
```
