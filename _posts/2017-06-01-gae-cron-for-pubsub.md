---
layout: post
title: GAE で PubSub に毎分毎時毎日メッセージを投げると便利
date: 2017-06-01 2:44 JST
description: GAE で PubSub に毎分毎時毎日メッセージを投げると便利
tags:
---

## TL;DR
Google App Engine を用いて Cloud PubSub に一定時間でメッセージを投げると、
それをトリガーに Cloud Functions を実行することができます（もちろん GKE などもサブスクライブできます）。
これは定期的に何か処理を実行したいときにとても便利です。

## 用語説明
- [Google App Engine](https://cloud.google.com/appengine/?hl=ja) (GAE)
  - Web Application を GCP 上において実行できるサービス
- PubSub
  - パブリッシャ・サブスクライバ型のメッセージングモデル
    - Cloud PubSub の説明は[こちら](https://cloud.google.com/pubsub/?hl=ja)
- [Cloud Functions](https://cloud.google.com/functions/?hl=ja)
  - AWS でいう Lambda

## UseCase
**ある Web サイトを定期的に監視して、その Web サイトに"変化"があったら Slack に通知を投げて自分のスマートフォンで Push通知を受け取りたい** というのはよくあること（?）だと思います。

このような場合、OS の Cron 機能を用いて簡単な Script を毎分実行させるなどするのが通常の方法だと思います。
しかし実現するには、24 時間動き続けるサーバーが必要で、少し前まではそれだけのために VPS を借りるなどしていたと思います。
自分で管理している既存のサーバーがあれば相乗りしておけば良いのですが、自分で管理しているサーバーがない場合にはお金とか時間とかコストがかかります。

そこで、GAE、Cloud PubSub および Cloud Functions を用いてサーバーレスで同様の処理を実現します。
これらを用いた場合の概要は以下になります。

![概要](/img/20170602.png "概要")

GAE は自身で [Cron の機能](https://cloud.google.com/appengine/docs/standard/python/config/cron)を持っており、定期的に処理を実行することができます。
この Cron の機能を用いて、GAE から Cloud PubSub へスケジューリングされたメッセージを発行し続けることができます。
一度発行してしまえば、任意の Cloud Functions の実行トリガにすることができるので、結果的に Cloud Functions を定期的に実行することができるようになります。

ここからさらに、

- Web サイトを監視して変化があった時に Slack 投稿のイベントを発行する Cloud Functions
- Slack 投稿のイベントを購読して Slack へ投稿する Cloud Functions

を用意すれば、最初にあげた **ある Web サイトを定期的に監視して、その Web サイトに"変化"があったら Slack に通知を投げて自分のスマートフォンで Push通知を受け取りたい** というユースケースを達成することができます。

上の図のようなアーキテクチャは 1 つのスクリプトとサーバーでまとめて処理を行うことに比べて、サーバーレスである点、疎結合である点で優れていると思います。
このユースケースの他にも定期的に何か Job を実行するシステムというのはたくさんあると思いますが、
定期的に実行したいあらゆる処理のトリガを PubSub のサブスクライバに設定しておけば、 HTTP アクセス（gRPCのアクセス）などを起点にして処理を実行するというロジックを書いておくだけで定期的に実行することが可能になります。

## gae-cron
上記の図のようなアーキテクチャを実現するために、毎分、毎時、毎日および毎週、メッセージを発行する Google App Engine を作りました。

[gae-cron](https://github.com/jyane/gae-cron)

README.md に書いてあるだけでは動かない（GCP の設定など必要）ですが、環境を用意すれば README.md にあるコマンドを実行するだけで上記の図のようなアーキテクチャを実現できます。
また、[cron.yml](https://github.com/jyane/gae-cron/blob/master/cron.yaml) を編集すればそのまま任意の Topic へ指定時間ごとにメッセージを発行することができるようになります。
既知の問題として、認証を考えていないのでエンドポイントがバレると任意のトピックへイベントが発行される問題があります。~~真面目にやるならここらへんは GAE の機能でなんとかなる？気がする。~~ [Identity-Aware Proxy](https://cloud.google.com/iap/?hl=ja) で簡単にアクセス制限できました。

## おわりに
このように PubSub は個人の用途でも十分に有効なユースケースが考えられます。
PubSub は、各サービスのロジックを疎結合にしたり Queue 機能を用いて重い処理を非同期に実行できるなど、特にマイクロサービスアーキテクチャにおいて真価を発揮します。
しかし、一般に PubSub が有効と思われる場面でもその名前が挙がることがない場面を見ており、使ったことがない人には良さがわかりにくいところがあるのかなと感じました。

もっと気軽に PubSub を用いることを検討する人が増えると良いなと思いました。
