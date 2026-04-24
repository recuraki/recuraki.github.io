---
layout: post
title:  "NVIDIA: Mellanox NICのFirmを追いかけるページ"
date:   2025-04-21T00:00:00+09:00
author: Akira Kanai
categories: aiml
cover:  "/assets/instacode.png"
---

# NVIDIA: Mellanox NICのFirmを追いかけるページ

# [DOCA Documentation v2.9.0_CX8](https://docs.nvidia.com/doca/archive/2-9-0-cx8/changes+and+new+features/index.html)

まず[NVIDIA DOCA Release Notes](https://docs.nvidia.com/doca/archive/2-9-0-cx8/nvidia+doca+release+notes/index.html)を見ると`DOCA 2.9.0_CX8 is a dedicated branch supporting ConnectX-8 devices only.`という表記があり、このFirmはConnectX-8専用である。つまり、このRelease Notesを読むとCX8の気持ちが分かるかもしれない。

[Bug Fixes](https://docs.nvidia.com/doca/archive/2-9-0-cx8/bug+fixes+in+this+version/index.html)はない。[Known Issues](https://docs.nvidia.com/doca/archive/2-9-0-cx8/known+issues/index.html)は2.9.0既存のものであるだろう。この辺りは[2.9.0の通常版](https://docs.nvidia.com/doca/archive/2-9-0/nvidia+doca+release+notes/index.html)を見れば分かるだろう。

2.9.0_CX8のNew Featuresを読むにあたり、[2.9.0通常版](https://docs.nvidia.com/doca/archive/2-9-0/changes+and+new+features/index.html)のFeaturesを一瞥する。すると、[2.9.0 CX8版](https://docs.nvidia.com/doca/archive/2-9-0-cx8/changes+and+new+features/index.html)と明らかに分量が違う。このことからCX8版はCX8対応固有の変更点であると期待できる。読む。

- Added support for NVIDIA® ConnectX®-8 SuperNIC: はこのReleaseの主たる目的であろう。ここで、従来はBF3の専売であった`SuperNIC`の名称がCX8に使われていることに注目する。

- ConnectX-8 specific features: 以下は固有の機能の説明。

- Support for multiple host pairs

- DPA programmability: [DPA](https://docs.nvidia.com/doca/sdk/dpa+subsystem/index.html)はdata-path acceleratorであり、BF3の場合は特定パケットに対してDPUが有する機能を`BlueField DPU に存在するすべてのアクセラレータ (暗号化/復号化、ハッシュ計算、圧縮/解凍など) `で処理することができる。CX8のアクセラレータすべてがBF3 SuperNICと同等であるのかは確認が必要。

- Add hop count to Prog CC RTT event: [PCC](https://docs.nvidia.com/doca/sdk/nvidia+doca+pcc+application+guide/index.html#src-3168394682_id-.NVIDIADOCAPCCApplicationGuidev2.9.0LTS-Introduction)って書いてほしい。PCCはProgrammable Congestion Control である。輻輳発生時のコントロールをユーザが独自に書くことができる。ワークロードに最適化された輻輳制御が実装できるらしい。ここでの輻輳とはECNのCNP受信時のことであり、輻輳制御におけるnotification point (NP)を自分で実装できる。

- Support SD using 2xGen5 PCIe to reach 800Gb/s line rate: 唯一性の低い略語はやめてほしいが文脈的に[Socket Direct](https://www.advancedhpc.com/pages/nvidia-socket-direct-adapters)だろう。違ったらごめんなさい。Socket Directは特にマルチCPU環境において1つのNICで複数のCPUソケットに足を持てるメカニズム。ここで言われている`2xGen5`はGen5 2 lanesではなくて、`2 x PCIe 5.0 x 16 lanes`だろう。PHY 800Gという表記にグラッとくるが、このユースケースにおいては`2 x 400G`を2CPU Socketでそれぞれ使うのが想定されているんじゃないだろうか。

- Indicate RX port utilization in RTT response packet 直訳するなら`RTT応答によりポートのRx側のUtilを示す`何ですがよく分かりません。だれか教えて。ある機能を示したいのか、汎用的にIBパケットなどにマーカーを打てるのか、単にstatのenhanceなのか分からない。

- Expose power, thermal, and event counters: よく分からないが、` /sys/class/infiniband/mlx5_0/ports/1/hw_counters/`系のstatが増えたのかな？

- Supported x86 architecture only: これはArmはサポートしていない、と言いたいのかな？

(以上。特定顧客向けのreleaseかなぁ。という感想。)

# [ZTR-RTT Congestion Control Algorithm Overview v1.0](https://docs.nvidia.com/networking/display/ztrrttcongestioncontrolalgorithmoverviewv10)
Zero Touch RoCE(ZTR)に関連した機能。

まず、[Overview](https://docs.nvidia.com/networking/display/ztrrttcongestioncontrolalgorithmoverviewv10)を読もう。ZTRはRoCEv2対応のスイッチでなくても数千台規模のクラスタでRoCEと同等の性能を出すための技術。ZTRのキーの1つはZTRラウンドトリップ制御(ZTR-RTT CC)アルゴリズム。これを話す。

ZTR-RTT CCはネットワークRTTをエンド-エンドでアクティブに監視して、ドロップ前の輻輳に対して能動的に対応する。この実装のポイントは以下である。

- HWすなわちDPA上の実装(BFやCX8など、と読んでいる)
- RTTベースのCC制御
- "Current default CC algorithm for RoCE "..とあるが、これが輻輳制御する量・戻す量がDefaultと同じ、なのかは不明。
- HPC/AIではDCQCN`よりもよい`性能
- StorageではDCDQN`と同じ`性能

次に[CC](https://docs.nvidia.com/networking/display/ztrrttcongestioncontrolalgorithmoverviewv10/congestion+control)を読もう。まずは、文章があまりにも足りないので想像で補う。

LineBlockingの図は`Oa`側が`S2`で多量の入力を受けており、その結果`S1-S2`間のinポート(queue)が溢れている部分を埋めてしまい、`V-Ov`間の通信を圧迫してしまうことを示している（はず）。

`Datacenter Congestion Control Challenges`では(行間を読むと)現在のDCQCNの課題が書かれている。

- 100G/400G NWで生じるusオーダの遅延。ただし、瞬時の発生ですぐに収まる。
- 複雑化するトラフィックパターン。また、トポロジやアプリケーション。すべてに対応することは困難。CC側も新しい輻輳通知で工夫はしている。
- HW側の実装がRobustでない。(この意図するところは不明)。
- SW側の反応(react)は遅すぎる。

そこで、ZTR-RTT CCは以下のようなことを提供する。

- CCアルゴリズムの実装のためのプラットフォーム。おそらく、`Prog CC RTT`のことだろう。
- DPA上のプロセッサで(CCの)コードが実装されること。ホストまで上がらない。
- CNPとRTTの計測が含まれる。おそらくはタイムスタンプが入っているということか？
- HWによりRate制御とデータ収集を行う。ソフトウェアでないということだろう。
- 宛先ごとにステートフル。E-Eで制御を行うということだろう。
- CCにかかる時間は1us-1.5usオーダ。

次のRTT要求のシーケンスは目を滑らしそうになるが、`CCアルゴリズムが要求`つまり、ユーザアプリケーションがRTT要求をできることに注意。この後のフローは一般的か。`A-Z`のRTT測定を考えると、1.データが揃ったらA側のTx Queueに入れるときにRTTを打刻する。2.Z側に到着したらRx Queueに入った時に打刻する。 3.さらに`Z側からのの返信時刻も打刻して返信する。 `これにより、RTTと書かれているが`A->Z`と`Z->A`の転送時間をしることができる。

一番大切なこの検知アルゴリズムを見ていく。`A(RP)-Z(NP)`において、輻輳の発生はRTTとCNPだ。レート調整は俗に言うAIMD(additive-increase and multiplicative-decrease, CCでよく使うので参照せよ)。RTTが変更されたときに擬似的なウィンドウサイズを調整(mimic)して制御を行う。

## [NVIDIA ConnectX-7 Adapter Cards Firmware Release Notes v28.39.2048 LTS](https://docs.nvidia.com/networking/display/connectx7firmwarev28392048lts)

Function
- ページが閲覧できない。たぶんなし。

