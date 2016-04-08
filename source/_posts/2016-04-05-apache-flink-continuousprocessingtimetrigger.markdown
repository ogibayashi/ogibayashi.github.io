---
layout: post
title: Apache Flink ContinuousProcessingTimeTriggerの話
date: 2016-04-05 11:33:23 +0900
comments: true
categories: [Flink]
---

(注: window周りのAPIを変えようという話([FLINK-3643](https://issues.apache.org/jira/browse/FLINK-3643))があるので、以下で書かれている内容は近い将来obsoleteになる可能性があります)

自分のところでは、ストリーム処理のユースケースとして、ある程度長期間のwindowでデータを保持しながら、集計結果は短い間隔で出力したい、というのがある。

例えば、Norikra(Esper)のクエリだとこんな感じ。site_idごとに、過去3日間(72時間)のユニークユーザを集計し、結果は1分ごとに出力する、というもの。 集計対象のwindowは72時間のsliding time windowとなる。こういうwindowが長いケースは、Norikraだとプロセスを再起動したりした時にwindowの内容が全て失われるので辛い。Flinkでいい感じに実装したい。

```sql
SELECT
    site_id,
    count(distinct userkey) as num_uu
FROM
    viewer_log.win:time(3 day)
GROUP BY
    site_id
OUTPUT LAST EVERY 1 min
```

[ドキュメント](https://ci.apache.org/projects/flink/flink-docs-release-1.0/apis/streaming/windows.html)を読むと、以下のようにwindowの定義とは別にいつ計算処理がトリガされるか、いつwindowの中身が削除されるか、を好きに設定できるように見える。

```scala
keyedStream
    .window(SlidingEventTimeWindows.of(Time.seconds(5), Time.seconds(1))
    .trigger(CountTrigger.of(100))
    .evictor(CountEvictor.of(10));
```

なので、上記のようなものは以下の様なコードで実現できるかと思った。ContinuousProcessingTimeTriggerは、指定した間隔で集計を実行するが、その際にwindowの内容の削除は行わない、というもの。

```scala
  def main(args: Array[String]) {
//...
    val input = env.readFileStream(fileName,100,FileMonitoringFunction.WatchType.PROCESS_ONLY_APPENDED)
      .map(lineToTuple(_))
      .keyBy(0)
      .timeWindow(Time.of(3, TimeUnit.DAYS))
      .trigger(ContinuousProcessingTimeTrigger.of(Time.seconds(60)))
      .fold(("",new scala.collection.mutable.HashSet[String]())){(r,i) => { (i._1, r._2 + i._2)}}
      .map{x => (new Timestamp(System.currentTimeMillis()), x._1, x._2.size)}
//...
  }
  def lineToTuple(line: String) = {
    val x = line.split("\\W+") filter {_.nonEmpty}
    (x(0),x(1))
  }

```

が、実際には期待どおりに動かないことが分かった。(MLで教えてもらった。[ここ](http://mail-archives.apache.org/mod_mbox/flink-user/201603.mbox/%3CCAMvK=YpGWZZQQnPVknGYY07AhieHE+9ELkq0i6Q_72FCghsm-Q@mail.gmail.com%3E)とか[ここ](http://mail-archives.apache.org/mod_mbox/flink-user/201603.mbox/%3CCAMvK%3DYo2NMk3Hxhie5Cr9coHS7qZ8ecaGOHHXg96mDB7H9_7Nw%40mail.gmail.com%3E))

* トリガが発火しないケースがある
* timeWindowが終了してもwindow内のデータが削除されない。結果、stateのサイズが増加する一方になる。(.trigger()を呼び出した時点で、元のtimeWindowのトリガは上書きされるらしい)

って、これContinuousProcessingTimeTriggerは使えないのでは・・・。

結局、ContinuousProcessingTimeTriggerに修正を入れたカスタムTriggerを作成し、これを使用することで期待する動作をするようになった。

```java
@PublicEvolving
public class MyContinuousProcessingTimeTrigger<W extends TimeWindow> extends Trigger<Object, W> {
	private static final long serialVersionUID = 1L;

	private final long interval;

	private final ValueStateDescriptor<Long> stateDesc =
			new ValueStateDescriptor<>("fire-timestamp", LongSerializer.INSTANCE, 0L);

    
	private MyContinuousProcessingTimeTrigger(long interval) {
		this.interval = interval;
	}

	@Override
	public TriggerResult onElement(Object element, long timestamp, W window, TriggerContext ctx) throws Exception {
		long currentTime = System.currentTimeMillis();

		ValueState<Long> fireState = ctx.getPartitionedState(stateDesc);
		long nextFireTimestamp = fireState.value();

		if (nextFireTimestamp == 0) {
			long start = currentTime - (currentTime % interval);
			fireState.update(start + interval);

			ctx.registerProcessingTimeTimer((start + interval));
			return TriggerResult.CONTINUE;
		}
		if (currentTime >= nextFireTimestamp) {
			long start = currentTime - (currentTime % interval);
			fireState.update(start + interval);

			ctx.registerProcessingTimeTimer((start + interval));

			return TriggerResult.FIRE;
		}
		return TriggerResult.CONTINUE;
	}

	@Override
	public TriggerResult onEventTime(long time, W window, TriggerContext ctx) throws Exception {
		return TriggerResult.CONTINUE;
	}

	@Override
	public TriggerResult onProcessingTime(long time, W window, TriggerContext ctx) throws Exception {

		ValueState<Long> fireState = ctx.getPartitionedState(stateDesc);
		long nextFireTimestamp = fireState.value();

		// only fire if an element didn't already fire
		long currentTime = System.currentTimeMillis();
		if (currentTime >= nextFireTimestamp) {
			long start = currentTime - (currentTime % interval);
                        fireState.update(0L);
                        if (nextFireTimestamp >= window.getEnd()) {
                           return TriggerResult.FIRE_AND_PURGE;
                        }
                        else {
                            return TriggerResult.FIRE;
                        }
                }
		return TriggerResult.CONTINUE;
	}

	@Override
	public void clear(W window, TriggerContext ctx) throws Exception {}

	@VisibleForTesting
	public long getInterval() {
		return interval;
	}

	@Override
	public String toString() {
		return "ContinuousProcessingTimeTrigger(" + interval + ")";
	}

	/**
	 * Creates a trigger that continuously fires based on the given interval.
	 *
	 * @param interval The time interval at which to fire.
	 * @param <W> The type of {@link Window Windows} on which this trigger can operate.
	 */
	public static <W extends TimeWindow> MyContinuousProcessingTimeTrigger<W> of(Time interval) {
		return new MyContinuousProcessingTimeTrigger<>(interval.toMilliseconds());
	}
}
```



