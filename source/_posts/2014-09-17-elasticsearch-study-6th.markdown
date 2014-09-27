---
layout: post
title: "第6回Elasticsearch勉強会"
date: 2014-09-17 00:19:06 +0900
comments: true
categories: ["elasticsearch"]
---

第6回Elasticsearch勉強会に参加したので、記憶が新しいうちにメモ. 内容は資料を見れば分かると思うので、個人的に特に記憶に残ったポイントと、感想のみ.

## Aggregationあれこれ @johtani 氏

* ElasticsearchのAggregation機能を使うと、SQLで言う集計 group by 的なものができると、という話
* 動きとしては、各shard内でaggregateし、最後にその結果を集約する、という形. 普通のqueryと変わらない
* クエリの種類によっては、正確な値ではなく近似値だったりする
* Field Collapsingという、URLごとにtop Nを検索するようなこともできる
* 個人的な感想
  * 便利な機能なので、是非使いたい. 現状だとKibana3で使えないのがネックだけど、Kibana4だと使えるようになるらしいので期待

## 秒間3万の広告配信ログをElasticSearchで リアルタイム集計してきた戦いの記録 @satully 氏

* [資料](http://www.slideshare.net/Satully/elasticsearch-study6threaltime20140916)
* 商用での利用事例、という意味ですごく興味深かった
* DSPの配信システムで利用. 最大秒間30000ログ. 1.5TB/日くらい
* 各サーバのログ -> fluentd -> Elasticsearch -> MySQL/Redis という構成
* fluentdは10インスタンス
* ElasticsearchはCoordinate:2ノード, Search:2ノード, Data:28ノード.全てAWSのr3.largeインスタンス. メモリは30GBくらい？
  * diskは途中でSSDに変えた
  * CoordinateとSearchを分離する、というのは世間で見かけるけど、実際に負荷をみると必要性が疑問
  * 1日1index, 1index12shard, 1レプリ.
* 単純なログのinsertのみではなく、Bidと紐付けるべきログが来ると、元のBidのログに項目を追加する. fluentd pluginとして実装
* 運用ツールとしては、Elasticsearchのpluginとしてhead, bigdesk, ElasticHQ. 死活監視やfluentdのメトリックにはZABBIX
* 個人的な感想
  * サーバスペック、台数、流量の具体的な事例としてとても参考になった
  * indexサイズはかなり大きくなるけど、いけるものなんだ.
  * お金の計算に関わる部分に、fluentdとかElasticsearchとか使っている、というのに驚いた

## Elasticsearch 日本語スキーマレス環境構築と、ついでに多言語対応 @9215 氏

* [資料](https://speakerdeck.com/kunihikokido/elasticsearch-ri-ben-yu-sukimaresuhuan-jing-gou-zhu-to-tuideniduo-yan-yu-dui-ying)
* スキーマをちゃんと設計して、それに合わせてマッピング定義するのは面倒. どんなフィールドでどんなマッピングが最適、とか個別に設計すると人材がスケールしない.
* そこで、dynamic templateとindex templateを使い、ノウハウが必要な部分はtemplateに集約、各利用者(データを投入する人)にはフィールド名のネーミングルールを周知する、というアプローチにした
* 例えば、利用する言語を"language"というフィールドの値として設定し、template側でそれに合わせて適切なanalyzerを定義しておくことで、多言語に対してもうまくindexできる.
* 個人的な感想
  * 確かに、どんな型にして、どんなanalyzerにして、とかはノウハウの部分になるので、それをtemplateに集約するのはとても良いアプローチだと思った

## elasticsearchソースコードを読みはじめてみた @furandon_pig 氏
* Elasticsearchのソースコードを読み始めてみた
* 開発者の人には、はじめにREST APIを受けて検索する部分の動作をみてみるといいよ、と言われたけど、それがどこか分からなかったので、起動の部分を追いかけてみた話
* 個人的な感想
  * ちょうど、そろそろソースみないと、と思っていたのでそのとっかかりとして大変参考になった

## (LT)Elasticsearch RerouteAPIを使ったシャード配置の制御 @pisatoshi 氏

* [資料](https://speakerdeck.com/pisatoshi/elasticsearch-rerouteapiwoshi-tutasiyadopei-zhi-falsezhi-yu)
* Reroute APIを使って、具体的にどのシャードをどこに配置するとかの制御ができるよ、という話
* リバランスを有効にしていると、APIでシャードを別のノードに配置しても、リバランスされてしまうので注意
* 個人的な感想
  * オチがとてもおもしろかった.
  * Reroute APIって、制御できるのは良いけど、逆にいちいち制御するのはやってられないし、どーいう場面で使うのだろう、と思った.

## (LT)検索のダウンタイム0でバックアップからIndexをリストアする方法 @k_bigwheel 氏

* [資料](http://www.slideshare.net/kbigwheel/0index-39143333)
* indexのスナップショットを取れるようになったけど、普通にやるとリストア対象のindexを一旦closeしないといけない. サービスの継続を考えると、ダウンタイム無しでやりたい、という話
* 予めクライアントからはalias経由でアクセスするようにしておく. aliasの切り替えはatomicにできるので、それを利用して、リストア完了後にスッパリ切り替える、という形にする
* 個人的な感想
  * リストアだけでなく、mappingを変えるためにindexしなおすとか、aliasを使うと便利な場面はありそう
  * しかし、自分の環境はfluentdで"name-YYYY.MM.DD"の形でindexを作っているので、この場合aliasはどうしたらいいんだろう

## 全体所感

* 最近、Elasticsearchを真面目に使い始めてみたので、どのトークも大変参考になった. 特に、 @satully 氏の話は良かった
* 結構、みんなAWSなんだな
* 懇親会で、発表についてのもう少し詳しい話とか、他の人がどんな感じに使ってるとか聞けたので、そっちも良かった. しかも参加費無料. 素晴らしい.
* 主催の@johtaniさん、会場提供のリクルートテクノロジーズさんに感謝.
