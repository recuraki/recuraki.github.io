---
layout: post
title:  "読んだ: NCCL関係"
date:   2025-04-21T00:00:00+09:00
author: Akira Kanai
categories: aiml
cover:  "/assets/instacode.png"
---

## [NCCL: Scaling Deep Learning Training with NCCL](https://developer.nvidia.com/blog/scaling-deep-learning-training-nccl/)
NVIDIA Collective Communications Library (NCCL) はNCCL2.3からオープンに提供されるようになった。allreduce やvariantなどのGPU間通信が必要な操作に最適な実装を提供する。NCCLとはトポロジを認識してMPI互換の操作を提供する。これによって、ノード内の複数のGPUあるいは、ノード間の複数のGPUでの演算を最適化する。
NCCLはノードの帯域を最大に利用する。このためにノード内ではPCIe/NVLink, ノード間ではInfiniBand(には限られず、Ethernetでもよいだろう)を活用する。

ノード内のパフォーマンスについて考える。同一ノード内で重要なネックはCPU間接続の細さだ。異なるCPU間の通信はsharedのメモリを利用する必要があり5GBps程度に制限される。CPUを介さずにGPUがmemory2memoryを行う場合は少し高速化され8GBps程度になる。
NVLinkがある場合、GPUは直接ローカルのGPUと通信できるため62Bpsや132GBpsなどの転送ができる。

ノード間のパフォーマンスについて考える。TCPの遅さやCPUを介することによるパフォーマンス低下が生じる。このために、IB/RoCEv2が活用でき、GPU Direct RDMAによって100GbEなら82Gbps(x8 GPU環境時)くらいの転送が可能になる。

レイテンシについて考える。NCCLではメモリフェンス（ここでは通常の意味でメモリの順序性のことだろう）を必要としないスレッド間の転送をおこうことで非常にレイテンシの少ない転送が可能になった。

操作の集約の工夫を行なっている。各操作をグループ化して複数のGPUに同時に投入することでGPUの操作命令の送信にかかるコストを減らせる。これにより64GPUの例では14倍高速になった。

NCCLはプロセスやスレッドからGPUをうまく操作することもできる。単一のプロセスから単一のGPUを操作するようにすることもできれば、複数のGPUを（単一のスレッドから）操作することもできる。

# [NCCL: Doubling all2all Performance with NVIDIA Collective Communication Library 2.12](https://developer.nvidia.com/blog/doubling-all2all-performance-with-nvidia-collective-communication-library-2-12/)

NCCLはCollective communicationsを実現するライブラリであり、
`all-gather`,
`all-reduce`,
`broadcast`,
`reduce`,
`reduce-scatter`,
`point-to-point send and receive`
と言った操作をサポートする。
NCCLはトポロジを意識することでこれらの最大の性能(高帯域と低遅延)を実現する。

さらにオンプレミスの環境だけではなくて、AWS, GCPに対応したプラグインもNVIDIAから公式にリリースさせておりクラウド上での利用も可能である。

`PXN(PCI x NVLink)`は、ノード内において、GPUがローカルNICではなくリモートNICに対して高速にアクセスできる仕組みだ。NCCL 2.12(2022/2)からサポートされた機能であり、GPUはPCIeを介するのではなくて、GPU向けのLinkであるNVLinkを介して他のNICにアクセスする機能である。これはマルチプロセッサ環境におけるQPI等を介した通信を大幅に改善できる。
今、`CPU1 - GPU1 - NVLink - GPU2 - NIC2`でデータを通信したいとする。この場合、GPU1は書き込みたいデータをGPU2側にNVLinkを介してデータを送る。そして、GPU2はCPU（これどっちだ？）にデータの書き込み準備ができたと通知する。

`Rail optimized`はNKLinkを備えたNVIDIAのGPUシステムで利用できるAll-ReduceをNW最適で実現するシステムである。[Rail Optimized Topology Validation](https://docs.nvidia.com/networking/display/ibdiagnetusermanualv211/rail+optimized+topology+validation)をみるとわかるのでが同一のToRに接続された全てのノードにおいて、同一のPCIeバス上のNICがそのToRに接続されているかを確認する。より具体的にはPCIeバスのアドレスが全て同一であるかを確認する。具体的な出力例をみるとわかりやすい。
```
Rail Optimized Topology Validation
-W-     Node rail connectivity mismatch on the switch: "SwitchIB Mellanox Technologies" GUID=0xe41d2d030003e470
-W-              rail A (PCIe 0000:04:00.0): 1 ports ==> r-ufm118/U1/P1 <--> Se41d2d030003e470/Ne41d2d030003e470/P35
-W-              rail B (PCIe 0000:09:00.0): 1 ports ==> r-ufm112/U2/P1 <--> Se41d2d030003e470/Ne41d2d030003e470/P30
-
```

`Rail optimized`をNWの視点で見るならば、NVLinkを活用してSpineを通らない通信だけを行う、と言える。全てのNICが上記の条件を満たしているならば、remoteノードの宛先のNICが属するToRにつながるlocalノードのNICまでNVLinkを介して届け、そのあとに相手側のCPUに対してその結果をsyncする(これはよく分かっていない。PCIe的ななんらかのシグナル？)。
実例をあげよう。`CPU0側のNIC0`が`CPU1側のGPU3`と通信したいとするとしよう。RailOptimizedでは`CPU0側のNIC3に転送した後、NIC3同志の繋がるLeafによりCPU1側のNIC3に到達してGPU3に転送`される。つまり`宛先のGPUが存在するLeafのNIC に接続されているローカルのGPUに情報を転送する`と言える。ここでの`ローカル`とは同じNVLinkに所属していることを意味する。

これらの機能は`NCCL 2.12から`サポートされている。すなわち、従来であれば`NIC0 -> Leaf0 -> Spine -> Leaf3 -> NIC3 -> GPU3`であった。これはSpineを介するたたオーバーサブされていることがあるだろうし、Switchの段数が多い。(段数は必ずしも遅延への関与だけを意味しない。たとえばリンクの負荷分散などの箇所を減らすことでよりネットワーク側で考えることが減るだろう)

## 送信側でのメッセージ集約(メッセージ数の削減)
この仕組みはall2allを実行する上で非常に効果的に動作しメッセージの数を減少させられる。ローカルの複数のGPUがある`あるリモートGPU3`と通信したいとしよう。`ローカルGPU3`はその対象宛のメッセージを集約して送ることができる。リモート側のGPUは相手のNVLINKのGPU数分(たとえば8)のバッファを用意し、一斉に`multi recive`を発行する。

これによってメッセージ数の減少と平均メッセージ長の増加が起こる。これによってネットワーク側のキュー制御などは今まで以上に効果的に動作するだろう。

PXNを実行した場合、記事のFig.3によれば128Nodes/1024GPU環境下でall-reduce操作のパフォーマンスは2.5倍ほど良くなる。(図を睨むと少数で2倍程度なので約2-2.5倍といっていい)

PXNは万能ではなくall2allを効果的に操作する。一方で`Tree/Ring`を最適化するわけでない。Tree-Ringについては[Massively Scale Your Deep Learning Training with NCCL 2.4](https://developer.nvidia.com/blog/massively-scale-deep-learning-training-nccl-2-4/)を参照せよ。

## 検討: GPU:NICが1:1の場合のall-reduce
従来の構成でのRing構成は1つのNICに2(以上？？)のGPUが必要であった問題が解決される。Ring構成とは、そのノード内の（あるCPU側の？）最初のGPUと最後のGPUがあり、それらがCPUの補助を得たPCIeなどでうまくルーティングされることを期待している。ところが、1:1での構成ではこれらはうまく動作しない。
ところがPXNはNVLinkを介して任意のGPU間が通信可能であるため1:1でGPU:NICが対応していようが最後のGPUはそのリングの外に出すべき情報を集めて外に出すことができる。

## まとめ
(まとめより少し前に書いてある内容だが)PXNによってNCCLトポロジの柔軟性は増す。例えば従来のモデルではGPU0,2,4,6というセットでしかフローを作れなかったものを、GPU0から7まで全てを活用したトポロジが作れるようになる。

このようにNCCL2.12はall2allのパフォーマンスが大きく向上した。
