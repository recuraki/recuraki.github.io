---
layout: post
title:  "読んだ: Thousands of NVIDIA Grace Blackwell GPUs Now Live at CoreWeave, Propelling Development for AI Pioneers"
date:   2025-04-21T00:00:00+09:00
author: Akira Kanai
categories: aiml
cover:  "/assets/instacode.png"
---

# [Thousands of NVIDIA Grace Blackwell GPUs Now Live at CoreWeave, Propelling Development for AI Pioneers](https://blogs.nvidia.com/blog/coreweave-grace-blackwell-gb200-nvl72/)

CoreWeaveがGB200 NVL72をクラウド提供する最初のプロバイダになったという話。GPU規模は数千。利用者はCohere、IBM、Mistral AIなど。

CoreWeaveとはなにか？USのクラウドスタートアップ企業。[L40以上を提供している](https://www.coreweave.com/pricing)GPUクラウド企業として見るのが良さそう。この環境は学習と推論両方に使える。記事上では`110,000台のGPUが連携`できると書いてある。が、文脈的にはGB200が10万台以上という訳ではなくて、様々なGPU合わせて。だろう。

NVIDIAのウリはもちろん、新しいGrace Blackwellをラックスケールとして導入できるところだ。カスタマー側がBlackwell向けのチューニングを行わなくてもoptimizeされた性能が得られることがメリット。具体的にはラックスケールとして最初から72GPUが`NVLinkとして`連携したワークロードが動かせる。これによって、システムとしては数千台のGPUを使ったワークロードが流せる。

具体的なユースケースはIBMでGraniteというモデルの学習をこの環境で行っている。

CoreWeaveはIBMのシステムをクラウドとしても利用しており、IBM Storage Scale Systemをクラスタで使っているらしい。
