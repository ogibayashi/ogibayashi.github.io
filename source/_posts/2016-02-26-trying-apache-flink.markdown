---
layout: post
title: Apache Flinkを試している
date: 2016-02-26 20:38:01 +0900
comments: true
categories: [Flink]
---
耐障害性と拡張性のあるストリーム処理基盤が欲しい、と思って[Apache Flink](https://flink.apache.org/)を調べている. 今はリアルタイム集計に[Norikra](http://norikra.github.io/)を使っていて、これはとてもカジュアルに使えて良いのだけど、以下の様なケースだと難しい。

* 比較的止めたくない処理で、サーバ障害時にも自動的に回復して欲しい
* 1日とか長いtime windowの集計をしているので、途中でサーバが落ちて集計中の状態が失われると辛い
* トラフィックが増えてきて、複数サーバに負荷を分散したい
* 例えばストリームに含まれているIDに対応する値を外部のテーブルから取ってくるような、ちょっと複雑な処理をしたい

### Flinkとはどのようなソフトウェアか

一言で言うと、対障害性と拡張性を備えた、分散ストリーム処理基盤。バッチ処理もストリーム処理の仕組みでできるよね、ということでバッチ用、ストリーム用両方のAPIが提供されている。実行環境としては、Hadoop等と同じようにワーカプロセスが複数のサーバで立ち上がっていて、そこに対してジョブを投げるような感じになっている。バッチではジョブを投げるとデータを処理して終了、だけど、ストリーム処理では投げたジョブはずっと生きていて、ストリームにデータが流れてくるたびにそのデータを処理して出力する、という形になる。ジョブは、JavaとScalaで書くことができる。ロードマップにはSQL的なものもあるっぽいけど、今は存在しない。

### SparkとかStormと何が違うの？

ググると色々出てくるが、[米Yahooのベンチマーク](https://yahooeng.tumblr.com/post/135321837876/benchmarking-streaming-computation-engines-at)とか、[このページ](http://www.altaterra.net/blogpost/288668/225612/Which-Stream-Processing-Engine-Storm-Spark-Samza-or-Flink)が分かりやすかった。

Stromとの比較で言うと、処理形態はほぼ同じ。ただ、大きな違いとしてStormでは各処理オペレータはstatelessになっていて、落ちると状態が失われる。永続化したかったら、自分で外部ストレージを使うようなコードを書く必要がある。耐障害性ということろだと、各レコードに対して処理が完了した際にackを返す仕組みがあるので、障害などでackが無かったら再度処理する、というような処理を書く感じになる。なので、例えば処理が途中まで行われて障害になると、そのレコードは重複して処理される形になる。Flinkは[ここ](https://ci.apache.org/projects/flink/flink-docs-master/internals/stream_checkpointing.html)に詳しく書いてあるけど、オペレータがstatefulになっていて、チェックポイントごとに状態が保存される。障害時には、チェックポイントから復旧し、それ以降のレコードをリプレイすることで、exactly-onceを実現している。

あとは、Stormだと各処理オペレータ(Bolt)をJavaのクラスとして書かないといけないけど、Flinkは`stream.keyBy(...).map{...}.timeWindow(...)`みたいな感じのハイレベルのAPIが提供されている。

Spark Streamingとの違いで言うと、Spark Streamingは正確にはストリーミングではなくてマイクロバッチなので、そのバッチの間隔にストリーム処理のwindowが左右される。sliding time windowとか、一定数のレコードを保持するようなwindowは作ることができない。(ちゃんとドキュメント読んでないけど、そのはず)

### 触ってみた

とりあえず触ってみるには、https://ci.apache.org/projects/flink/flink-docs-release-0.10/quickstart/setup_quickstart.html に従えば良い。

ざっくりした流れとしては、

* バイナリをダウンロードして展開
* `./bin/start-local.sh` 実行
* http://localhost:8081/ でJobManagerを開く
* exampleがバイナリと一緒に配布されているので、それを実行

という感じで試すことができる。

クラスタを組むのも割と簡単で、 https://ci.apache.org/projects/flink/flink-docs-release-0.10/setup/cluster_setup.html に従えばできる。

JobManagerというのがマスタで、TaskManagerというのがワーカになっているので、これをサーバに分散配置することになる。

流れとしては、

* 各サーバにflink用のユーザとssh keyを用意
  * ssh keyは、起動スクリプトの中でssh経由でTaskManagerを起動するので必要.自分で各サーバのTaskManagerを起動して回るなら無くても良い
* 各サーバにバイナリを配布
* 設定ファイル(flink-conf.yaml, slaves)を用意
  * flink-conf.yamlは、配布されているものをそのまま使って、JobManagerのアドレスを設定すれば良い
  * slavesはTaskManagerホストを列挙したもの。クラスタ起動スクリプトの中で使われる

となる。
  ジョブを書くには、https://ci.apache.org/projects/flink/flink-docs-release-0.10/quickstart/scala_api_quickstart.html に従う。mavenのアーキタイプが用意されてるので、それでプロジェクトのテンプレートを作って、ドキュメントの中にあるWordCountをコピーして走らせれば良い。

### 感想

試しにfluend→Kafka→Flink→Elasticsearch、とつなげて分散実行してみたり、プロセスを落としてみたりしたが、期待通りに動作した。例えば、5分間のtime windowで件数を集計するような処理を作って、実行中にプロセスを落とすと、障害が検知されてジョブが再実行される。そして、勝手に必要なリカバリがされるので、障害があっても無くても実行結果は変わらない。すごい。(もう少しちゃんと見ると、ずれるケースもありそうだけど、まだ詳しく見ていない)

並列度は、ジョブの投入時に指定できるので、ちゃんとKafkaのパーティションを複数作って並列度を指定して実行すれば、各TaskManagerに分散して実行される。

ただ、性能のところが今一つで、処理を3サーバに分散、5,000msg/secを投入してやってみたら、各サーバ1CPUを100%使って、Kafkaにメッセージが滞留する状態になった。実際に処理できたのは4,300msg/secくらい。3CPUで4,300msg/secだと大分コストが高いなあ、という印象。まあ、とりあえず動かしてみただけなので、何か正しくない可能性はある。もう少し試してみたい。
