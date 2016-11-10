---
layout: post
title: 多種多様なログをFluentd-Elasticsearch-Kibanaしたメモ
date: 2016-11-09 19:36:46 +0900
comments: true
categories: [elasticsearch, fluentd]
---
自分のところでは、社内の様々なログをfluentdで集めているのだけど、それらをElasticsearchに入れて、Kibanaで見えるようした話を書いてみる.

Fluentd, Elasticsearch, Kibanaな組み合わせは既に多くの人が使ってるし、ブログ等の記事も沢山ある. けど、いくつか
引っ掛かった点などもあるので、それらを書き留めて置こうと思う.

## 要件

* fluentdでストリームで集めてるログを、リアルタイムに近い鮮度でKibanaで見たい.
  * 見たい、というのは検索したり、集計・可視化したり、ということ
* Kibanaでは短期(2日とか)のログが見えれば良い
* 規模感的には、ログの種類(=fluentdのtag数)が200くらい、流量はピークで2万メッセージ/秒くらい.
  * が、実際には最も流量の多いログは除いたので、この半分〜3分の1くらいの流量
  * 10shard/index にしてあるので、２日分だと4000 shardくらいになる. これはかなり多いと思う
* ログの種類が増えたときも、人手をかけずにKibanaで見えるようになって欲しい

## 構成

fluentd→Kafka→[kafka-fluentd-consumer](https://github.com/treasure-data/kafka-fluentd-consumer)→fluentd→Elasticsearch

な感じ. Elasticsearchへの投入が詰まる、的な話はよくあるけど、Kafkaを挟んでいるのでfluentdの他の経路に影響が出ることは無いようになっている.

## 経験した問題と対応

### fluentdからの投入タイムアウト

これに出会ったのは大分前なのだけど、流量が多い場合にバッファのサイズが大きくなると、Elasticsearchへの投入でタイムアウトが発生したりした.

```
2015-09-30 18:02:48 +0900 [warn]: temporarily failed to flush the buffer. next_retry=2015-09-30 18:01:02 +0900 error_class="Fluent::ElasticsearchOutput::ConnectionFailure" error="Could not push logs to Elasticsearch after 2 retries. read timeout reached" plugin_id="object:3fe40898b7d8"
```

対応として、fluent-plugin-elasticsearchの設定で`request_timeout`を長めに設定している. 

### Size of the emitted data exceeds buffer_chunk_limit

kafka-fluentd-consumer からログを受けているfluentdで、以下のようなwarningが多発した.

```
2016-07-31 21:22:51 +0900 [warn]: Size of the emitted data exceeds buffer_chunk_limit.
2016-07-31 21:22:51 +0900 [warn]: This may occur problems in the output plugins ``at this server.``
2016-07-31 21:22:51 +0900 [warn]: To avoid problems, set a smaller number to the buffer_chunk_limit
2016-07-31 21:22:51 +0900 [warn]: in the forward output ``at the log forwarding server.``
```

fluentdがin_forwardで受け取ったchunkのサイズが、受け側fluentdでのbuffer_chunk_limit(デフォルト8MB)より大きいよ、という
メッセージ.

kafka-fluentd-consumerが大きいchunkで送信していたと思われる. 当時はこれを制御する術は無かったのだけど、issueをあげたら
対応してもらえた. このためにkafka-fluentd-consumerが内部で使っている[Fluency](https://github.com/komamitsu/fluency)にも、改修を入れてくれた様子. 有り難い.

https://github.com/treasure-data/kafka-fluentd-consumer/issues/15

こんな感じで、Fluencyのバッファ周りのパラメータを指定できるようになったので、`fluentd.client.buffer.chunk.retention.bytes`がfluentdのbuffer_chunk_limitより小さくなるようにした.
あとは、Elasticsearchに投入するfluentdプロセスの数を多めにして、一つ一つの負荷が大きくならないようにしている.

```
fluentd.client.buffer.chunk.initial.bytes = 1000000
fluentd.client.buffer.chunk.retention.bytes = 8000000
fluentd.client.buffer.max.bytes = 512000000
```

実際には、この設定を入れる前の段階で(なぜか)warningが消えていたので、明確にこれの効果があったのかは不明. だけど、その後ログの流量を大分増やしたりしても
出ていないので、多分効果はあったのだろう.

### 日付が変わったタイミングでElasticsearchが固まる問題

時系列データなので、indexは`somelog-YYYY.MM.DD`のような形で、日毎に分けるようにしている. 一部のログだけを入れていた時は特に問題は無かったのだけど、
全ての(大体200種類の)ログを入れるようにした所、日付が変わった際にElasticsearchへのindexingが固まる、という事象に出会った.

![Elasticsearch_indexing_rate](/images/2016/Elasticsearch_indexing_rate.png)

Elasticsearchのログを見ると、以下のようなログが大量に発生している.

```
ProcessClusterEventTimeoutException[failed to process cluster event (put-mapping [fluentd]) within 30s]
at org.elasticsearch.cluster.service.InternalClusterService$2$1.run(InternalClusterService.java:349)
at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
at java.lang.Thread.run(Thread.java:745)
```

これについては、Elasticsearchのフォーラムで質問をしてみた.
https://discuss.elastic.co/t/timeout-exception-with-many-time-based-indices-after-00-00/63340/1

mapping等の情報はcluster stateと呼ばれるメタデータで管理されていて、これの更新はシングルスレッドになっている. dynamic mappingを使っている場合、
新規indexの作成時に新たなmappingの作成も発生し、indexの数が多い場合は、これが大量に発生するために固まったような状態になるのだろう、ということ.

対策として、ここで教えてもらったように、翌日分のindexを事前に作成しておくようにした.

### フィールド名にドットが含まれている場合の問題

色々なログが入って来た結果、フィールド名にドットが含まれているものがあり、以下のようなエラーが発生

```
[2016-11-10 11:30:06,915][DEBUG][action.admin.indices.mapping.put] [ESHOST1] failed to put mappings on indices [[some.log-2016.11.10]], type [fluentd]
MapperParsingException[Field name [somefield.channel] cannot contain '.']
        at org.elasticsearch.index.mapper.object.ObjectMapper$TypeParser.parseProperties(ObjectMapper.java:277)
        at org.elasticsearch.index.mapper.object.ObjectMapper$TypeParser.parseObjectOrDocumentTypeProperties(ObjectMapper.java:222)
        at org.elasticsearch.index.mapper.object.ObjectMapper$TypeParser.parse(ObjectMapper.java:197)
        at org.elasticsearch.index.mapper.object.ObjectMapper$TypeParser.parseProperties(ObjectMapper.java:309)
        at org.elasticsearch.index.mapper.object.ObjectMapper$TypeParser.parseObjectOrDocumentTypeProperties(ObjectMapper.java:222)
        at org.elasticsearch.index.mapper.object.RootObjectMapper$TypeParser.parse(RootObjectMapper.java:139)
        at org.elasticsearch.index.mapper.DocumentMapperParser.parse(DocumentMapperParser.java:118)
        at org.elasticsearch.index.mapper.DocumentMapperParser.parse(DocumentMapperParser.java:99)
        at org.elasticsearch.index.mapper.MapperService.parse(MapperService.java:549)
        at org.elasticsearch.cluster.metadata.MetaDataMappingService$PutMappingExecutor.applyRequest(MetaDataMappingService.java:257)
        at org.elasticsearch.cluster.metadata.MetaDataMappingService$PutMappingExecutor.execute(MetaDataMappingService.java:230)
        at org.elasticsearch.cluster.service.InternalClusterService.runTasksForExecutor(InternalClusterService.java:468)
        at org.elasticsearch.cluster.service.InternalClusterService$UpdateTask.run(InternalClusterService.java:772)
        at org.elasticsearch.common.util.concurrent.PrioritizedEsThreadPoolExecutor$TieBreakingPrioritizedRunnable.runAndClean(PrioritizedEsThreadPoolExecutor.java:231)
        at org.elasticsearch.common.util.concurrent.PrioritizedEsThreadPoolExecutor$TieBreakingPrioritizedRunnable.run(PrioritizedEsThreadPoolExecutor.java:194)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)
```

ドットを含むフィールド名の扱いはバージョンによって違っていて、1.xでは可能だったが、2.xでは不可になった.
https://www.elastic.co/guide/en/elasticsearch/reference/2.0/breaking_20_mapping_changes.html#_field_names_may_not_contain_dots

その後、2.4以降では起動時のオプションに`-Dmapper.allow_dots_in_name=true`を付けると、ドットを含むフィールド名も入るようになった.
https://www.elastic.co/guide/en/elasticsearch/reference/2.4/dots-in-names.html

そして、5.0以降では再びドットを含むフィールドがOKになった. 今使っているElasticsearchは2.4.0なので、オプションで解決することもできたが、
開発者がドットを含まない形にログの形式を変更してくれることになった. 

### Kibanaでのindex pattern作成が面倒

kafka-fluentd-consumerは、Kafkaから読み込むtopicを正規表現のパターンで指定することができて、fluentd tag=topicが増えた場合は自動的に追随してくれる.
なので、ログの種類が増えてもElasticsearchまでは自動的に格納される.

が、Kibanaで見るためにはindex patternの設定が必要になる. これを毎回やるのは面倒なので、どうにかしたい.

同じような要望はあって、この操作をAPIでできるようにして欲しい、というissueが存在している. https://github.com/elastic/kibana/issues/5199

結構長く存在するissueだが、対応されてKibana5ではできるようになるらしい.

が、Kibana5はつい先日GAが出たばかり. 使っているのはまだKibana4だ. Kibana4では、index patternを始めKibana上で作ったオブジェクトの定義は、.kibanaというindexに格納されているので、
これを自分で作る方針にする.

.kibanaに格納されているindex patternの定義は、例えばこんな感じ.

```
  {
      "_index": ".kibana",
      "_type": "index-pattern",
      "_id": "log.hiveserver2-*",
      "_version": 2,
      "_score": 1,
      "_source": {
          "title": "log.hiveserver2-*",
          "timeFieldName": "@timestamp",
          "fields": "[{"name":"_index","type":"string","count":0,"scripted":false,"indexed":false,"analyzed":false,"doc_values":false},{"name":"_source","type":"_source","count":0,"scripted":false,"indexed":false,"analyzed":false,"doc_values":false},{"name":"tag_key","type":"string","count":0,"scripted":false,"indexed":true,"analyzed":false,"doc_values":true},{"name":"message","type":"string","count":0,"scripted":false,"indexed":true,"analyzed":false,"doc_values":true},{"name":"hostname","type":"string","count":0,"scripted":false,"indexed":true,"analyzed":false,"doc_values":true},{"name":"@timestamp","type":"date","count":0,"scripted":false,"indexed":true,"analyzed":false,"doc_values":true},{"name":"_id","type":"string","count":0,"scripted":false,"indexed":false,"analyzed":false,"doc_values":false},{"name":"_type","type":"string","count":0,"scripted":false,"indexed":false,"analyzed":false,"doc_values":false},{"name":"_score","type":"number","count":0,"scripted":false,"indexed":false,"analyzed":false,"doc_values":false}]"
      }
  }
```

fieldsの中身がちょっと面倒. 基本的には、既存のindexのmappingを取得し、Kibanaのソースコードを参考にしながら定義を作る. 

https://github.com/elastic/kibana/blob/4.6/src/ui/public/index_patterns/_map_field.js
https://github.com/elastic/kibana/blob/4.6/src/ui/public/index_patterns/_cast_mapping_type.js

例えば、`indexed`フィールドは、mappingの中の`index`フィールドが存在しない、または`no`のときのみ`false`として、その他の場合は`true`にする. とか、ElasticsaerchとKibanaでは型が違うので、long→numberみたいに変換する、とか.

これも、日次のバッチで回すようにした

## まとめ

* こんな感じで様々なログを、Elasticsearch+Kibanaで見ることができるようになった.
* fluentd→Elasticsearchについては、以下の部分がキャパシティ的に問題になりやすい印象
  * fluentd: 流量が多いと普通にCPUが上がって詰まる. 対策として、プロセスを増やして負荷分散する
  * Elasticsearch: 流量よりは、index数(shard数)が増えることに対して注意がいる. 流量はスケールアウトで対応できるが、cluster state管理の部分はスケールしない. 今回は、日付が切り替わった際の翌日分index作成で問題がおきたので、事前にindexを作成することで対応した. できるだけ、index数は多くなりすぎないように設計した方が良いと思う.
  
