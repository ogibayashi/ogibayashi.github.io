---
layout: post
title: EsperのEPLでjava methodとGROUP BYを使ったらレコードが重複した話
date: 2015-08-13 10:19:29 +0900
comments: true
published: true
categories: [Esper, Norikra]
---
Norikraで以下の様な感じの、GROUP BYして、SELECTの中でjavaメソッドを使うようなクエリを登録していたのだけど、なんか生成されるイベントが多い、というのが発端で調べてみた.

```
SELECT str, str.split(",") as splitted, count(*) as cnt FROM str_stream.win:time_batch(10 sec) GROUP BY str
```

上記のクエリを登録した状態で、以下のようにイベントを投入する.

```
$ echo '{"str":"a,b,c"}' | norikra-client event send str_stream
$ echo '{"str":"a,b,c"}' | norikra-client event send str_stream
$ echo '{"str":"a,b,c"}' | norikra-client event send str_stream
```

と、こんな感じで同じようなレコードが重複して生成される. `count(*)`は正しくカウントされているのだが、GROUP BYしているので1レコードになってて欲しい.

```
$ norikra-client event sweep
{"time":"2015/07/31 12:06:52","query":"splittest","str":"a,b,c","splitted":["a","b","c"],"cnt":3}
{"time":"2015/07/31 12:06:52","query":"splittest","str":"a,b,c","splitted":["a","b","c"],"cnt":3}
{"time":"2015/07/31 12:06:52","query":"splittest","str":"a,b,c","splitted":["a","b","c"],"cnt":3}
```

Norikraではなく、Esperで同じことを実験してみても発生するので、Esperの事象っぽい.[公式ドキュメント](http://www.espertech.com/esper/release-5.2.0/esper-reference/html/processingmodel.html)の"3.7.2. Output for Aggregation and Group-By"を見ると、以下の様な記述がある。 GROUP BYの中に無いフィールドをSELECTすると、入力イベントと同じ数の出力イベントが発生するよ、という話.

> If your statement selects non-aggregated properties and aggregation values, and groups only some properties using the group by clause, your statement may look as below:
> 
>  select account, accountName, sum(amount)
> from Withdrawal.win:time_batch(1 sec)
> group by account
> 
> At the end of a time interval, the engine posts to listeners one row per event. The aggregation result aggregates per unique account.

とは言え、GROUP BYの中に`str.split(",")`を入れても同じ. 値は同じでも、Stringクラスのオブジェクトは別になってしまうからなのかなぁ. 一応、回避策としてはfirsteverという、イベントの中で最初の値だけを取る関数があるので、`firstever(str.split(",")) `のようにすると1レコードに集約できた.

Issueで聞いてみた.
https://github.com/espertechinc/esper/issues/18

若干うまく伝わらなかった気がするけど、javaメソッドではなく同等の機能を持つUDFを作ればうまくいくよ、ということらしい。実際、EsperのUDFを作ってみたら、ちゃんと集約された。

ちなみに、EsperのUDFを作るのは簡単で、以下のようにstaticなメソッドを持つクラスを作って


```java
public class MyUtilityClass {
    public static String[] split(String str, String regex){
        return str.split(regex);
    }
}
```

ConfigurationクラスのaddPlugInSingleRowFunctionメソッドで、function名、クラス名、メソッド名を渡してやれば良い。

```
addPlugInSingleRowFunction("split", MyUtilityClass.class.getName(), "split");
```

まあ、都度UDF作るのも面倒なので、firsteverで回避出来る場合はそうするかなぁ.

そんな話でした.
