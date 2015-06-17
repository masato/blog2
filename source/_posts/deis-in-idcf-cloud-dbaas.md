title: "Deis in IDCFクラウド - Part4: DBaaS"
date: 2014-08-17 12:43:19
tags:
 - Deis
 - CoreOS
 - DBaaS
 - fleet
description: DockerでオープンソースPaaS - Part5 tsuruのDBaaSでtsuruの場合のDBaaSをみました。Deisの場合はService registry lookup #231というissueの中で議論されていましたが、今のところ正式に採用はされていません。
---

[DockerでオープンソースPaaS - Part5: tsuruのDBaaS](/2014/07/06/docker-opensource-tsuru-dbaas/)でtsuruの場合のDBaaSをみました。
Deisの場合は[Service registry lookup #231](https://github.com/deis/deis/issues/231)というissueの中で議論されていましたが、今のところ正式に採用はされていません。

### Stackoverflowの情報

Stackoverflowに投稿がありました。DeisではPostgreSQLとRedisを使っていますが、Deisサービス用途のためユーザーが利用するDBではないそうです。

[How can I setup and deploy a database with Deis (PaaS)](http://stackoverflow.com/questions/23298732/how-can-i-setup-and-deploy-a-database-with-deis-paas)

### Configure the Application

今のところ、環境変数に`deis config:set`で接続情報をセットする方法しかありません。

[Configure the Application](http://docs.deis.io/en/latest/developer/config-application/)

```
$ deis config:set DATABASE_URL=postgres://user:pass@example.com:5432/db
=== peachy-waxworks
DATABASE_URL: postgres://user:pass@example.com:5432/db
```

普通にデータベースのサービスは自分でfleetから管理する必要があるので、[deis-database.service](https://github.com/deis/deis/blob/master/database/systemd/deis-database.service)を参考にしながら、次からfleetのsystemdの書き方を勉強していきます。
