---
layout: post
title:  "AI Data Center GPU Backend Fabric with Juniper RDMA-aware LB and BGP-DPF—Juniper Validated Design (JVD)"
date:   2025-11-17T00:00:00+09:00
author: Akira Kanai
categories: Network
cover:  "/assets/instacode.png"
---


[AI Data Center GPU Backend Fabric with Juniper RDMA-aware LB and BGP-DPF—Juniper Validated Design (JVD)](https://www.juniper.net/documentation/us/en/software/jvd/jvd-ai-dc-qpp-bgp-dpf/jvd-ai-dc-qpp-bgp-dpf.pdf)

Juniper Validated
Design (JVD)はJuniperのエンジニアによって検証されたデザインであり、これを使うことでデザインリスクを抑えることができる。
より具体的にメリットを表現するならば、`検証済みの構成`、`スケーラブル`、`リスク低減`, `自動化への対応`, `制約などが予測可能`, `再現可能`, `開発の加速`, `境界が明確であるデザイン`, `ベストプラクティスに基づいている`などが挙げられる。

このドキュメントはJuniperのAIクラスタのデザインに関するものであり。
- Juniper RDMA-Aware Load Balancing (RLB)
- Juniper BGP Deterministic Path Forwarding
(BGP-DPF) in

という２つの技術をGPUバックエンドのために用いる。ネットワークとしてはQFX5240およびH100ノードを想定する。このネットワークは3stageのCLOS環境として構築される。

# ネットワークの概要
このNWは`フロントエンドFabric`, GPUのための`バックエンドFabric`, ストレージのための`ストレージバックエンドFabric`で構成される。これらのネットワークを全てEthernetで構築する。特殊なのはバックエンドFabricでこれにはRoCEv2が用いられており、ロスレスな環境を実現する。
