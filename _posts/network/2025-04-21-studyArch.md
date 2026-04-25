---
layout: post
title:  "Arch: Cisco IOS XE &ASIC Architecture"
date:   2025-05-21T00:00:00+09:00
author: Akira Kanai
categories: network
cover:  "/assets/instacode.png"
---

# Arch: [Cisco IOS XE &ASIC Architecture](https://www.ciscolive.com/c/dam/r/ciscolive/emea/docs/2023/pdf/BRKARC-2092.pdf)

# Arch: 2020 [Nexus 9000 Architecture] (https://www.ciscolive.com/c/dam/r/ciscolive/emea/docs/2020/pdf/BRKDCN-3222.pdf)

Nexus 9k系についてのCiscoLive資料。第二世代と呼ばれる世代を対象としている。
すべてのNexusがこれらをサポートするわけでないことに注意。例えばNexus9364CなどはFTを持ち、SSXは持たない、など。

## Basic

Cloud Scale ASICはNexus 9000系で使われるもの。Broadcomなどのマーチャントシリコンではなく、カスタムシリコンとも呼んでいる。
カスタムシリコンを使うのはいくつかのCiscoらしい特徴を実現するためである。1:ACIポリシーベースの輻輳なぢのネットワーク 2:柔軟性のあるフォワーディングプレーンの実現 3:高速なトンネリング 4:暗号化技術 5:インテリジェントバッファ 6:ハードウェアテレメトリ

クラウドスケールASICはいくつかの種類を持つ。これらのASICはボックスごと、ラインカードごとの単位と思えば良い。各ASICは複数のスライスから構成されており、スライスごとはスライスインターコネクトによって接続されている。
スライスはそれぞれにおいてIngressとEgressのバッファなどを持つ。スライス間はany-anyで接続でき、十分な帯域を持っている(はず)。

SliceのForwardingの仕組みを述べる。
Ingressのフレームはパケットパーサによって処理され、宛先を決めるためにForwarding TableのLookupをTCAMとFlex Tiles(後述)で行う。また、Ingress ClassigicationをTCAMを使って行う。この結果、ロードバランス、AFD、DPPなどが必要なら適切なキューを選択し、Egress側に送る。
EgressのフレームはEgressスライスのBuffer/QueueによりSchedulingをされる。SPANなどが必要な場合、レプリケーションが行われる。次に、Egressのポリシ評価とぽ開けっとの書き換えが行われてポートに出ていく。

## FlexTiles and CAM

Flex Tilesは`Flexible Forwarding Tiles`とも呼ばれている。いわばルックアップテーブルのプールである。このテーブルの特徴はさまざまな目的に利用できる。MACエントリ、IPルーティング、IPマルチキャストテーブルなどである。これらはプロファイルとしてもたらており、`system routing template`でテンプレートを選べる。
いくつかの省略形が出てくるが、LPMとはLongest-prefix matchのことであり、HRTはホストルートテーブルである。

これらが分かれている理由として、適切なルックアップアルゴリズムが異なるからである。ß例えばIP経路表にはTrie木が好まれるし、ホストルートの場合hash関数が使われる。といった具合である。

VXLANの場合のASICの処理を具体的にみていこう。Encapsulation時、L2/L3のLookuptableが呼ばれたあと、TunnelのDestが解決され、Rewriteの機能でEncapされる。
Decapsulationの場合はTEPテーブルによってトンネルのはがしがLookupされたあと、 InnerのL2/L3のLookupが行われ、Rewriteさて宛先インタフェースに送信される。

ロードシェアリングする場合はECMPとDLPの２つがある。ECMPはStaticなフローベースのロードシェアリングである。一般的には5tupleのハッシュ、GREなどはそのキーフィールドを使ってハッシュを行う。このため、フローごとに固定的なハッシュとなる。DLB(Dynamic Load-Balancing)はACIで使用される。各リンクの混雑度を観測しながら動的な制御を行う。

マルチキャストの処理はホストテーブルを使うSingle Write & Multi Readの仕組みで実現される。マルチキャストのパケットはIngress側でMrouteの検索(HRTを用いる)した後、METと呼ばれるポインタを生成し、各EgressはそれをReadする。(これよりも深い説明はなし)

TCAMについて話す。特にここではACL, NAT, SPANなどのTCAMを指す。まず、TCAMは複数の目的にプロファイルで割り当てが可能である。それぞれ256のサイズが8-16個程度ある。プロファイルは`hardware access-list tcam region`で設定できる。

## FlowTable
クラウドASICを述べる上でまずいくつかのキーワードを述べる。
FT(Flow Table)は全てのフローをキャッチできるフローのテーブルでいくつかのメタデータを持つ。FTE(Flow Table Events)は特定の閾値によって発火するトリガである。SSX(Streaming Statistics Export)はユーザの設定によって発行されるストリーム型の統計情報である。

この特徴はフローの完全な情報を持っている。例えば、キューの情報、ドロップに関する情報、バーストの情報などでこれらはNetflowで用いられる。そしてこれはハードウェアでサポートされる。フローの処理の仕方はフロントパネル側のポートの直前で行われる。入ってきたパケットはTCAMのFilterにて抽出され、FlowTableで管理される。NetFlow Exportする場合はFlowはCPUにFlushされ、CPUはNetflowのメッセージを作り、mgmtポートから吐く。

FTEは特定のイベントの閾値を超えた時に発生するもので、ハードウェアで処理される。特徴は適切にスロットリングされたハードウェアへの直接的なExportがサポートされる。同様にフロントポート側に直接吐かれる。

SSXは定期的な統計値の送信を行う仕組みである。SSXのモジュールはASICからの各種カウンタ情報を定期的に取得し、CPUを介することなく統計値を送る仕組みである。

## QoS
(これについてはこのBlog上QoSをみること)

## Nexus Cloud Scaleシリーズ

EXシリーズはC93180-EX, 93108-EXなどがある。ACI/NX-OS両方をサポートし、Native25Gをサポートする。また、QoSのADFとDPPをサポートしている。
FXシリーズはMACSECおよび、FCをsupportする。また、RS-FECをサポートするため、25G(DAC?)の3m以上の伝送にも対応する。
Cシリーズは9364C, 9332Cに使われるもので、また異なった特性を持つ。FT, FTEはサポートしていないがSSXをサポートする。AFD/DPPは変わらずサポートされる。MACSecは一部のポートのみがサポートする。
FX2はC9336などがあり、FT, FTE, SSXすべてとMACSECをサポートする。LS3600FX2が使われる。
GはFX2の400G対応とも言えるもので、LS6400GXが搭載されている。

9700系のシャーシの説明はここでは割愛する。

MACSECについて触れる。MACSECはEtherレイヤでフレームを暗号化する技術で、SrcDstMACの直後のEtherTypeからPayloadを隠蔽する。スイッチの中間経路ではinportに入った後はSW内部ではdecapされて、outportで再度encapされる。

CloudSecとはVTEP間のMACSECを提供する。これは、VTEPでサポートされるもので、VXLANでEncapれたフレームのVXLAN部分、つまり、UDP以降のみをMACSec Encapして、通常のIP網でVXLANの中身を秘匿したまま通信することができる。
