---
layout: post
title:  "壁紙を作りたいメモ"
date:   2023-10-22T00:00:00+09:00
author: Akira Kanai
categories: diary
cover:  "/assets/instacode.png"
---

# 20231022: 壁紙を作りたいメモ

## やりたいこと
ノートPCで違う壁紙を作りたい。作り置きをしておいても良いが、Modalを使って生成を任せる。

ビデオメモリの制約からMBPや5K2Kディスプレイの画像を生成することは現実的でない。そこでSDで小さめの画像を作成し、upscaleすることにする。

## upscaleの仕方
ターゲット
5K2K = [5120,2160] = [1280, 540] * 4
MBP14 = [3024 , 1964] = [756, 491] * 4

## 主にdiffuser周りメモ

https://github.com/huggingface/diffusers/blob/main/docs/source/en/using-diffusers/custom_pipeline_examples.md

- Pipeline(Stable Diffusion Pipeline)は推論のためのパイプラインである。コミュニティに依る完成されたパイプラインは以下のように取得できる。
>StableDiffusionPipeline.from_pretrained("runwayml/stable-diffusion-v1-5")
- カスタムPipelineを作ることもできる。
- Long Prompt Weighting Stable Diffusionは標準では77トークンの制限があるものをトークン制限なしに処理する。()によって強調し、[]によって抑制が可能になる
- 