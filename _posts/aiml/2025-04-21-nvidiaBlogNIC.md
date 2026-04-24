---
layout: post
title:  "読んだ: SuperNIC: What Is a SuperNIC?"
date:   2025-04-21T00:00:00+09:00
author: Akira Kanai
categories: aiml
cover:  "/assets/instacode.png"
---

# [SuperNIC: What Is a SuperNIC?](https://blogs.nvidia.com/blog/what-is-a-supernic/)

SuperNICとは2023頃からBlueFieldのファミリで`イーサネットベース`のAIワークロードを強化するための400G(以上の？)NICである。Spectrum-4 SWとの相性が良い。特徴を次に挙げる。

- NVIDIA SWと組み合わせることで順序性を保った分散ができる。(Adaptive Routingかな？)
- テレメトリデータとこれを利用した服装制御を行い（賢くRoCEを動作させ）AIネットワークの輻輳を効果的に抑制する
- プログラマブル可能であり、NW拡張が可能である(VXLANオフロードなど？)
- 低電力である
- フルスタックなフレームワークを搭載しておりAIネットワークに最適(??)

SuperNICの開発の背景にあるはEthernetとは必ずしもAIワークロードに適さないことである。端的に言えばEthernetは疎なアプリケーションを前提としているが、AIワークロードとはとても密なアプリケーションなのである。Ethernetは互換性を高く保つように設計されてきた。

一般的なNICには、AIワークロードに適切なデータ転送、低遅延性、確定したパフォーマンスのための機能が不足している。

さて、冒頭でBF3というシリーズを述べたがBFはDPU版とSuperNIC版がある。この違いを比較する。
原文は[この図](https://blogs.nvidia.com/wp-content/uploads/2023/11/bluefield-supernic-diagram.png)を参照すること。

`DPU版に比べて`の特徴を述べる。

- AIコンピューティングに最適化されており、RoCEに最適。East-West通信に最適化されている。
- コンピューティングではなくネットワーキングに注目
- セキュリティ・ゼロトラストではなく、AIネットワーキングに注目
- データストレージに注目ではなく、フルスタックなAIアプリケーションに注目
- 柔軟なネットワークよりも、電力効率が高い。x8を積むようなシステムでは大きな差となる。
- システムにつき1,2枚という構想ではなく、GPUごとに1枚(システムに8枚というような構成)

SuperNICが最も注力したのは、GPUとNICの比率を1:1とすることである。コンピューティングのリソースを削減し、高い伝率効率を行う。

また、`DPUにはない機能(難しい機能)が含まれている`。それは、`Adaptive Routing`, `out-of-order処理`, `輻輳の最適化`である。これらはいずれもEthernet上のAIワークロードで重要な要素である。


(この後に色々書いているけど繰り返しなので割愛)

訳注: DPU版とSuperNIC版の違いは消費電力でSuperNICはPCIeスロットからの75W給電で良いのに対してDPUは補助電源が加えて必要というコメントをXでいただいた。


# [BlueField:NVIDIA Accelerates Open Data Center Innovation - DOCA](https://blogs.nvidia.com/blog/bluefield-doca-dpu-open-data-center/)

OPI立ち上げの記事なのであまりtechな記事ではない。

NVIDIAは2022/06にLinux Foundation の Open Programmable Infrastructure (OPI) プロジェクトを立ち上げた（メンバの１つ）。このために、NVIDIA DOCAが非常に重要な役割を果たす。OPIプロジェクトはDPUを用いたコミュニティ主導のオープンエコシステム。DOCAはDPUを使うためのAPI/SDKであって、DPDK,OVSなどを使うことができる。これは将来BF以外のDPUもサポートするようになるだろう。DPUが目指すのは、Software Define, ZTNA, East-West通信の効率化などである。
