---
layout: post
title:  "PowerShell"
date:   2025-04-21T00:00:00+09:00
author: Akira Kanai
categories: memo
cover:  "/assets/instacode.png"
---

# PowerShell
Contents:



## ログの取得方法


> Get-EventLog system -after "2017/08/08" -before "2017/08/10" -EntryType Information -Message "*システム*"
