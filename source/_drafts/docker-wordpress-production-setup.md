title: docker-wordpress-backup-restore
date: 2014-10-17 00:06:05
tags:
---

WordPressは[BackUpWordPress](https://wordpress.org/plugins/backupwordpress/)を使い毎日バックアップを取得しています。

docker-lloydのバックアップ手順ができたので、MySQLとWordPressのバックアップに設定します。

本番用のsalt-minionをIDCFクラウド上に用意して、docker-registryからイメージの取得と、オブジェクトストレージからアーカイブのリストアを行います。もしクラウドの障害が発生した場合でも、別のsalt-minionを作成してバックアップから復旧できるようにします。


