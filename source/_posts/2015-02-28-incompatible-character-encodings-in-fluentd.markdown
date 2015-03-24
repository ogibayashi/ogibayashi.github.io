---
layout: post
title: Fluentdで incompatible character encodingのエラーが出た話
date: 2015-02-28 22:24:31 +0900
comments: true
categories: [fluentd]
---

Fluentdに対して、in_httpのインタフェースで送信してくるアプリケーションがあるのだけど、そのアプリケーションに対してリリースがあった後、一部のメッセージがうまく送れないという話があった. 見てみたら、受信側Fluentdに以下のようなメッセージが多発していた. fluentdのバージョンは0.10.48. 

```
2015-02-06 14:14:52 +0900 [warn]: fluent/engine.rb:156:rescue in emit_stream: em
it transaction failed  error_class=Encoding::CompatibilityError error=#<Encoding
::CompatibilityError: incompatible character encodings: ASCII-8BIT and UTF-8>
  2015-02-06 14:14:52 +0900 [warn]: plugin/in_http.rb:147:on_request: /opt/td-ag
ent/embedded/lib/ruby/gems/2.1.0/gems/fluentd-0.10.48/lib/fluent/event.rb:32:in
`<<'
  2015-02-06 14:14:52 +0900 [warn]: plugin/in_http.rb:147:on_request: /opt/td-ag
ent/embedded/lib/ruby/gems/2.1.0/gems/fluentd-0.10.48/lib/fluent/event.rb:32:in
`to_msgpack'
  2015-02-06 14:14:52 +0900 [warn]: plugin/in_http.rb:147:on_request: /opt/td-ag
ent/embedded/lib/ruby/gems/2.1.0/gems/fluentd-0.10.48/lib/fluent/event.rb:32:in
`block in to_msgpack_stream'
  2015-02-06 14:14:52 +0900 [warn]: plugin/in_http.rb:147:on_request: /opt/td-ag
ent/embedded/lib/ruby/gems/2.1.0/gems/fluentd-0.10.48/lib/fluent/event.rb:54:in
`call'
  2015-02-06 14:14:52 +0900 [warn]: plugin/in_http.rb:147:on_request: /opt/td-ag
ent/embedded/lib/ruby/gems/2.1.0/gems/fluentd-0.10.48/lib/fluent/event.rb:54:in
`each'
  2015-02-06 14:14:52 +0900 [warn]: plugin/in_http.rb:147:on_request: /opt/td-ag
ent/embedded/lib/ruby/gems/2.1.0/gems/fluentd-0.10.48/lib/fluent/event.rb:31:in
`to_msgpack_stream'
  2015-02-06 14:14:52 +0900 [warn]: plugin/in_http.rb:147:on_request: /opt/td-ag
ent/embedded/lib/ruby/gems/2.1.0/gems/fluentd-0.10.48/lib/fluent/output.rb:424:i
n `emit'
  2015-02-06 14:14:52 +0900 [warn]: plugin/in_http.rb:147:on_request: /opt/td-ag
ent/embedded/lib/ruby/gems/2.1.0/gems/fluentd-0.10.48/lib/fluent/output.rb:33:in
 `next'

```

全てのメッセージが送れてないわけではない模様. 適当なメッセージをcurlで送ってみても問題ないので
なんだろう、、、と思っていたら、以下のpull requestを見つけた. 偶然、事象が発生したのと同日. 

https://github.com/fluent/fluentd/pull/550

Messagepackの不具合?で、5KBを超える、且つエンコードがASCII-8BIT以外の文字列をシリアライズしようとした時に、このエラーが出ることがあるそうな. msgpack 0.5.11で修正.

アプリケーションの人に聞いてみると、確かにメッセージサイズは大幅に増えているようだし、エラー発生時のパケットを見ても、メッセージサイズが数百KBととても大きい. このpull requestは v0.10.60で取り込まれているようなので、td-agentを2.1.4に上げてテストをしてみたら発生しなくなった.

もっと前にこれが発生してたら困ってただろうな. ちょうど修正されてて助かりました. 


