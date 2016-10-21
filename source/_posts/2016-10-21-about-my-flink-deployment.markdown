---
layout: post
title: 現在稼働しているFlinkクラスタについて
date: 2016-10-21 22:35:50 +0900
comments: true
categories: [Flink,Streaming]
---
先日の[発表](http://www.slideshare.net/linecorp/b-6-new-stream-processing-platformwith-apache-flink)で、Apache Flinkを導入するに至った経緯を話したのだけど、具体的な構成とかには触れられなかったので書いておく。

## クラスタの構成について

今運用してるFlinkクラスタは２つ。サービスで使うためのデータを生成しているものと、社内のレポーティングやモニタリングで使っているもの。前者の方は安定性重視、後者は割とカジュアルにジョブを追加したり、構成を弄ったりできるもの、という感じになっている.

Flinkとしては、クラスタのデプロイメント方式として、独立したdaemonとして動かす方法と、YARNの上で動かす方法があるのだけど、前者の方法にしている. その方が運用上もわかりやすいし、レイヤが少ない分トラブルも少ないだろう、というのが理由.

どちらも物理サーバで、TaskManagerサーバは前者が3台、後者が10台になっている. Flinkのバージョンはそれぞれ1.0.3と1.1.1. JobManagerはもちろんHAで、Flinkが使うHDFSやZookeeperは既存Hadoopクラスタを共用している.

## 周辺の構成

Flinkへのインプットはfluentdで集めたログをKafkaに投入したもの。もともとログはがっつりfluentdで集めてたので、Kafkaを導入して、そちらにも飛ばすようにした。ちなみに、fluentd→kafkaの部分について、導入当初はfluent-plugin-kafkaが使っていたposeidon gemがメンテナンス停止を宣言したりしてて若干不安があったけど、その後poseidonはruby-kafkaに置き換えられ、pluginもfluentdの公式になったりして、安心感が大分増した。開発者の方々には感謝してます。

アウトプットはRedis, Elasticsearch, Kafkaの3パターンがある. 汎用的に使いやすいのはElasticserachで、Kibanaで見たり、集計結果をAPIサーバ経由で他に提供したりしている. Redisは最新の情報にしか興味がなくて、かつ更新頻度が高い場合に使っている. 特定のキーの値を10秒間隔でアップデート、とか. Kafkaは、集計結果を他と連携したいケース. 今は、[kafka-topic-exporter](https://github.com/ogibayashi/kafka-topic-exporter) 経由で集計結果を[Prometheus](https://prometheus.io/)に入れ、[Grafana](http://grafana.org/)で見るために使っている. 

## 運用周り

モニタリングは基本的にPrometheus. [flink-exporter](https://github.com/matsumana/flink_exporter)と[jmx-expoter](https://github.com/prometheus/jmx_exporter)を使ってメトリクスをPrometheusに送り、Grafanaで見ている.

Flinkのメトリクスについては https://ci.apache.org/projects/flink/flink-docs-release-1.1/apis/metrics.html に記載があるが、JMXで見るためには、flink-conf.yamlに以下の設定を追加する. なお、これが使えるのはFlink 1.1以降.

```
metrics.reporters: jmx_reporter
metrics.reporter.jmx_reporter.class: org.apache.flink.metrics.jmx.JMXReporter
env.java.opts: -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.port=5560
```

jmx_expoterの設定は、[こんな感じ](https://gist.github.com/ogibayashi/26c976442592b285fd0ff6b07b08ec60) のものを使っている

すると、こんな感じでFlinkのKafkaConsumerのlagやconsumeされているレコード数を見ることができる.

![Flink_Grafana_Kafka](/images/2016/grafana_flink_kafka.png)

以下は、あるジョブのチェックポイントのサイズ. 長期のスパンで見ると、チェックポイントサイズが増加傾向なので何かしら不要なstateがpurgeされずに溜まっていることが疑われる

![Flink_Grafana_Checkpoint](/images/2016/grafana_flink_checkpoint.png)

監視は外部からdaemonのTCPポートが空いていることを確認している. JobManagerはWebUIポート(8081)を見れば良いが、TaskManagerのポートは起動するたびに変わるので、以下のような設定を入れて固定している.

```
taskmanager.rpc.port: 6122
taskmanager.data.port: 6121
```

Flink的なメトリクスとしては、Exceptionの発生数とか、ジョブが失敗や再起動状態にあるジョブの数とかも使ってアラートを上げた方がいいんだろうなぁ、と思いつつまだやっていない

## まとめ

今productionで動かしているFlinkクラスタについて、構成や運用周りを紹介してみた. これからFlinkを使ってみよう、という人の参考になれば幸いです. 
