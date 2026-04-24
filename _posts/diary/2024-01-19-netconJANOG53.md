---
layout: post
title:  "20240119_日記_JANOG53 NETCONアプローチ"
date:   2024-01-19T00:00:00+09:00
author: Akira Kanai
categories: diary
cover:  "/assets/instacode.png"
---

# 20240119_日記_JANOG53 NETCONアプローチ
手元でJANOG53 NETCONの全問題の考察メモを書いていたのでどのようにアプローチしたか紹介します。

## チュートリアル1: OSPF auth-key誤り
正常性の確認: `sh ip os int`と`sh ip os nei`
configのdiffを取るとauthentication-keyが誤っていました。keyを合わせて`clear ip ospf process 1`。

## チュートリアル2: 
正常性の確認: `sh ip route`, `sh bgp sum`で経路が足りておらずrecv 0なことに気がつきます。`show ip bgp`に経路は存在するのでadvされていないことがわかります。`sh bgp sum`の上部に警告が出ているのでIOS-XRなのにroute-mapが存在しないことに注意して`permit any any`のroute-mapを書きました。
in route-mapが効くか不安だったのでおまじないとして`clear bgp`でhard resetしました。

## チュートリアル3:
`sh ip route`, `sh bgp sum`でneighborが上がっていないので`sh log`するとwrong ASなので設定を眺めて`remote-as`を直しました。
対象ルータで`sh bgp neigh recv`すると経路は聞こえているのに`sh ip route`に載っていないのは変なので`show bgp detail`のようなものを叩きます。next-hopが`sh ip route`に載っていないのでダメなのでeBGPルータを見るとnext-hopがないので足します。

## チュートリアル4:
`sh ip route`, `sh ip route`, `sh ip int bri`。
問題文をメタ読みするとIPアドレスミスかVRFミスが疑われるのでトポロジ図をじっと比較します。admin downなポートがあるので`no shut`します。diffをとるとGI5が明らかにVRF設定が足りていないので足します。
OSPFのrouter-idがないので入れます。これはなくてもOKかもしれません。

## 1-1
`sh ip bgp sum`でidleなのでneighborにpingします。NGなので`sh ip route`すると経路がないです。staticを書いてよさそうなので書きます。ping OKになってpeerがあがりました。

## 1-2
ピンポン防止とIPのみなのでdiscard不足か集約ミスか変なstaticだとメタ読みします。tracerouteの結果してからR1, R2が怪しそうだと。R2の`sh ip route`とトポロジ図を比べると/24のdiscardを書くのがよさそうなので書きました。

## 1-3
configのdiffをみると`no ip routing`が入っているので、仮想環境とEOSであることからスイッチング問題ではないと判断し制約を確認した後これを消します。
また、diffからnext-hopはtunnelでないとおかしそうなのでtunnel1経由にします。
diffからip routeが不足していそうなことを見つつ、tunnel問題でipsecもないのでsrc/destへの到達性を確認します。ないのでstatic経路を書きました。

## 1-4
configのdiffをみるとnetworkの数が違います。トポロジが対称なのにおかしいので足しました。

## 1-5
`sh bgp sum`でBGPが正しいことを確認した後、すべてのルータで両サーバへのrouteを確認します。RT1, RT2の間でClientの経路が交換されていないので`show bgp ipv4 unicast`します。`sh bgp ipv4 uni nei x.x.x.x adv`で経路送出されていないのでroute-mapがかかっていないことを横目にnetworkが足りていないので足します。
SVからtracerouteするとfirst hopが見えないのでip route showして足りないので足します。

## 1-6
`sh ip os int`があることを確認して`sh ip os neigh`してあがっていないため、`debug ip ospf packet/hello`します。明らかにtimerなので`ip os hel`で調整します。DeadがHelloの4倍なのでhelloだけ調整します。

## 1-7 
`R1とR4では、それぞれで /32 のスタティックルートを1つづつ、設定することが出来ます。`という制約をメタ読みします。R2に入れない条件からAD値を調整したstaticしかなさそうなので入れるとうまくいきました。

## 1-8
CiscoとJuniperですが頭で変換しつつdiffをとるとJuniperで再帰ルーティングされていないことがわかるのでメーカによる挙動差分だろうなと想定しながら検索してresolveをいれます。にしても

## 1-9: 
`sh bgp sum`や`sh bgp neigh recv`を確認しつつ`sh bgp 4.4.4.4`すると`r>i`という見慣れないフラグがあるのでヘッダを見ると`r RIB-failure`です。
1.1.1.1を届けられればいいのでdistanceで強くすればRIBに乗るかと思ったけどダメでした。ちょっと考えるとiBGPなのでフルメッシュかRRにしてあげたいのでRRにすればOKです。

## 1-10:
diffを取ろうとConfigを眺めると`ip ospf priority 0`で`sh ip os nei`で`never`なのでPrio1にするとOKでした。

## 1-11
`RTにconfig追加はOK`と`DNSの設定はかのうです`からメタ読みします。RTはACLが書いてあるので制約を見て`permit any any`を書いてpingをします。これでRTは考えなくてよさそうです。
axfrの転送を試みると失敗するので制約を見ながら`allow-transfer { any; };`を入れます。