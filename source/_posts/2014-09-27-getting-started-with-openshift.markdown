---
layout: post
title: Getting Stated with OpenShiftを読んだ
date: 2014-09-27 08:16:08 +0900
comments: true
categories: [openshift]
published: true
---
[Getting Stated with OpenShift](http://shop.oreilly.com/product/0636920033226.do)を読んだので、簡単にまとめ. とりあえず4章まで.

## この本について
OpenShiftの管理者ではなく、Webアプリケーション開発者向けの本. OpenShift Onlineを使って、どのようにWebアプリケーションを動かすことができるか、ということが書いてある

## 1. Introduction

* OpenShiftとは？
	* RedHatが提供するPaaS
* 3つのバージョンがある
	* OpenShift Origin : オープンソースであり、最新版. 自分の環境に入れて使うことができる. OnlineやEnterpriseのUpstreamとなる.
	* OpenShift Online : RedHatが提供するクラウドサービス版. AWS上で動いており、アカウントを作ればOpenShiftの環境を使うことができる.  本書はこれを対象に書かれている.
	* OpenShift Enterprise : Productionでの利用を想定し、RedHatによりサポートされる安定版. 自分でインストールして使う.
* 基本的な用語
	* Application : OpenShift上で動かすWebアプリケーション
	* Gear : サーバコンテナ. 使えるリソースに応じて、small, medium, largeの3タイプがある
	* Cartridge: Gearに追加する機能の固まり. JBossとか、Pythonとか、DatabaseとかCronとか.

## 2. Creating Application

* OpenShift Onlineを使うための流れは以下の通り
  * アカウントを作る
  * コマンドラインツール(rhc)を入れる. rhcはrubygemなので、`gem install rhc`で入れることができる
  * コマンドラインツールをセットアップする. `rhc setup`で行う. OpenShiftのAPIをコマンドラインから使うための認証情報などを登録する.
* 以後、OpenShift上のアプリケーション管理は`rhc`コマンドで行う
* OpenShiftのアプリケーションを作るには`rhc app create insultapp python-2.7`のようにする. 引数は、アプリケーション名と言語.
  * 実行すると、アプリケーションにアクセスするためのURLや、GearにSSHログインするためのユーザ名@ホスト名が表示される
  * また、カレントディレクトリにGitリポジトリが作成される. このリポジトリ内にアプリケーションのコードや色々な設定が格納される.
* アプリケーションを作成する際に、 `-g`オプションを付けるとautoscaling機能が有効になる. 有効にすると、HAProxyがセットアップされ、アプリケーションがスケールアウトできるようになる
  * 本書では簡略化のためautoscalingは使っていないが、有効にしておいた方が良い
* smallのgearであれば無償で使える. 無償版の場合、48時間リクエストが無いとアプリケーションは一旦ディスクに書かれ、次回リクエスト時に再ロードされる (応答に時間がかかる)
  * お金を払えば、gearを増やしたり、より大きなgearを使ったり、ディスク容量を拡張したり、JBoss EAP等の有償Cartridgeを使ったり、サポートチケットを発行したり、などなどができる.

## 3. Making Code Modifications

* アプリケーション作成時に作成されたGitリポジトリ内にコードを書いて、`git push`するとデプロイされる
* `.openshift/action_hooks `ディレクトリ内にファイルを作成しておくことで、デプロイの特定のタイミングで任意の処理を実行することができる. (action hook script)
* 通常、アプリケーションのデプロイ時にアプリケーションは一旦停止する. `.openshift/markers/hot_deploy` ファイルを作成しておくと、ホットデプロイが有効になる.

## 4. Adding Application Components

* Cartridgeを追加することで、アプリケーションに様々な機能を追加することができる(`rhc cartridge add <cartridge名>`). この章では以下のCartridgeが紹介されている.
* Database
  * PostgreSQL, MySQL, MongoDB等のデータベースを使うことができる
  * アプリケーションがscalableでなければ同じgearに、そうでなければDatabase専用gearが作成され、その中にインストールされる
  * 追加時に、DBへアクセスするためのアカウントやURL等が表示される
* Cron
  * cronを使うことができる
  * git repositoryの`openshift/cron/{minutely,hourly,daily,weekly,monthly}/`以下にファイルを作成し、`git push`すると有効になる
* Continuous Integration (jenkins)
  * まず、`rhc app create`でJenkins用のアプリケーションを作成する
  * その上で、元のアプリケーションに`rhc cartridge add`でJenkinsのクライアントをCartridgeとして追加.
  * これにより、アプリケーションをpushした際にJenkins上でビルドやテストが走るようになる. ビルドが失敗した場合、アプリケーションはデプロイされない
* Metrics and Monitoring
  * `rhc cartridge add metrics-0.1 -a appname`のようにすることで、アプリケーションのモニタリング画面が追加される. CPUやメモリの状況等を確認可能
* その他にも、コミュニティで開発された多数のCartridgeが存在し、利用することができる. 








