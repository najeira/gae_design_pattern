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
必要になる時点まで読み込みを行わないことで、
スピンアップが遅くならないようにする。

importをモジュールレベルで行うと、スピンアップ時に読み込まれる:

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

インスタンスを終了させずに残しておきたい場合は、
Min Idle Instancesを設定する。

インスタンスが起動したまま残っていれば、
スピンアップは発生せず高速に応答できる。

設定の手順：

- Admin Consoleの左側のメニュー「Administration」の
「Application Settings」を選択する

- 「Performance」の「Idle Instances」の項目の上側のつまみが
「Min Idle Instances」、下側が「Max Idle Instances」

- つまみを左に動かすと小さな値に、右に動かすと大きな値になる

ここでMin Idle Instancesを1に設定すると、処理を行なっていない
アイドル状態のインスタンスが最低1つは残されることになる。


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
バックエンドで実行させることが出来る。

長い時間がかかる処理を行う必要がある場合で、
レスポンスが返るまでにその処理を終えなくてもよい場合、
タスクを登録だけして即座にレスポンスを返し、
必要な処理はTaskQueueを使ってバックエンドで行う。

例えば、ブログシステムでコメントが投稿された場合にメール通知する必要がある時、
コメントの保存とTaskQueueへのタスク登録を行ってレスポンスを返し、
メール通知はTaskQueueで行うようにする。

このように、ユーザが待たなくて良い処理はTaskQueueを使うと良い。



## CDN

# データストア

## モデル非正規化

AppEngineのデータストアでは、RDBのSQLにおけるJOINに相当するものがない。
複数のモデルのデータを結合してクエリすることが出来ないため、
あえて正規化せず（非正規化）にモデルを設計する。

例えば、人物と連絡先のデータがある場合、
PersonとContactの2つのモデルとして定義すると、以下のようになる。

class Person(ndb.Model):
    name = ndb.StringProperty()

class Contact(ndb.Model):
    person = ndb.KeyProperty()
    phone = ndb.StringProperty()
    address = ndb.StringProperty()

この時、住所 `address` からPersonを


## Entity Group

## Sharding Counter

AppEngineのデータストアで、
1つのエンティティグループにおける更新のスループットは高くない。
高くても10qpsほどで、目安として1qps以下となるように設計すると良い。

この時、更新頻度の高いカウンタのようなものを実装する時に、
1つのエンティティで実装してしまうと、更新スループットが出せずにうまく行かない。

更新頻度が高いデータは、異なるEGに属する複数のエンティティに分散するようにする。

class Counter(ndb.Model):
  count = ndb.IntegerProperty()

  @classmethod
  def incr(cls):
    rnd = random.randint(0, 10)
    key = ndb.Key(cls, rnd)
    def txn():
      obj = key.get()
      if obj:
        obj.count += 1
      else:
        obj = cls(count=1)
      obj.put()
    ndb.run_in_transaction(txn)

  @classmethod
  def get(cls):
    ret = cls.query().fetch(10)
    return sum( r.count for r in ret )

上記の例では、10個のエンティティに対してランダムに更新処理が振り分けられる。
このため、1エンティティで実装した場合よりも高い更新頻度に対応できる。

ただし、シャードの数を増やしすぎると、その合計値を取得するためのクエリが
遅くなるので、無闇に大きい数にせず、要件に応じて設定すること。


## 非同期API

データストア、Urlfetchなど、非同期版のAPIが用意されている。



## BASE Transaction

## TaskQueue

# 課金

# 運用

## バージョン


