---
layout: default
title: Home
---

# スピンアップ

AppEngineではリクエストに応じて自動で、
インスタンス（サーバ）を起動したり、
不要になったインスタンスを終了したりする。

このインスタンスの起動のことを「スピンアップ」と言う。

スピンアップに時間がかかってしまうと、
レスポンスに時間がかかってしまうのはもちろんのこと、
AppEngineの自動スケールアウトの恩恵を受けにくくなる。

このため、素早くスピンアップできる状態にしておく必要がある。


## Lazy loading

ライブラリの読み込みに時間がかかってしまうと、
スピンアップが遅くなってしまう。

そこで、起動時に全てのライブラリを読み込まず、
必要になった時点で読み込むことで、
スピンアップが遅くならないようにする。

importをモジュールTOPで行うと、スピンアップ時に読み込まれる:
    import lib_a
    import lib_b
    
    def foo():
      # use lib_a
      ...
    
    def bar():
      # use lib_b
      ...

importを必要になった時点で行うようにする:
    def foo():
      import lib_a
      # use lib_a
      ...
    
    def bar():
      import lib_b
      # use lib_b
      ...


## Warmup

リクエストがあった場合、AppEngineは必要に応じて
新しいインスタンスを起動する（スピンアップ）。

起動済みのインスタンスがレスポンスする場合と比べて、
スピンアップでは起動処理が行われるため時間がかかる。

Warmup Requestsを有効にすることで、この問題を緩和することが出来る。

app.yaml の inbound_services ディレクティブに設定する:
    inbound_services:
    - warmup

Warmup Requestsが有効な場合、
AppEngineは新しいインスタンスが必要になりそうだと判断すると、
新しいインスタンスを起動するためのリクエストを送信する。

Warmup Requestsは`/_ah/warmup`に対して送信されるので、
これを受け取るハンドラを記述し、ライブラリの読み込み等を行なっておくと良い。

なお、Warmup Requestsによって起動されたインスタンスも、
通常のリクエストの場合と同様に課金対象となる。


## Min Idle Instances

AppEngineは処理を行なっていないインスタンスを自動で終了する。

スピンアップを回避するために、インスタンスを終了させずに
残しておきたい場合は、Min Idle Instancesを設定する。

Admin Consoleから


## Bytecode upload


## Python Precompilation

AppEngineはデフォルトで、Pythonのファイルをプリコンパイルする。
これにより、多くの場合は処理速度が向上する。

通常、これは有効なままで使用すれば良い。

もしプリコンパイルを無効にする場合は、
appcfg.py の --no_precompilation を使用する。


# レスポンス

レスポンスはなるべく早く返す。

2012年3月の時点で、AppEngineのデータセンターは
アメリカ内にあると思われ、日本にはない。

このため、リクエストとレスポンスは

## Memacache

AppEngineのデータストアは非常にスケーラビリティがあるが、
最小の処理でも20ms程度の時間がかかる。

キャッシュからレスポンスを返すことができれば
データストアやその他の処理を行わないので高速になる。

AppEngineでは標準でMemacache APIを提供しているので、
キャッシュ可能なものは出来る限りキャッシュしたほうが良い。

なお、Memacache APIの利用には課金は発生しない（無料）。


## FrontEnd Cache

FrontEnd CacheはBillingが有効になっている場合に、
自動で適用される。

HTTPレスポンスに含まれるヘッダによって、
レスポンスがキャッシュされてもよいことが示されると、
自動でFrontEnd Cacheが使われる。

Cache-Control ヘッダ:
    Cache-Control:public, max-age=3600

上記の例では、レスポンスは最大3600秒キャッシュされる。

FrontEnd Cacheが有効な場合は、AppEngineのアプリケーションに
リクエストは到達せず、その手前でレスポンスが返される。
このため、インスタンスの処理が行われず、IHの節約にもなる。

さらに、FrontEnd CacheはGoogleのCDN（Contents Delivery Network）が利用され、
リクエスト元に近い位置からの高速な応答が期待できる。

なお、FrontEnd CacheによるレスポンスはBandwidth Outに含まれる。


## TaskQueue

TaskQueueを使って、一部の処理をレスポンスの処理とは別に
バックエンドで実行させることが出来ます。

長い時間がかかる処理を行う必要がある場合で、
レスポンスが返るまでにその処理を終えなくてもよい場合、
タスクを登録だけして即座にレスポンスを返し、
必要な処理はTaskQueueを使ってバックエンドで行います。

例えば、ブログシステムでコメントが投稿された場合にメール通知する必要がある時、
コメントの保存とTaskQueueへのタスク登録を行ってレスポンスを返し、
メール通知はTaskQueueで行うようにします。

このように、ユーザが待たなくて良い処理はTaskQueueを使います。



## CDN

# データストア

## モデル非正規化

## 非同期APU

## Sharding Counter

## Entity Group

## BASE Transaction

## TaskQueue

# 課金

