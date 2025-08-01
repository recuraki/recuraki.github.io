---
layout: post
title:  "Modal入門"
date:   2025-07-29T00:00:00+09:00
author: Akira Kanai
categories: HostNetwork
tags: HostNetwork
cover:  "/assets/instacode.png"
---

# [Introduction](https://modal.com/docs/guide)

ModalはCloud Function Platformである。コードを短時間で起動させることができ、あらかじめビルドしたコンテナを展開できる。そして、数千のコンテナに展開をでき、GPUを簡単にAttachできる。このコードはWebのエンドポイントとして後悔することもできるし、スケジュールされたジョブの実行もできる。また、辞書、キューのようなデータ蓄積も可能である。

Modalの最大の特徴はこれらをサーバレスで構築できることであり、よくある時間や分単位ではなくて`秒単位`の課金であることである。また、これらはコードベースで書かれ、YAMLなども必要がない。

`Getting started`は省略する。

`How does it work?`:
Modalはコードを受け取るとそれをもとに１コンテナに立てて実行する。ここでは、そのコードがどこで実行されるかを意識する必要はないし、k8sのようにどのpodで動かすかを意識する必要はない。

# [Images](https://modal.com/docs/guide/images)

ここでは、Modalの関数が動くImage/環境について見ていく。

環境はコンテナと呼ばれる。コンテナは一般的なコンテナと同じ意味合いで捉えればよく、OSの上に実行環境を作り、また、簡単に環境を複製できる。つまり、他の環境に汚される、あるいは、環境を汚すことなく実行環境を手に入れることができる。
尚、Modalのコンテナはセキュリティの都合から[gVisor container runtime](https://cloud.google.com/blog/products/identity-security/open-sourcing-gvisor-a-sandboxed-container-runtime)の上で実行されている。

各コンテナは起動時に`Image`と呼ばれるファイルシステムのスナップショットから起動される。Imageを作ることは`Building`と称される。Modalでは標準的なコンテナはDebianであり、Python3.xが入っている。

Modalで適切なイメージを作るために、任意のパッケージやPythonのライブラリをインストールしてカスタムイメージを作ることができる。Modalではいくつかの抽象度のコマンドでこれらを生成することができ、`pip_install`を使うこともできるし、`run`では非常に具体的なコマンドを実行することもできる。

Modalではベースイメージから`method chaining`という方法を使ってカスタムイメージを生成できる。これにより、階層Buildが実行される。

`Add Python packages with pip_install`:
いくつかの例を実施に見ていく。pip_installを使うことでPythonのパッケージをインストールできる。もちろん。`torch == 1.9.1`のようにexactなバージョン番号を指定することだってできる。

さて、Modalはシングルソースコードで複数のModal関数を動かせるわけだけども、そうすると全ての関数で同じバージョンのpipライブラリが使われてしまうのだろうか？この心配は無用で、関数ごとにイメージを指定できる。イメージは１つのコードの中で複数宣言できる。つまり、関数ごとに違うイメージが使える。
尚、imageを指定する際に`python_version`でベースイメージにインストールされるPythonのバージョンも制御可能である。

`Add local files with add_local_dir and add_local_file`:
イメージを作る時にローカルのファイルをイメージ側に送ることもできる。いわば`cp`コマンドのようなものだ。ここでTIPSを1つ。階層Buildが行われるのでこのコマンドは後ろのほうにchainする方が良い。

ローカルPythonライブラリの追加
Modalではローカルのライブラリを追加することもできる。詳細は割愛。

ローカルに追加していないライブラリをインポートしたい場合
Modalを使い始めて問題になるのはソースコードで記述を行うために、利用するfunction内でローカルにないライブラリを利用したい時にどうするか？という問題であるが、これは、deployするfunctionの中にimport句を記載すればよい。
一方で、ライブラリをグローバルスコープで運用したいこともあり、その場合は`imports()`という特殊な構文を使う。これも割愛する。

ここまでに述べたインポートの作業はコンテナの入力前に実施される処理であるため、[メモリスナップショット](https://modal.com/docs/guide/memory-snapshot)と組み合わせることで、[コールドスタートの速度](https://modal.com/docs/guide/cold-start#share-initialization-work-across-cold-starts-with-memory-snapshots)を向上できる。

`run_command`はその名の通りコマンドを実行するコマンドで、uvをインストールしてからruncommandでuvを実行することもでき、これは高速に実行される。尚、uvを実行する時`--compile-bytecode`をつけることが非常に重要である。なぜなら、pipと異なってuvはバイトコードを生成しないため、これをつけないとライブラリのインポート時に各コンテナがバイトコードコンパイルをしてしまい勿体無い。

run_commandに似ているが異なる`run_function`がある。これはシェルのコマンドではなくて、Pythonの関数を実行する。ユースケースとして、Hugging Faceのファイルをダウンロードすることを考えれば良い。この場合、diffusers.StableDiffusionPipeline.from_pretrained().save_pretrained(path)などをPythonのコードとして実行し、その結果をrunしたくなるだろう。

尚、セットアッププロセスはCPUリソースで行われる。このため、GPUを必要とするsetupプロセスがある場合、gpu=引数をつけてそのプロセス中にGPUをattachすることもできる。

micromamba_installは割愛。

`from_registry`は外部のレジストリから適切なイメージを引っ張ってこられる。一定のPythonバージョンがインストールされていることなど条件はあるが、nvcr.io, ghcr,ioやdocker HUBなどを使うことができる。また、プライベートreposへのアクセスも可能である。

`from_dockerfile`を使うと、あらかじめ作ってあった`Dockerfile`を使ってdocker buildを走らせることもできる。この後にchainも可能である。

尚、`force_build=True`によりイメージのbuildを強制できます。

Modalが提供するベースイメージはあまり更新されません。なぜなら全部の顧客に影響が及ぶからです。このため、基本的にはワークスペースごとにあるベースイメージのバージョンは決まっていて、自分のタイミングでこれを更新することもできます。
