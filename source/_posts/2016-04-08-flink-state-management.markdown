---
layout: post
title: Apache Flinkのstate管理について
date: 2016-04-08 12:09:26 +0900
comments: true
categories: [Flink]
---

## はじめに

ストリーム処理の中で、処理をstatefulにしたい、という要求はよくある。例えば、1時間のtime windowで件数を集計している場合、ストリームが流れるにつれて内部で保持しているカウンタは増加していく.
そして、障害等で再起動をした時とかには、そのカウンタの値も一緒に復旧したい.

## Flinkにおけるstateの保存

これに対して、Apache Flinkは定期的に処理状態のスナップショットを取得する、という方法で対応している.
そして、分散環境でまともに全てのスナップショットを取るのは辛いので、分散してスナップショットを取るようにしている. 具体的には[ここ](https://ci.apache.org/projects/flink/flink-docs-master/internals/stream_checkpointing.html) に詳しいが、ストリームのソースから定期的にBarrierと呼ばれる印を流して、各オペレータはこれを受け取るとスナップショットを保存するようになっている. こうすることで、処理全体を止めずに一貫性のあるスナップショットを取れるよね、という話. 障害時にはスナップショットから状態を復旧し、スナップショット以降のログをリプレイすることで処理を継続する.

スナップショットの保存先は、以下から選択することができる. グローバルのデフォルトを設定することも、ジョブ単位でそれを上書きすることも可能.  (詳しくは[ここ](https://ci.apache.org/projects/flink/flink-docs-master/apis/streaming/state_backends.html))

* MemoryStateBacked
    * デフォルト. チェックポイントの際にJobManagerのメモリ上に状態を保持する. デフォルトはサイズが5MBまで.
* FsStateBackend
    * ファイルシステムに保持する. 最新の状態はTaskManagerのメモリに保持し、チェックポイントの際にファイルシステムに保存する. ファイルシステムはローカルでもHDFSでも良いが、障害時に別ノードで復旧することを考えるとHDFSになるだろう.
* RocksDBStateBackend
    * 最新の状態はTaskManagerローカルファイルシステム上のRocksDBに保持し、チェックポイントの際にそれをFsStateBackendと同じようにファイルシステムに保存する. スループットは若干落ちるが、大量のstateを保持することができる.

## 実験

とは言え、処理によってはstateが大きくなるケースもあるわけで、そういう時にどうなるんだろう？と思って実験してみた.
環境はFlink 1.0.0でKafka, Flink, HDFSは物理サーバ. 

こんな感じで、foobarというフィールドに100バイトのランダム文字列を入れて、それのdistinct countを取る. 重複を判断するためには、過去の値をすべて持たないといけないので状態は大きくなる.

```
$ head -n 1 dummy_log.aa 
id:0000 time:[2016-04-05 12:40:57]      level:DEBUG     method:POST     uri:/api/v1/people1     reqtime:3.053540515830725       foobar:dNjtvYKxgyym6bMNxUyrLznijuZqZfpVasJyXZDttoNGbj5GFk
```

こんな感じのコードで、処理を記述した(MyContinuousProcessingTimeTriggerはデフォルトのContinuousProcessingTimeTriggerに若干の改修を入れたもの). 
保存先はHDFSとしている. timeWindowAllなので、処理は分散されず単一のスレッドで処理される. 

```scala
    val stream = env
      .addSource(new FlinkKafkaConsumer09[String]("kafka.json2", new SimpleStringSchema(), properties))

    .map(parseJson(_))
      .timeWindowAll(Time.of(10, TimeUnit.DAYS))
      .trigger(MyContinuousProcessingTimeTrigger.of(Time.seconds(5)))
      .fold(Set[String]()){(r,i) => { r + i}}
      .map{x => (System.currentTimeMillis(), x.size)}
      .addSink(new ElasticsearchSink(config, transports, new IndexRequestBuilder[Tuple2[Long, Int]]  {
        override def createIndexRequest(element: Tuple2[Long, Int], ctx: RuntimeContext): IndexRequest = {
          val json = new HashMap[String, AnyRef]
          json.put("@timestamp", new Timestamp(element._1))
          json.put("count", element._2: java.lang.Integer)
          Requests.indexRequest.index("dummy3").`type`("my-type").source(json)
        }
      }))
```

まずは、260万件のログを送ってスナップショットのサイズを確認してみる. 送信したログのサイズ(100bytes×260万件)とほぼ同じ. そして、同じログを2回投入しても(distinct countなので
)サイズは増えない. 処理内部で集計した結果を保存している模様. 

```
$ cat dummy_log.aa >> /tmp/dummy_log2.log
$ hadoop fs -du  /apps/flink/checkpoints/9005988ca4dc6e871d50c2b310ed0a63
26000230  /apps/flink/checkpoints/9005988ca4dc6e871d50c2b310ed0a63/chk-183
$ cat dummy_log.aa >> /tmp/dummy_log2.log
$ hadoop fs -du  /apps/flink/checkpoints/9005988ca4dc6e871d50c2b310ed0a63
26000230  /apps/flink/checkpoints/9005988ca4dc6e871d50c2b310ed0a63/chk-196
```

そして、その時のチェックポイント時間はJobManagerのログから確認できる. 大体6秒くらいかかる. TaskManagerのCPU使用率は定常的に高い.

```
INFO  org.apache.flink.runtime.checkpoint.CheckpointCoordinator     - Completed checkpoint 43 (in 6291 ms)
```

さらに、260万件を追加で投入するとチェックポイントに10秒〜数十秒かかるようになった. ジョブを実行しているTaskManagerは
CPU100%になり、スループットも大幅に落ちていた.

```
INFO  org.apache.flink.runtime.checkpoint.CheckpointCoordinator     - Completed checkpoint 145 (in 41835 ms)
INFO  org.apache.flink.runtime.checkpoint.CheckpointCoordinator     - Completed checkpoint 146 (in 13530 ms)
```

[MLで聞いてみた](http://mail-archives.apache.org/mod_mbox/flink-user/201604.mbox/%3CCAMvK%3DYpTb6yZiv9ioL7%2Bmdn0W5Q3fRyNL210aGYcjzcbXQWxqg%40mail.gmail.com%3E) ところ、バックエンドでRocksDBを
試してみたら、とアドバイスをもらったのでやってみたが、変わらなかった. この処理の場合、状態を保存するストアには1レコードしか入らなくて、その値が非常に大きいので、効果が出なかった模様. 

## 思ったことなど

* ユニークユーザ数等をカウントするようなケースだと、値の種類が数千万というのもあり得るので、上のような性能だと辛い. そして、このような処理の場合、最終的には一箇所にまとめて
数を数える感じになるので、分散スナップショットの恩恵も受けづらい. 
* 現行のFlinkだと、スナップショットのたびに保持しているデータを全て書き出すので、incremental snapshotが導入されれば改善されるかも. 
* 今の時点での対応としては、値のリストをFlinkに保持せずに外部のデータストア(Redisとか)に保持するというのが一つの解決策. その場合、デメリットとしてはスケールアウトが
制限されることと、障害時などにログがリプレイされることへの対応. ただ、後者については処理の性質上、何度同じデータを投入しても結果は変わらないので、問題にならない.
* もしくは、HyperLogLogのような推定アルゴリズムを使って、精度と引き換えに状態のサイズを削減するか.

試しにFlinkで受け取った値をRedisのSETに入れて同じ処理を実現してみたら、あっさり1000万件くらい処理できた. stateが小さい、または分散される場合はFlinkに任せて、
今回のような一部のケースでは外部のデータストアを使う、というのが現実的だろうか、と思っているところ.
