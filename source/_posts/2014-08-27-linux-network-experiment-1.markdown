---
layout: post
title: Linuxネットワーク周りの実験1
date: 2014-08-27 19:02:47 +0900
comments: true
categories: Linux
published: false
---

## はじめに

Linuxネットワーク周りの仕組みと、カーネルパラメータの調整方法を知りたくて実験してみた. 目標としては、ネットワークで何かしらボトルネックが発生している際に、何を見て、どうパラメータなどを調整するべきなのかを知りたい.

## 使ったプログラムと環境

Rubyの標準ライブラリでサーバとクライアントを作った.

- https://gist.github.com/ogibayashi/4c77b4b74286fd75ce0c#file-client1-rb
- https://gist.github.com/ogibayashi/4c77b4b74286fd75ce0c#file-server1-rb

サーバ環境としては、クライアント用、サーバ用に物理サーバを各1台用意した. OSはCentOS 6.2. 初期状態では、カーネルパラメータはいじっていない.

## わかったことのまとめ

## backlogの実験1: listen時のbacklogを超えてクライアントが接続する

サーバプログラムを、listen()時のbacklogサイズを指定して起動. acceptはしない. 従って、クライアントが接続できる数は、backlogサイズで制限される想定.

	# ./server1.rb -b 10

クライアントは、100多重で接続する

	# ./client1.rb -h 192.168.1.3 -n 100

### 結果

この実験の結果. 詳細は後述.

- クライアントからの接続のうち、backlogサイズ+1が接続成功、残りはタイムアウトした. 想定通り.
- サーバ側は、backlog + 1(11本)がESTABLISHED状態になる. SYN_RECVは4本. SYN_RECVの数が何で決まるのか不明
- クライアント側は、サーバでESTABLISHED+SYN_RECV状態のものがESTABLISHEDに見える. 残りはSYN_SENT. 
- クライアントはSYN_SENTだが、サーバでは無視されているコネクションが多数存在. netstatにも計上されていない.

### セッション状況(netstat)

サーバ側. backlogで指定した値+1となった. SYN_RECV状態の4本、というのが何に依存しているのかが不明

    # netstat --tcp -na | grep :11000 | awk '{print $6}' | sort | uniq -c
     11 ESTABLISHED
      1 LISTEN
      4 SYN_RECV

クライアント側. サーバでSYN_RECVとなっているもの含め、15本がESTABLISHED、残りはSYN_SENTでサーバからのSYN-ACKを待っている.

	# netstat --tcp -na | grep :11000 | awk '{print $6}' | sort | uniq -c
     15 ESTABLISHED
     85 SYN_SENT

しばらくすると、クライアント側は15本のESTABLISHEDのみとなった.

	[root@psdata4 ~]# netstat --tcp -na | grep :11000 | awk '{print $6}' | sort | uniq -c
     15 ESTABLISHED

### ネットワーク統計情報 (netstat -s)

backlogを溢れて落とされた分は、以下のメトリックに現れると想定. 初期状態は以下の値.

    17920 SYNs to LISTEN sockets ignored

実行後の値. +14なので、SYN_SENTのセッション達の分は含まれていなそう

	17934 SYNs to LISTEN sockets ignored

## backlogの実験2: backlogと同数のクライアントが接続する

サーバ側のbacklogは100とする. ここでも、acceptはしないので、セッションが溜まっていく想定.

    # ./networktest/server1.rb -b 100

クライアントも100多重で接続する.

    # ./networktest/client1.rb -h 192.168.1.3 -n 100

### 結果

接続要求数分(100セッション)、ESTABLISHするようになった

### セッション状況(netstat)

上がサーバ、下がクライアント. どちらから見ても100本のセッションがESTABLISHEDになっている.

    # netstat -na | grep 11000 | awk '{print $6}' | sort | uniq -c
      100 ESTABLISHED
      1 LISTEN
   # netstat --tcp -na | grep :11000 | awk '{print $6}' | sort | uniq -c
      100 ESTABLISHED

### ネットワーク統計情報(netstat -s)

開始前.

    17830  SYNs to LISTEN sockets ignored

終了後. 特にこのメトリックの値は増えていない

    17830   SYNs to LISTEN sockets ignored


## backlogの実験3: backlog, 接続数を200 (>somaxconn) とする.

今度は、前のケースと同じだが接続数を200にしてみる. somaxconnのデフォルト値が128なので、これに引っかかる想定

	# sysctl -a | grep somax
	net.core.somaxconn = 128

サーバ側.

	# ./server1.rb -b 200

クライアント側. 

	# ./client1.rb -h 192.168.1.3 -n 200

### 結果

接続確立できたセッション数は129(somaxconn + 1)だった

### セッション状況(netstat)

サーバ側. 129セッションがESTABLISHEDになっている

	# netstat --tcp -na | grep :11000 | awk '{print $6}' | sort | uniq -c
    129 ESTABLISHED
      1 LISTEN
      4 SYN_RECV

クライアント側. サーバ側でESTABLISED/SYN_RECVとなっているセッションがESTABLISHEDに見えている.

	[root@psdata4 ~]# netstat --tcp -na | grep :11000 | awk '{print $6}' | sort | uniq -c
    133 ESTABLISHED
     67 SYN_SENT

#### ネットワーク統計情報(netstat -s)

開始前

    # netstat -s | grep SYNs
    18096 SYNs to LISTEN sockets ignored

終了後. やはり、増えているが67セッションがSYN_SENTのままだったことを考えると少ない.

    # netstat -s | grep SYNs
    18112 SYNs to LISTEN sockets ignored




