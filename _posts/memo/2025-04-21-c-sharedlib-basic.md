---
layout: post
title:  "SharedLibraryを用いたSystemCallのHook"
date:   2025-04-21T00:00:00+09:00
author: Akira Kanai
categories: memo
cover:  "/assets/instacode.png"
---

# SharedLibraryを用いたSystemCallのHook

-  dlsym/dlopenを用いた動的なSharedLibraryのLoad

`undefined symbol: dlsym` と怒られる場合は、コンパイル時に-ldlを入れてください。

## 概要

LD_PRELOADを利用すると、shared libの関数をhookできる。
ただし、本来の関数名を他のアドレスに向けてしまうため、処理後に本来の関数をCALLするには、hookした関数内から元の関数を呼び出す必要がある。
この手法を実現するにあたって必要な要素となる、dlsymを使った動的なSharedLibraryのLoad方法について述べる。

### 基本コード

- main.c
```
  #include "lib.h"
  int main()
  {
    hoge();
  }
```

- lib.c
```
  #include <stdio.h>

  int static_value = 100;

  int hoge(void)
  {
    printf("hoge%d\n", static_value);
  }
```

- lib.h
```
  int hoge(void);
```

### コンパイル
```
  実行例
  $gcc -c lib.c
  $gcc -o a.out main.c lib.o
  $ldd a.out
  a.out:
  libc.so.7 => /lib/libc.so.7 (0x2807d000)
```

これは静的に一つのバイナリにライブラリを組み込んでいるもっとも基本的なパターンです。

### 共有libとしてのコンパイル

- soとしてのコンパイル
```
    gcc -o lib.so -shared lib.c
```

archiveのライブラリにしたいのなら、
```
  ar r lib.a lib.o
  gcc main.c lib.a
```

lib.soはshared libとして作成される。lddしようとすると、
```
  ldd lib.so
  lib.so:
  ldd: lib.so: Shared object "lib.so" not found, required by "ldd"
  lib.so: exit status 1
```
ので、
```
    LD_LIBRARY_PATH=. ldd lib.so
    lib.so:
    libc.so.7 => /lib/libc.so.7 (0x28089000)
```

とするとみえる。(何故、LDのLIBに解析対象のsoのdirが入っていないといけないのかは要調査)

### 実行
```
    gcc -o a.out main.c lib.so
    LD_LIBRARY_PATH=. ./a.out
    hoge100
```

ここで、lib.cのprintfの中身をhogehogeに変えると、
```
    gcc -o lib.so -shared lib.c
    LD_LIBRARY_PATH=. ./a.out
    hogehoge100
```
となる。ちゃんと、動的にlib.soを呼びに行っている。

## プログラム内からのsharedlibの読み込み

ここまでは、Linuxにおける、libの説明であった。ここから本題。

- main_load.c
```
  #include <stdio.h>
  #include <dlfcn.h>

  int main()
  {
    void *dl_handle;
    int  (*func) (void);

    dl_handle = dlopen("./lib.so", RTLD_LAZY);
    func = dlsym(dl_handle, "hoge");
    (*func)();
    dlclose(dl_handle);
  }
```
dl_handleは指定されたsoを開く。RTLD_LAZYはman参照。
OR演算でいくつかのオプションを付けることができるのだけど、dlsymは第二引数で指定したシンボルのアドレスを返す。シンボルなので、関数へのポインタでなくてよい。

```
  #include <stdio.h>
  #include <dlfcn.h>

  int main()
  {
    void *dl_handle;
    int  *value;

    dl_handle = dlopen("./lib.so", RTLD_LAZY);
    value = dlsym(dl_handle, "static_value");
    printf("%d\n", *value);
    dlclose(dl_handle);
  }
```

### 実行結果
```
    ./a.out
    100
```

と参照できる。これは、nmなどで参照すると、
```
    nm lib.so
    00001698 D static_value
```

というようにデータセクションのシンボルとして登録されていることがわかる。データセクションなので、書き込み可能で、

```
  #include <stdio.h>
  #include <dlfcn.h>

  int main()
  {
    void *dl_handle;
    int  (*func) (void);
    int  (*value);

    dl_handle = dlopen("./lib.so", RTLD_LAZY);
    func = dlsym(dl_handle, "hoge");
    value = dlsym(dl_handle, "static_value");
    *value = 200;
    (*func)();
    dlclose(dl_handle);
  }
```

とすると、
```
    hogehoge200
```
となる。面白い。

## 余談
```
  ar r lib.a lib.o lib.o
```
みたいに、同じobjを2回archiveしたらどうなるんだろうと気になった。

::
```
    nm lib.a
    lib.o:
    00000000 T hoge
    U puts
    lib.o:
    00000000 T hoge
    U puts
```

両方入る。

# 本来loadされるべきSharedLibraryの関数Hook

## 概要
-----

FreeBSD等のELFを用いているシステムではLD_PRELOADを使用することで、
通常のshared libに先駆けて指定したshared libを読み込むことができます。
これによって、
*既存のELFバイナリの*
システムコールなどに任意の処理を割り入れることができます。

今回はlibcのwrite(2)にhookをかけることを例に
見てみましょう。

LD_PRELOADの基本的な使い方

詳細はman ld.soなどで見れるRTLD(1)に記載されています。
まずは、システムコールを「上書き」する例を見てみましょう。


*write_hook_override.c*
```
  #include <stdio.h>

  size_t write(int d, const void *buf, size_t nbytes)
  {
    printf("write called.\n");
  }
```

*write_hook_test.c*
```
  main()
  {
    write(0, "hoge\n", 5);
  }
```

## 実行

```
    $gcc write_hook_test.c
    $ gcc -shared  -o write_hook.so write_hook_override.c
    $ ./a.out
    hoge
    // ここで、環境変数にLD_PRELOADを設定して再度実行します
    $ LD_PRELOAD=./write_hook.so ./a.out
    write was called.
```

a.outの起動時にwrite_hook.soのwriteシンボルが先にloadされ実行されました。
libc.soのwrite(2)がloadされていないことが分かります。

## hook後に従来の動作をおこなう
--------------------------------------

この例では、通常のwrite(2)の動作では、
`write(0, "hoge\n", 5);``
は"hoge"と出力されることが期待されますが、
write_hook.soのwriteで関数が上書きされてしまい、
本来の動作が行われません。

RTLD_NEXTを用いて元の処理に戻してみましょう。

### コード

*write_hook.c*
```
.. code-block:: c
   :linenos:

  #include <stdio.h>
  #include <dlfcn.h>

  ssize_t write(int d, const void *buf, size_t nbytes)
  {
    void *dl_handle;
    int  (*o_write) (int d, const void *buf, size_t nbytes);

    o_write = dlsym(RTLD_NEXT, "write");

    printf("write was called.\n");

    return(o_write(d, buf, nbytes));
  }
```

### 実行
```
    $ gcc -shared  -o write_hook.so write_hook.c
    $ LD_PRELOAD=./write_hook.so ./a.out
    write was called.
    hoge
```

dlsym(RTLD_NEXT, <symbol>);
は
>    Thus, if the function is called
>    from the main program, all the shared libraries are searched.

の通り、の動作を行われるため、結果として、
通常loadされるべきshared libから<symbol>を検索して、ポインタを返します。

## sudo時の注意

最近のsudoは通常env_resetがおこなわれるため、注意が必要です。
```
    $ LD_PRELOAD=./write_hook.so sudo ./a.out
    hoge
    $ sudo LD_PRELOAD=./write_hook.so ./a.out
    write was called.
    hoge
```

のように順番に気をつけてください。

### メモ
```
OSXの場合はDYLD_INSERT_LIBRARIESを使う(elf以外の仕様について調査)
```
