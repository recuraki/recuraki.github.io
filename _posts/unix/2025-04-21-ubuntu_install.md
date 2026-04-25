---
layout: post
title:  "ユーザアカウント"
date:   2025-04-21T00:00:00+09:00
author: Akira Kanai
categories: unix
cover:  "/assets/instacode.png"
---

Ubuntu インストール(sakura)
===========================

# ユーザアカウント
## regist
```
$ passwd
$ adduser kanai
 ...(Input)
 Is the information correct? [Y/n] y
$ sudo visudo
 適当に選ぶ
 kanai ALL=(ALL) ALL
 kanai ALL=NOPASSWD: ALL
```

## sshの鍵を登録する

```Shell
su - kanai
mkdir /home/kanai
mkdir /home/kanai/.ssh
chmod 700 /home/kanai/.ssh
touch /home/kanai/.ssh/authorized_keys
chmod 400 /home/kanai/.ssh/authorized_keys
chown -R kanai:kanai /home/kanai
```

# ufw
```
systemctl status ufw
 -> active
ufw disable
sudo ufw status
 -> inactive
sudo ufw status verbose
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 53/udp
sudo ufw default deny
sudo ufw enable
sudo ufw reload
sudo ufw status verbose
```


動作確認
------------------

```
sudo apt-get install apache2
sudo ln -s /etc/apache2/mods-available/userdir.conf /etc/apache2/mods-enabled/userdir.conf
sudo ln -s /etc/apache2/mods-available/userdir.load /etc/apache2/mods-enabled/userdir.load
sudo ln -s /etc/apache2/mods-available/auth_digest.load /etc/apache2/mods-enabled/auth_digest.load
sudo systemctl reload apache2
cd ~; mkdir public_html; cd public_html; touch index.html
sudo journalctl -u apache2

sudo vi /etc/apache2/sites-enabled/000-default.conf
```
DocumentRoot /home/kanai/public_htmlへ



# NS(bind9)
```
sudo apt-get install bind9
sudo ln -s /etc/bind /var/namedb
cd /etc/bind/
rm named.conf.default-zones named.conf.local named.conf.options
vi named.conf
named-checkconf
service bind9 restart
```

```
options {
        directory "/var/cache/bind";
        allow-transfer { trust-network; };
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
 127.0.0.1;
 ::1;
};

zone "hogetan.net" {
 type master;
 file "/etc/bind/zone.hogetan.net";
};
```

> vi zone.hogetan.net

```
$TTL 86400
@               IN      SOA     os3-384-25246.vs.sakura.ne.jp. root.os3-384-25246.vs.sakura.ne.jp.(
                                8      ; Serial
                                180            ; Refresh(1h)
                                900             ; Retry(15min)
                                3600000         ; Expire(1000h)
                                86400           ; Minimum(24h)
                                )
                IN      NS      os3-384-25246.vs.sakura.ne.jp.
                IN      A       133.167.108.250
www             IN      A       133.167.108.250
```


sphinx
=====================

> sudo apt-get install texlive-latex-base


ntp
===============

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

syslog-ng
==========================

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

python
=============================

```
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


VLANconfigの基本
=========================

```
 cat <<EOF>> /etc/sysconfig/network
 VLAN=yes
 VLAN_NAME_TYPE=VLAN_PLUS_VID_NO_PAD
 NETWORKING_IPV6=yes
 NOZEROCONF=yes
 EOF
```

T400の設定
=======================

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

ブリッジにする
========================

> apt-get install bridge-utils


router化
=============================

```
 net.ipv4.tcp_syncookies = 1
 net.ipv4.ip_forward = 1
 net.ipv4.icmp_echo_ignore_broadcasts = 1
 net.ipv4.icmp_ignore_bogus_error_responses = 1
 iptables -t nat -A POSTROUTING -o eth0.100  -j MASQUERADE
```


gmailをsmtpサーバとして活用する
==============================================

relayの設定：このホストを家庭ネットワークのrelayサーバとする場合、mynetworksに追加する

```
 sudo vi  /etc/postfix/main.cf
 mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 192.168.0.0/16
```
 とか。

gmailへのSSLトンネル確立

```
 sudo apt-get install stunnel
 cd /etc/ssl/certs
 openssl req -new -x509 -nodes -days 365 -out stunnel.pem -keyout stunnel.pem
 chmod 600 stunnel.pem
 dd if=/dev/urandom of=temp_file count=2
 openssl dhparam -rand temp_file 512 >> stunnel.pem
 ln -sf stunnel.pem `openssl x509 -noout -hash < stunnel.pem`.0
```

```
debug用コマンド: smtp.gmail.comにアクセスできるかは以下のコマンドで確認
```

```
 openssl s_client -host smtp.gmail.com -port 465
```

```
sudo vi /etc/stunnel/stunnel.conf
```
```
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

tftpd
=============

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
 https://help.ubuntu.com/community/TFTP
```
