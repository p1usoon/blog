---
title: "初参加のISUCON9 予選で敗北した"
date: 2019-09-08T23:00:00+09:00
draft: false
description: ISUCON9に参加してきました。初ISUCONで結果は惨敗でしたが、来年に向けて今年やったことを備忘録として残しておきます。
categories:
- ISUCON
tags:
- ISUCON
- Go
- MySQL
- Nginx
eyecatch: /posts/isucon9/result.png
share: true
---

こんにちは{{< link href="https://twitter.com/p1ass" text="@p1ass" >}}です。  
9月8日に同じく{{< link href="https://camph.net" text="CAMPHOR-" >}}の運営であるtomoyat1さんとISUCON9の予選に出場してきました。

結果は惨敗でしたが、来年に向けて今年やったことを備忘録として残しておきます。

<!--more-->

## 事前準備

ガッツリしたことは何もしていないです。先日に1時間ほど、初動をどうするかの話し合いをしました。また、軽くpprofやスロークエリログ、nginxのアクセス解析の方法を調べました。

過去問で予行練習をしたかったのですが、時間がなくそのまま本番に突入しました。

## 当日

### 10:00 ルールを読む

tomoyat1さんがサーバのインスタンス立ち上げを行っている間に予選マニュアルを読んでました。

「ユーザにあわせた商品の一覧を返すことで、購入の機会を増やすことができます。」という文章に疑問を持ちつつも、ひとまず全部を読みました。

### 10:17 webappをgit管理する

インスタンスが立ったので、ソースコードをgit管理するようにしました。

この時点でローカルで開発ができるようになりました。去年はGo Moduleに対応していなくて大変だったらしいのですが、今年は幸いなことに`go.mod`があったので、特に戸惑うことなく環境を整えることができました。

### 10:?? Nginxのaccess logを吐くようにする & alp で解析できるようにする


初回ベンチマークを回す前にNginxのアクセスログとMySQLのスロークエリログの設定(こっちはtomoyat1さんが担当)をしました。

他にもappサーバとDBサーバの分割とnetdataの準備をtomoyat1さんがやってくれました。

### 11:00 初回ベンチ

2,310: ｲｽｺｲﾝ

netdataを見るとDBの負荷が100%に張り付いていたので、SQLクエリを軽くしていこうという話になりました。


### 11:41 LIMIT 1 をつける

1件しか取得しないSELECT文にLIMIT 1をつけました。

2,410 ｲｽｺｲﾝ: ちょっと増えた

### 12:37 `getNewItems`のN+1を潰す

alpで見ると頻繁に `/new_items` や `/new_items/*.json` が叩かれていたので、このハンドラーを読んでみるとN+1だったので、JOINを駆使して修正しました。

2,310 ｲｽｺｲﾝ: ちょっと減った

### 12:53 pprofを導入

細かいプロファイルを取りたくなったので導入しました。

`getNewCategoryItems`に一番時間が使われていたので修正することにしました。

![12時57分時点のpprof](pprof-12-57.png)

### 13:20 `getNewCategoryItems` と `getTransaction` のN+1を直す

自分が`getNewCategoryItems` をtomoyat1さんが`getTransaction`を直しました。

2,710 ｲｽｺｲﾝ: 300増えた

### 13:32 `BcryptCost`を1にする

`postLogin` のbcryptoの処理が重たかったのでBcryptCostを1にしました。

後から気づいたのですがbcryptのコストの最小値は4なので1にしても無意味です(ライブラリ側でデフォルトの10にされるため)。

### 14:26 getCategoryByIDのSQLを1回で住むようにする

pprofで割合が大きかった`getCategoryByID` がSQLクエリを2回発行していたので一回で済むようにしました。

2,710 ｲｽｺｲﾝ: まさかのあまり変化せず

### 昼休憩

昼休憩です。近くのコンビニに買い出しに行きました。

itemsのGETを高速化しても商品の購入がスムーズにいかないと点数上がらないよねという話をしつつ午後の方針を決めました。

### 15:10 `postBuy` で叩く外部APIをgoroutineで叩くようにする

`POST /buy`が地味に遅かったので外部APIをgoroutineで同時に叩くようにしました。本当はトランザクション周りも良くしたかったのですが、あまり良いアイデアが思いつかずそのままにしました。

3,010 ｲｽｺｲﾝ: 3000点台に載った

### 15:52 `postComplete` のトランザクションを一部外す

本当にトランザクションを外していいのか分からなかったのですが、「駄目だったらベンチ落ちるやろ！w」ということで外しました。ついでに外部APIをgoroutineで叩くようにしました。

3,310 ｲｽｺｲﾝ: 少し伸びた


### 16:04 `BcryptCost`を4にする

`BcryptCost`の最小値が4と気づいたので変更しました。

あまりスコアに変化はありませんでした。



### 17:20 新着一覧ページをパーソナライズする

「ユーザにあわせた商品の一覧を返すことで、購入の機会を増やすことができます。」という文章があったので、各ユーザが最後に購入したカテゴリと同じカテゴリを表示するようにしました。

3,420 ｲｽｺｲﾝ: 少し伸びた

### 17:22 `campaign`を1にする & Nginxの設定を変更 by tomoyat1

これまでに`campaign`を1にするのを試していたのですが、500系が多く返ってきて変更を躊躇っていました。

このタイミングでtomoyat1さんがNginxのtoo many connectionsが原因なことを突き止めて設定を変更しました。

4,310 ｲｽｺｲﾝ: めっちゃ伸びた！



### 17:47 インデックスをinit.shから貼るようにする

今までDBのインデックスを直接貼っていたのですが、initializeでDROPしているので意味がないことに気づき、ベンチのスタート時にセットするようにしました。

5,610 ｲｽｺｲﾝ: 結構伸びた

### 18:00 ~ ポータルが落ちて延長戦 & ベンチを回す

色々とログを吐く設定をやめたりしてベンチを回したが点数が半分になりました。ワロタ

2,810 ｲｽｺｲﾝ: 😇

## スコアの遷移

failは沢山ありますが、基本は右肩上がりにすることが出来ました。
最後だけが悲しいです。

![スコアの推移](result.png)

## 感想

初のISUCONでしたがものすごく楽しかったです。他のチームがスコア伸ばしてるのを見ると負けられないぞという気持ちになりますし、色々と勉強になりました。

来年は本戦行きたいです！💪💪