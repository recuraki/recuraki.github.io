---
layout: post
title:  "VXLAN/EVPNメモ"
date:   2025-05-21T00:00:00+09:00
author: Akira Kanai
categories: network
cover:  "/assets/instacode.png"
---

# VXLAN/EVPNメモ
[Cisco IOS XE Bengaluru 17.6.x（Catalyst 9300 スイッチ）BGP EVPN VXLAN コンフィギュレーションガイド](https://www.cisco.com/c/ja_jp/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/vxlan/b_176_bgp_evpn_vxlan_9300_cg/bgp_evpn_vxlan_overview.html)

VXLANとはMAC-in-UDPであり、VLAN, STPの問題に対して出現した技術である。一方でカプセリングだけが行えればいいというわけではない。
- VXLANはそのend-endでIP接続性を持つ必要がある
- VXLAN上のフラッディングを扱う必要がある
- VXLAN上の特定のL2/L3接続性を流通させる必要がある

EVPNとはL2/L3の接続性を伝搬させるためのMP-BGPで用いられるプロトコルである。VXLANと組み合わせた状態をBGP EVPN VXLANファブリックと称する。

VXLANを用いることで以下の利点がある。
- VLAN-ID 10bitの壁 -> 1600万テナントを実現する
- 柔軟性が向上する。延伸が楽。
- モビリティが工場ある。あるVNIを伝搬させれば同じネットワークに接続できる。

## VXLANオーバレイ
VXLANにより実現される仮想ネットワークのことである。これにより作られるそれぞれをVXLANセグメントと呼ぶ。

`VXLANネットワーク接続識別子(VNI)`は各ネットワークを区別するためのものである。

`VTEP(仮想トンネルエンドポイント)`は「すべてのVXLANセグメントに」存在する。そして、フレームをencap/decapする。VTEPとはL2-L3変換と言い換えられる。セグメント側のインタフェースはL2のブリッジであり、適切な相手に対してL3(IP)でカプセリングして転送する。さらにVTEPは他のVTEPの持つリモートの情報を知っており、適切なVTEPに対してカプセリングパケットを転送する必要がある。

`オーバレイマルチキャスト`はセグメント側のマルチキャストを転送する仕組みである。TRM(テナント ルーテッド マルチキャスト)という仕組みがあり、これはVXLANオーバレイがマルチキャストをサポートする仕組みである。このサポートがない場合、マルチキャストフレームはセグメント内のブロードキャストとして扱うので、セグメントを跨いだノードは通信ができません。

`アンダーレイ`はカプセル化されたフレームを転送するIP網のことです。

`EVPN C-plane`はVXLANオーバレイネットワーク上においてどのノードがどのVTEPの下にいるかを交換するプレーンである。また、戻りのトラフィックに関する工夫もあり、外側から自分のVXLANセグメント上に送られた通信はSrc-Dstを記録しておき、戻りのパケットがVXLANセグメント側から着信した際、高速にカプセリングする工夫がされている。
さて、フラッディングをSrc側で行ってこの仕組みを愚直に実装すると拡張性の限界を迎える。このために使われるのがEVPN(Ethernet VPN)であり、MACアドレス及びIPアドレスの情報を交換する。これはMP-BGP上で行われる。

## EVPNルートに含まれる情報
- Route Target

- EVPN ルートタイプ
- EVPN インスタンス
- イーサネットセグメント
- EVPN マルチホーミング


# [EVPN VXLAN レイヤ 2 オーバーレイネットワークの設定](https://www.cisco.com/c/ja_jp/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/vxlan/b_176_bgp_evpn_vxlan_9300_cg/configuring_evpn_vxlan_layer_2_overlay_network.html)

Layer2 オーバレイネットワークとは異なるセグメントにいるノード間がイーサネットでフラットに通信をできることである。L2ネットワークをVNIを介して通信できることである。

L2ネットワークで注目するのはBUMトラフィックであり、これをどのように扱う（転送する）かが論点となる。
`BUMをアンダーレイマルチキャスト`で扱う方式はL2 VNIをマルチキャストにmapする。これによって、アンダーレイのマルチキャストにパケットの複製を任せることができる。各VNIを持つVTEPはそのmulticastグループにjoinすることで自身の持つVNI宛てのBUMを受け取ることが可能になり効率的な転送を実現する。

入力を`レプリケーションする方法ヘッドエンドレプリケーション`はその名の通り、入力側のデバイスがroute-type 3を元にパケットをユニキャストとして複製して転送する手法である。アンダー例にマルチキャストルーティングを必要としません。ただし、この方法は拡張性に問題があります。このため、BUMのレートリミットを設定することもできます。

## フラッディングに対する工夫
さて、フラッディングの主な動機はARP(IPv6ならNS)です。EVPNではMACアドレスとIPアドレスを個別に扱うのではなくてIP-MACのバインディング(ARPテーブルのようなものです)を配布することができます。これによりエンドポイントはブロードキャストで入力されたARPやNSをユニキャストに変換して必要なデバイスのみに転送を行います。
