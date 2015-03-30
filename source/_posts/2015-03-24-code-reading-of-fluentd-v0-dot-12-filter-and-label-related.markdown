---
layout: post
title: FluentdのFilter&Label周りのコードを読んだメモ
date: 2015-03-24 13:45:46 +0900
comments: true
published: false
categories: [Fluentd]
---

Fluentd v0.12の目玉機能としてFilterとLabelがある. この機能の導入にあたってはメッセージのルーティングを行う部分のコードがガラリと変わっているはずなので、興味本位で読んでみた.

## 機能についての参考文書

そもそもFilterやLabelって何？というあたりは以下が参考になる。

* [Fluentd v0.12でのFilterとLabel](http://repeatedly.github.io/ja/2014/08/fluentd-filter-and-label/)
* [Fluentd v0.12 is Released](http://www.fluentd.org/blog/fluentd-v0.12-is-released)
* [Fluentd v0.12の目玉機能らしいFilterを試してみた](http://qiita.com/muran001/items/f73d19398b750abb9c2d)
* [Fluentd v0.12 ラベル機能の使い方とプラグインの改修方法](http://qiita.com/sonots/items/a01d2233210b7b059967)

## v0.10ではどうだったか

Matchクラスが`match`ディレクティブで宣言されたタグのパターンと、行き先のOutputクラスを保持していて、`EngineClass#emit` (最終的な呼び出し先は`emit_stream`)で該当するMatchを探し出し、そこに向けて`emit`する、という形だった.

```ruby
  def emit_stream(tag, es)
      target = @match_cache[tag]
      unless target
        target = match(tag) || NoMatchMatch.new
        # this is not thread-safe but inconsistency doesn't
        # cause serious problems while locking causes.
        if @match_cache_keys.size >= MATCH_CACHE_SIZE
          @match_cache.delete @match_cache_keys.shift
        end
        @match_cache[tag] = target
        @match_cache_keys << tag
      end
      target.emit(tag, es)
```

なので、ルーティングを管理するテーブルは`EngineClass`の`@matches`一つだったし、tagがルーティングのキーになっていた. 複数のOutputで順番に処理したい場合は都度tagを書き換えていく必要があった. 

## v0.12の場合

`Agent`, `RootAgent`, `Label`, `EventRouter`と言った新しいクラスが導入されている.

* `Label`
    * 各`label`ディレクティブの中に存在するFilterおよびOutputプラグインを管理するクラス
* `RootAgent`
    * `label`ディレクティブに属さない(設定ファイルのルート直下にある)Input, Filter, Outputプラグイン、および各Labelクラスを管理するクラス
* `Agent`
    * `RootAgent`および`Label`の親クラス. 
* `EventRouter`
    * ルーティングのためのルール(どのタグパターンに対して、どのようなfilterやmatchが存在するか)を管理し、イベントのルーティングを行うクラス

`RootAgent`および`Label`のインスタンスは、それぞれ自分自身が管理する範囲のルーティングを行う`EventRouter`クラスのインスタンスを保持している.

以下、実際のコードを見てみる.

## 起動部分

まずは、configurationを読み込んでいく段階でどのようなクラスが生成されていくのかを見てみる.

### Supervisor#start (init_engine)

[Supervisor#start](https://github.com/fluent/fluentd/blob/f6aa9a4b275e4e9885bb912bcf0e930966d8e246/lib/fluent/supervisor.rb#L124)

Supervisorが起動して、色々と準備していく部分. `init_engine`内で [Engine#init](https://github.com/fluent/fluentd/blob/f6aa9a4b275e4e9885bb912bcf0e930966d8e246/lib/fluent/engine.rb#L41) が呼ばれ、ここで`RootAgent`が生成される.

`RootAgent`の親クラスである`Agent`のコンストラクタは以下のようになっている

[Agent#initialize](https://github.com/fluent/fluentd/blob/f6aa9a4b275e4e9885bb912bcf0e930966d8e246/lib/fluent/agent.rb#L29)

```ruby
    def initialize(opts = {})
      super()

      @context = nil
      @outputs = []
      @filters = []
      @started_outputs = []
      @started_filters = []

      @log = Engine.log
      @event_router = EventRouter.new(NoMatchMatch.new(log), self)
      @error_collector = nil
    end

```

自身が管理するOutputクラス、Filterクラス達を保持するための変数が存在している. また、そのスコープでのルーティングを行う`EventRouter`クラスをここで生成している.

`RootAgent`については、これに加えて更にInputやLabelクラスも管理するような構造になっている. ([root_agent.rb](https://github.com/fluent/fluentd/blob/f6aa9a4b275e4e9885bb912bcf0e930966d8e246/lib/fluent/root_agent.rb#L46))

### Supervisor#start (run_configure)

`Supervisor#run_conigure`が呼ばれると、`Engine#configure`を経由して`RootAgent#configure`が呼ばれる.

[RootAgent#configure](https://github.com/fluent/fluentd/blob/f6aa9a4b275e4e9885bb912bcf0e930966d8e246/lib/fluent/root_agent.rb#L62)

ちょっと長いが引用.これにより以下が行われる

* labelディレクティブがあった場合
    *  [add_label](https://github.com/fluent/fluentd/blob/f6aa9a4b275e4e9885bb912bcf0e930966d8e246/lib/fluent/root_agent.rb#L155)により新規`Label`オブジェクトを生成
    *  さらに、その`Label`オブジェクトの`configure`を呼び出す. `configure`の内容については、以下の`Agent#configure`を参照.
* sourceディレクティブがあった場合
    * [add_source](https://github.com/fluent/fluentd/blob/f6aa9a4b275e4e9885bb912bcf0e930966d8e246/lib/fluent/root_agent.rb#L141)によりInputプラグインのインスタンスを生成
    * Inputプラグインがemitする際の投げ先として、以下を登録.
        * そのInputプラグインで`@label`が設定されている場合→設定された`Label`オブジェクトの`EventRouter`を登録
        * それ以外の場合→`RootAgent`の`EventRouter`を登録

Inputプラグインについては`@label`が設定されている場合とそうでない場合で、emit先の`EventRouter`を切り替えることができるようになっている.

```ruby
    def configure(conf)
      error_label_config = nil

      # initialize <label> elements before configuring all plugins to avoid 'label not found' in input, filter and output.
      label_configs = {}
      conf.elements.select { |e| e.name == 'label' }.each { |e|
        name = e.arg
        raise ConfigError, "Missing symbol argument on <label> directive" if name.empty?

        if name == ERROR_LABEL
          error_label_config = e
        else
          add_label(name)
          label_configs[name] = e
        end
      }
      # Call 'configure' here to avoid 'label not found'
      label_configs.each { |name, e| @labels[name].configure(e) }
      setup_error_label(error_label_config) if error_label_config

      super

      # initialize <source> elements
      if @without_source
        log.info "'--without-source' is applied. Ignore <source> sections"
      else
        conf.elements.select { |e| e.name == 'source' }.each { |e|
          type = e['@type'] || e['type']
          raise ConfigError, "Missing 'type' parameter on <source> directive" unless type
          add_source(type, e)
        }
      end
    end

```

さらに、`RootAgent`の親クラスである`Agent#configure`では同様にmatchに対して`add_match`、filterについて`add_filter`が呼ばれる.

[Anget#configure](https://github.com/fluent/fluentd/blob/f6aa9a4b275e4e9885bb912bcf0e930966d8e246/lib/fluent/agent.rb#L50)

```ruby
    def configure(conf)
      super

      # initialize <match> and <filter> elements
      conf.elements.select { |e| e.name == 'filter' || e.name == 'match' }.each { |e|
        pattern = e.arg.empty? ? '**' : e.arg
        type = e['@type'] || e['type']
        if e.name == 'filter'
          add_filter(type, pattern, e)
        else
          add_match(type, pattern, e)
        end
      }
    end

```

`Agent`は`Label`の親クラスでもあるので、新しい`Label`オブジェクトの`configure`が呼び出された際もこのコードが実行されることになる.

[add_filter](https://github.com/fluent/fluentd/blob/f6aa9a4b275e4e9885bb912bcf0e930966d8e246/lib/fluent/agent.rb#L134)や[add_match](https://github.com/fluent/fluentd/blob/f6aa9a4b275e4e9885bb912bcf0e930966d8e246/lib/fluent/agent.rb#L122)が何をしているかというと、その`Agent`が持っている`EventRouter`に対してルーティングのルール(`Rule`オブジェクト)を登録している. 

### 絵にすると、、、

[FluentdのBlog](http://www.fluentd.org/blog/fluentd-v0.12-is-released)に書かれているサンプルを元に、どんな感じのオブジェクトたちが出来上がるかを絵にするとこんな感じ.

![RootAgent](/images/2015/201503_Fluentd_Routing.png)

labelごとにルーティングテーブルを持つので、labelが違えば異なるルールでルーティングする、ということができるようになる.

## emitの動き

ここまでで、各Input, Filter, Outputプラグインインスタンスは、自分がemitする先の`EventRouter`オブジェクトを知っていることになる.

まず、Inputプラグイン内では自分が知っている`EventRouter`に`emit`する.

```ruby
router.emit(tag, time, record)
```

`emit`は`emit_stream`に飛ぶので、以下のコードが呼び出される. `match`メソッドが返してきたオブジェクトに対して`emit`する.

[EventRouter#emit_stream](https://github.com/fluent/fluentd/blob/f6aa9a4b275e4e9885bb912bcf0e930966d8e246/lib/fluent/event_router.rb#L87)

```ruby
    def emit_stream(tag, es)
      match(tag).emit(tag, es, @chain)
    rescue => e
      @emit_error_handler.handle_emits_error(tag, es, e)
    end

```

[EventRouter#match](https://github.com/fluent/fluentd/blob/f6aa9a4b275e4e9885bb912bcf0e930966d8e246/lib/fluent/event_router.rb#L101) に飛ぶ. `match`はemitされたtagを受け取るべきCollectorを返す.

```ruby
    def match(tag)
      collector = @match_cache.get(tag) {
        c = find(tag) || @default_collector
      }
      collector
    end
```

このCollectorを探す部分がどうなっているかと言うと、

[event_router#find](https://github.com/fluent/fluentd/blob/f6aa9a4b275e4e9885bb912bcf0e930966d8e246/lib/fluent/event_router.rb#L156)

こんな感じになっている. つまり、Filterが使われていれば`Pipeline`オブジェクトを生成してそこにFilterやOutputを順次追加していく. Filterがなければ`Pipeline`の代わりにOutputを直接返す.

```ruby
    def find(tag)
      pipeline = nil
      @match_rules.each_with_index { |rule, i|
        if rule.match?(tag)
          if rule.collector.is_a?(Filter)
            pipeline ||= Pipeline.new
            pipeline.add_filter(rule.collector)
          else
            if pipeline
              pipeline.set_output(rule.collector)
            else
              # Use Output directly when filter is not matched
              pipeline = rule.collector
            end
            return pipeline
          end
        end
      }

      if pipeline
        # filter is matched but no match
        pipeline.set_output(@default_collector)
        pipeline
      else
        nil
      end
    end
```

よって、`emit`されたレコードは`Pipeline`または`Output`に`emit`されることになる. そして、`Pipeline`に`emit`された場合は以下のコードに辿り着き、順番にFilterを通った後に最終的にOutputに`emit`されることになる.

[Pipeline#emit](https://github.com/fluent/fluentd/blob/f6aa9a4b275e4e9885bb912bcf0e930966d8e246/lib/fluent/event_router.rb#L147)

```ruby
      def emit(tag, es, chain)
        processed = es
        @filters.each { |filter|
          processed = filter.filter_stream(tag, processed)
        }
        @output.emit(tag, processed, chain)
      end
```

このようにFilterを実現するためにPipelineという新しい仕組みを導入しているため、tagの書き換えによる多段フィルタをしなくて済むようになっている.


## まとめ

* v0.12のLabel, Filterを実現している部分のコードを読んでみた
* Labelの部分は、RootAgent(設定ファイルのROOT部分)および各labelディレクティブごとにルーティングテーブル(`EventRouter`)を分けることにより実現されている
* Filterは、レコードに対する連続した処理を表現する、Pipelineという新たな仕組みを導入することで実現されている
