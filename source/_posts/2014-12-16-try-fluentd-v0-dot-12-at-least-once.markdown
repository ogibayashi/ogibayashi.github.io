---
layout: post
title: "Fluentd v0.12のAt-least-once semanticsを試す"
date: 2014-12-16 00:34:17 +0900
comments: true
categories: [fluentd]
published: true
---
Fluentd v0.12のin/out_forwardでAt-least-once semanticsがサポートされるようになった. 今まではアプリケーションレイヤでの到達確認がなかったので、一部のネットワーク障害などのケースでは、送信されたように見えて実は送信されていない、という事象が発生し得た. v0.12から導入された`require_ack_response`オプションを使うと、このような事象を避けることができる.

この機能が導入されたpull requestはこちら.
[https://github.com/fluent/fluentd/pull/428](https://github.com/fluent/fluentd/pull/428)

ということで試してみた.

## require_ack_responseがない場合

fluentd 0.10.56で試す. (0.12で試しても良かったのだけど..)

送信側は以下の設定. 相手先fluentdが早々にdetachされてしまうのを避けるため、`hard_timeout`と`phi_threshold`を入れた

```
<source>
   type forward
</source>
<match test.**>
   type forward
   flush_interval 1s
   heartbeat_type tcp
   hard_timeout 600
   phi_threshold 300
   buffer_type file
   buffer_path /var/log/fluentd.*.buffer
   <server>
     host  192.168.1.2
     port 24224
   </server>
</match>

```

受信側はこんな感じ

```
<source>
  type forward
</source>

<match test.**>
  type file
  path /tmp/fluentd_forward.log
</match>

```

で、パケットが届かないが、アプリケーションにエラーが返らない状況を作るため、受信側のiptablesでSYNが立っていないパケットをドロップするようにする. SYNは相手に到達し、SYN-ACKも返るため、アプリケーションからは正常に接続されている様に見えることになる.

```
# iptables -A INPUT -p tcp --syn --dport 24224 -j ACCEPT
# iptables -A INPUT -p tcp --dport 24224 -j DROP
```

これでログを送ってみる

```
# echo '{"aaa": 1}' | fluent-cat  test.data
# echo '{"bbb": 2}' | fluent-cat test.data
```

netstatで送信側のソケットを見る. Send-Qにデータが溜まっている.

```
# netstat -na | grep 24224
tcp        0      0 0.0.0.0:24224               0.0.0.0:*                   LISTEN
tcp        0      1 192.168.1.1:10652          192.168.1.2:24224          FIN_WAIT1
tcp        0     41 192.168.1.1:10655          192.168.1.2:24224          FIN_WAIT1
udp        0      0 0.0.0.0:24224               0.0.0.0:*

```

しばらくすると、ソケットが破棄される.

```
# netstat -na | grep 24224
tcp        0      0 0.0.0.0:24224               0.0.0.0:*                   LISTEN      
tcp        0      1 192.168.1.1:10664           192.168.1.2:24224          FIN_WAIT1   
udp        0      0 0.0.0.0:24224               0.0.0.0:*             

```

この状況だと、アプリケーション的には正常に送れているように見えてしまうので、バッファは削除される. つまりログがロストした状況.

```
# ls /var/log/fluentd.*.buffer
ls: cannot access /var/log/fluentd.*.buffer: そのようなファイルやディレクトリはありません

```

## require_ack_responseを使う

次に、送受信共に`v0.12.1`にして、送信側に`require_ack_response`の設定を入れてみる.

```
  <source>
    type forward
  </source>
  <match test.**>
    type forward
    flush_interval 1s
    heartbeat_type tcp
    hard_timeout 600
    phi_threshold 300
    buffer_type file
    buffer_path /var/log/fluentd.*.buffer
    require_ack_response 
    <server>
      host 192.168.1.2
      port 24224
    </server>
  </match>

```

同様にfluent-catで送る. 今度は、一定時間後に以下のようにエラーになった.

```
2014-12-15 15:25:56 +0900 [warn]: no response from 192.168.1.2:24224. regard it as unavailable.
2014-12-15 15:26:56 +0900 [warn]: temporarily failed to flush the buffer. next_retry=2014-12-15 15:22:46 +0900 error_class="Fluent::ForwardOutputACKTimeoutError" error="node 10.29.254.66:24224 does not return ACK" plugin_id="object:16c7e3c"
  2014-12-15 15:26:56 +0900 [warn]: /usr/local/rvm/gems/ruby-2.1.5/gems/fluentd-0.12.1/lib/fluent/plugin/out_forward.rb:321:in `send_data'
  2014-12-15 15:26:56 +0900 [warn]: /usr/local/rvm/gems/ruby-2.1.5/gems/fluentd-0.12.1/lib/fluent/plugin/out_forward.rb:169:in `block in write_objects'
  2014-12-15 15:26:56 +0900 [warn]: /usr/local/rvm/gems/ruby-2.1.5/gems/fluentd-0.12.1/lib/fluent/plugin/out_forward.rb:163:in `times'
  2014-12-15 15:26:56 +0900 [warn]: /usr/local/rvm/gems/ruby-2.1.5/gems/fluentd-0.12.1/lib/fluent/plugin/out_forward.rb:163:in `write_objects'
  2014-12-15 15:26:56 +0900 [warn]: /usr/local/rvm/gems/ruby-2.1.5/gems/fluentd-0.12.1/lib/fluent/output.rb:459:in `write'
  2014-12-15 15:26:56 +0900 [warn]: /usr/local/rvm/gems/ruby-2.1.5/gems/fluentd-0.12.1/lib/fluent/buffer.rb:325:in `write_chunk'
  2014-12-15 15:26:56 +0900 [warn]: /usr/local/rvm/gems/ruby-2.1.5/gems/fluentd-0.12.1/lib/fluent/buffer.rb:304:in `pop'
  2014-12-15 15:26:56 +0900 [warn]: /usr/local/rvm/gems/ruby-2.1.5/gems/fluentd-0.12.1/lib/fluent/output.rb:320:in `try_flush'
  2014-12-15 15:26:56 +0900 [warn]: /usr/local/rvm/gems/ruby-2.1.5/gems/fluentd-0.12.1/lib/fluent/output.rb:140:in `run'

```

バッファも残っている

```
# ls /var/log/fluentd.test.data.*.buffer
/var/log/fluentd.test.data.b50a3b457dcfed028.buffer  /var/log/fluentd.test.data.q50a3b455b1eac4ca.buffer
```

しばらく放置した後、iptablesを解除したら無事に送信された.

## まとめ

Fluentd v0.12で導入されたAt-least-once semanticsを試してみた. アプリケーションレイヤでの到達確認が実装されることで、TCPレイヤでパケットがうまく届いていないケースについても、fluentdがそれを検知して再送してくれることが確認できた. 

ちなみに自分のところでは、ruby1.9上でfluentdを動かしていた時にプロセスが短時間ブロックするような事象が多発していて、それに起因してログのロストが発生したことがある. 恐らく、上記のようにTCPのコネクションは確立したように見えて、実は相手側がハング状態だったためにソケットバッファに滞留、最終的にソケットクローズ時にパケットが破棄されたのだと考えている.
(この時は、td-agent2にしたら解消した)

`require_ack_response`により、そのようなケースでもfluentdがちゃんと検知して再送してくれるので、このオプションは是非入れておきたい.
