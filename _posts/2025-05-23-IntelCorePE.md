---
layout: post
title:  "Intel Pコア, Eコア"
date:   2025-05-23T00:00:00+09:00
author: Akira Kanai
categories: cpu
tags: cpu
cover:  "/assets/instacode.png"
---


[北森瓦版 “Panther Lake”の動作デモを行う―市場流通は2026年初め](https://northwood.blog.fc2.com/blog-entry-12741.html)を読んでいた。

> おそらくは4 P-core + 8 E-core + 4 LP E-coreの構成のものだろう。“Lunar Lake”や“Arrow Lake”と同様に“Panther Lake”もHyper Threading technologyは無効となっている。キャッシュ構成はL1 1.6MB / L2 24MB / L3 18MBである。うちL2はP-core L2 3MB×4＋E-core L2 4MB×2＋LP E-core 4MBと推定される。L3 cacheはP-core 3MB×4＋E-core 3MB×2だろうか。さらにこの他にSLC cacheが存在するという情報もある。

Pコア、Eコア、LP E-Core ? HTとの関係は？

[Intel、次期CPU「Alder Lake」に搭載される新コアの詳細を発表](https://pc.watch.impress.co.jp/docs/news/1344892.html)を見ると、これらのコアの種別はAlderLake(SkyLake後継)から出てきたもの。

AlderLake世代を例にとれば
- 電力効率が高い「E: 高効率コア」がGracemontであり、`mont`というAtom系の名前から来ているもの。
- 性能が高い「P: 高性能コア」が「Golden Cove」。Ice Lake世代にはコアはSunny Coveという名前だったらしく、その名前から来ている。

Gracemontに関しては`5,000エントリーの分岐ターゲットキャッシュを備え、より正確な分岐予測`ということでこの5,000エントリをどう読むというのはあるが結構投機実行をするのだなぁ。と感じた。

Golden Coveについて、最近のCPUは`AMX(Advanced Matrix Extensions)`と呼ばれる行列の演算命令があるらしい。`INT8なら2,000、Bfloat16であれば1,000の処理を1クロックサイクル`ということでそこまで大きな並列性ではないし、おそらくメインメモリとの通信が必要になるのだろうか(キャッシュ内で完結できるのかもしれないが)？

とはいえ、このようなP,Eコアという種類が異なるものがある。

https://pc.watch.impress.co.jp/docs/news/1532473.html
この上の方の表を見るのが分かりやすそう。

[なぜMeteor Lakeでは2種類のEコアがあるのか](https://pc.watch.impress.co.jp/docs/news/1532473.html)を見ると、
そして、
> コンピュートタイルではなく、SoCタイルと呼ばれるSoCの中核チップにも、Intelが低電力Eコア(LP E Core)と呼んでいる、デュアルコアのEコアが用意されていることだ。

らしい。難しい。コンピュート側ではなく、SoCのタイルにいる？

https://image.itmedia.co.jp/l/im/pcuser/articles/2309/20/l_si7101-Intel-11.jpg

記事を読んでいくと、特にSoC(特殊用途)向けのコアというわけではなく、汎用である。読んだ限りでは、

- 最近のコアってハイスペック過ぎるから、OSの定常的な軽いバックグラウンドプロセスくらいはこっちで動かせば良いよね。

って話だと理解した。

さて、そこで、[Lunar LakeではPコアのハイパースレッディングを廃止　インテル CPUロードマップ](https://ascii.jp/elem/000/004/207/4207185/2/)に何故繋がるか？に書いてありそう。今日はここまで。

> そしてSMTを無効化することで、消費電力も若干減る(扱うスレッドが1つで済むから、スレッド間の調停も不要だし、SMTをサポートするための回路もなくなる分、そこで費やしていた消費電力も削減できる)。こういう判断に基づき、Lunar Lakeではハイパースレッディングが廃止された。
