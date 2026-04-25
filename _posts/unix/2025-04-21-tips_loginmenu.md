---
layout: post
title:  "シェルを与えたくないユーザのパスワード変更"
date:   2025-04-21T00:00:00+09:00
author: Akira Kanai
categories: unix
cover:  "/assets/instacode.png"
---

## シェルを与えたくないユーザのパスワード変更


APOPのパスワードみたいに、ログインパスワードと一致しないパスワード変更を、
SSHでさせたいけど、シェルは与えたくない場合。
以下みたいなスクリプトをlogin-shellにする。

```
 #!/usr/local/bin/bash

 flagend=1

 if [ ! -x /usr/bin/passwd ]; then
 echo "System error"
 flagend=0
 fi

 if [ ! -x /usr/local/sbin/popauth ]; then
 echo "System error"
 flagend=0
 fi

 while [ $flagend = "1" ] ; do

 echo "[1]: Change your login password"
 echo "[2]: Change your APOP password"
 echo "[0]: Do Nothing"

 read input

 if [ $input = 1 ]; then
 exec /usr/bin/passwd
 flagend=0
 elif [ $input = 2 ]; then
 exec /usr/local/sbin/popauth
 flagend=0
 elif [ $input = 0 ]; then
 flagend=0
 fi

 done
```
