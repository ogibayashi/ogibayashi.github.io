---
layout: post
title: Apache Flinkの性能 - デフォルトのJSONパーサが遅かった話
date: 2016-04-05 10:50:12 +0900
comments: true
categories: [Flink]
---
軽くApache Flinkの性能を測ってみた. 構成としては、Fluentd(in_tail→out_kafka_buffered)→Kafka→Flink→Elasticsearchで、仮想アクセスログ的なものに対して、URIごとの件数を1分ごとに集計して出力する、というもの。
メッセージフォーマットはJSON。

```scala
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.enableCheckpointing(1000)
// ....
    val stream = env
      .addSource(new FlinkKafkaConsumer09[String]("kafka.dummy", new SimpleStringSchema(), properties))
      .map{ json => JSON.parseFull(json).get.asInstanceOf[Map[String, AnyRef]] }
      .map{ x => x.get("uri") match {
        case Some(y) => (y.asInstanceOf[String],1)
        case None => ("", 1)
      }}
      .keyBy(0)
      .timeWindow(Time.of(1, TimeUnit.MINUTES))
      .sum(1)
      .map{ x => (System.currentTimeMillis(), x)}
      .addSink(new ElasticsearchSink(config, transports, new IndexRequestBuilder[Tuple2[Long, Tuple2[String, Int]]]  {
        override def createIndexRequest(element: Tuple2[Long, Tuple2[String, Int]], ctx: RuntimeContext): IndexRequest = {
          val json = new HashMap[String, AnyRef]
          json.put("@timestamp", new Timestamp(element._1))
          json.put("uri", element._2._1)
          json.put("count", element._2._2: java.lang.Integer)
          println("SENDING: " + element)
          Requests.indexRequest.index("dummy2").`type`("my-type").source(json)
        }
      }))

```

環境は以下の通り

* Kafka : 8vCPU, 8GBMemoryのVM*3, HDP2.4 (Kafka-0.9)
    * パーティション数は1なので、実質broker1台
* Flink : JobManager, TaskManagerともに8vCPU, 8GBMemoryのVM (Flink-1.0.0)
    * TaskManagerは3台だが、ジョブ並列度を1にしたので1台でしか動かない状態
* Elasticsearch: 4CPU, 4GB MemoryのVM*1 (Elasticsearch 1.7.2)

URIは9種類なので、Elasticsearchには毎分9レコードが出力されることにいなる。 Elasticsearchに出力されたURIごとの件数を1分単位に合計したものを、Flinkのスループットと考える。 レコードの生成には [dummer](https://github.com/sonots/dummer)を使用した。

で、普通にやったら2,000msg/secの入力を与えて、89,000msg/min = 1,483msg/secしか処理できなかった。FlinkプロセスのCPUが100%(1CPU使いきり)となり、Kafkaのlagが増えていく状態。
チェックポイント外したりESへの出力をやめたりしてみたのだけど、性能はさほど変わらず。

何にCPUを食っているんだろう？と思って、Flinkプロセスのjstackを何回か取ってみたら、こんな感じでJSONのパースを実行中であるケースが殆んどだった。

```
        at scala.util.parsing.combinator.Parsers$$anon$2.apply(Parsers.scala:881)
        at scala.util.parsing.json.JSON$.parseRaw(JSON.scala:51)
        at scala.util.parsing.json.JSON$.parseFull(JSON.scala:65)
        at KafkaTest3$$anonfun$1.apply(KafkaTest3.scala:46)
        at KafkaTest3$$anonfun$1.apply(KafkaTest3.scala:46)
        at org.apache.flink.streaming.api.scala.DataStream$$anon$4.map(DataStream.scala:485)
```

ということで、パーサをJacksonにしてみた。


```scala
  val mapper = new ObjectMapper()

  def main(args: Array[String]) {

    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.enableCheckpointing(1000)

    // ...

    val stream = env
     .addSource(new FlinkKafkaConsumer09[String]("kafka.json", new SimpleStringSchema(), properties))
      .map(parseJson(_))
      .map{ x=> (x.get("uri").asInstanceOf[String], 1)}
      .keyBy(0)
      .timeWindow(Time.of(1, TimeUnit.MINUTES))
      .sum(1)
      .map{ x => (System.currentTimeMillis(), x)}
      .addSink(new ElasticsearchSink(config, transports, new IndexRequestBuilder[Tuple2[Long, Tuple2[String, Int]]]  {
        override def createIndexRequest(element: Tuple2[Long, Tuple2[String, Int]], ctx: RuntimeContext): IndexRequest = {
          val json = new HashMap[String, AnyRef]
          json.put("@timestamp", new Timestamp(element._1))
          json.put("uri", element._2._1)
          json.put("count", element._2._2: java.lang.Integer)
          Requests.indexRequest.index("dummy2").`type`("my-type").source(json)
        }
      }))

    env.execute("KafkaTest9")
  }

  def parseJson(x: String): Map[String,AnyRef] = {

    mapper.readValue(x,classOf[Map[String,AnyRef]])
  }

```

そうしたら、スループットが大幅に向上して900,000msg/min=15,000msg/secを20%程度のCPU(1CPUの5分の1)で処理できた。

http://www.slideshare.net/nestorhp/scala-json-features-and-performance にScalaで使えるJSONライブラリを比較したスライドがあるのだけど、
これを見るとScala標準のパーサと、その他のライブラリで速度が3桁くらい違う。えーーー、、、

ともあれ、十分に実用的な性能が出そうであることがわかったので良かった。
