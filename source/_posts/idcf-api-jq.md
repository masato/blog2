title: 'IDCFクラウドのCLIをjqで便利に使う'
date: 2014-05-28 00:43:46
tags:
 - IDCFクラウド
 - CLI
 - idcf-compute-api
 - jq
description: これまで何度かdockerコマンドとjqを組み合わせて使ってきました。IDCFクラウドCLIのidcf-compute-apiコマンドは、デフォルトでレスポンスをJSONで返すので、同じようにjqでパースしてみます。また、idcf-compute-apiコマンドはJSON以外にも、CSVや表形式でレスポンスを表示できるので、サンプルを見ようと思います。
---

これまで何度かdockerコマンドと[jq](http://stedolan.github.io/jq/)を組み合わせて使ってきました。
IDCFクラウドCLIの[idcf-compute-api](http://www.idcf.jp/cloud/docs/Getting%20Started)コマンドは、デフォルトでレスポンスをJSONで返すので、同じようにjqでパースしてみます。

また、idcf-compute-apiコマンドはJSON以外にも、CSVや表形式でレスポンスを表示できるので、サンプルを見ようと思います。

<!-- more -->

### deployVirtualMachine

[deployVirtualMachine](http://www.idcf.jp/cloud/docs/api/user/deployVirtualMachine)コマンドを使うと、IDCFクラウドにインスタンスを作成できます。
このコマンドは、非同期APIなのでジョブIDを返して終了します。

`--serviceofferingid`、`--templateid`、`--zoneid`は必須のパラメータのため、値は予め調べておく必要があります。
調べ方はあとで書きますが、数値はそれぞれ以下のIDを表しています。

* `--serviceofferingid 21` -> M4サイズ
* `--templateid 7183` -> Ubuntu Server 12.04.04
* `--zoneid 1` -> jp-east-t1ゾーン

とりあえずサンプルです。

``` bash
jobid=$(idcf-compute-api deployVirtualMachine --keypair mykey \
          --serviceofferingid 21 --templateid 7183 --zoneid 1 \
          --name myhostname --displayname myhostname  \
          | jq '.deployvirtualmachineresponse | .jobid')
echo jobid: ${jobid}　
```

このジョブIDを使い、インスタンスの作成の終了をポーリングしてみます。

``` bash
while :
do
  jobstatus=$(idcf-compute-api queryAsyncJobResult --jobid=${jobid} \
              | jq '.queryasyncjobresultresponse | .jobstatus')
  if [ ${jobstatus} -eq 1 ]; then
    echo -ne "."
    sleep 10s  
  else
    json=$(idcf-compute-api queryAsyncJobResult --jobid=${jobid})
    echo ${json} | jq '.queryasyncjobresultresponse | .jobresult | .virtualmachine | {id,password,name,displayname}, {ipaddress: .nic[].ipaddress}'
    break
  fi
done
```

### レスポンスJSONのjqパース例

`queryAsyncJobResult`コマンドは、以下のようにレスポンスを返します。

``` javascript
{
  "queryasyncjobresultresponse": {
    "jobid": 1068081,
    "jobprocstatus": 0,
    "jobresult": {
      "virtualmachine": {
...
        "name": "minion2",
        "nic": [
          {
            "gateway": "10.1.0.1",
...
```

先ほど、

``` bash
echo ${json} | jq '.queryasyncjobresultresponse | .jobresult | .virtualmachine | {id,password,name,displayname}, {ipaddress: .nic[].ipaddress}'
```

のようにパースしましたが、JSONの構造をドット(.)でパイプしながら降りていきます。
パイプの最後が出力される結果になり、上記の例ではでは一つ深い`nic`要素の`ipaddress`をオブジェクトとして作り直しています。

ちょっとわかりずらいですが、[マニュアル](http://stedolan.github.io/jq/manual/)もあるので、使っていくうちにだんだん覚えていくと思います。


### listServiceOfferings

`listServiceOfferings`コマンドは、作成するインスタンスサイズをリストします。
`deployVirtualMachine`の`--serviceofferingid`オプションではIDを指定するので、予め調べておきます。

jqを使うとこんな感じです。

``` bash
$ idcf-compute-api listServiceOfferings | jq '.listserviceofferingsresponse | .serviceoffering[] | {id,name}'
{
  "name": "DL32",
  "id": 14
}
{
  "name": "XL16",
  "id": 15
}
...
```

`idcf-copute-api`には、結果をCSV型式で表示する`-c`や、表形式の`t`オプションがあります。
表示したいプロパティを指定できるので、便利な場合もあります。

``` bash
$ idcf-compute-api listServiceOfferings -c=id,name
"id","name"
14,"DL32"
15,"XL16"
...
$ idcf-compute-api listServiceOfferings -t=id,name
+----+------+
| id | name |
+----+------+
| 14 | DL32 |
| 15 | XL16 |
| 16 | L32  |
...
```

### listTemplates

`deployVirtualMachine`の`--templateid`オプションで指定する、テンプレートのイメージをリストします。

jqのフィルタを使いUbuntuだけに限定し、idとname要素だけ選択してみます。
書き方はORMのDSLみたいでちょっと苦手です。

``` bash
$ idcf-compute-api listTemplates --templatefilter=featured | jq '.listtemplatesresponse | .template[] | select(contains({name:"Ubuntu"}) and contains({name:"LATEST"})) | {id,name}'
...
{
  "name": "[LATEST][Dedicated Hardware] Ubuntu Server 12.04.1 LTS 64-bit",
  "id": 2213
}
{
  "name": "[LATEST] Ubuntu Server 10.04 LTS 64-bit",
  "id": 1022
}
{
  "name": "[LATEST] Ubuntu Server 10.04 LTS 64-bit",
  "id": 1022
}
...
```

同じことを`idcf-compute-api`の`-c`オプションと、sortやgrepをパイプした例を試してみます。
UNIXコマンドに慣れていれば、jqのフィルタよりわかりやすいかも知れません。

``` bash
$ idcf-compute-api listTemplates --templatefilter=featured  -c=id,name | egrep 'LATEST.*Ubuntu' | sort -t , -k 2
1022,"[LATEST] Ubuntu Server 10.04 LTS 64-bit"
1022,"[LATEST] Ubuntu Server 10.04 LTS 64-bit"
7183,"[LATEST] Ubuntu Server 12.04.04 LTS 64-bit"
7183,"[LATEST] Ubuntu Server 12.04.04 LTS 64-bit"
1012,"[LATEST][Dedicated Hardware] Ubuntu Server 10.04 LTS 64-bit"
3303,"[LATEST][Dedicated Hardware] Ubuntu Server 10.04 LTS 64-bit"
2213,"[LATEST][Dedicated Hardware] Ubuntu Server 12.04.1 LTS 64-bit"
3311,"[LATEST][Dedicated Hardware] Ubuntu Server 12.04.1 LTS 64-bit"
```

### まとめ

`--keypair`は[listSSHKeyPairs](http://www.idcf.jp/cloud/docs/api/user/listSSHKeyPairs)コマンドで、`--zoneid`は[listZones](http://www.idcf.jp/cloud/docs/api/user/listZones)コマンドで確認できます。

jqを使うと、ジョブIDやステータスをAPIのレスポンスから抽出できるので、
シェルスクリプトで変数に詰めたり、if分で評価するときに便利に使えます。

[この前](/2014/05/15/gce-salt-cloud/)は、Saltを使ってGCEにプロビジョニングしました。
IDCFクラウドでも、GCEやCoreOSで便利なcloud-initが使えたらよいのですが、今のAPIだとちょっと難しそうです。

また、CloudStack用のAPIライブラリに、Goで書かれた[gopherstack](https://github.com/mindjiver/gopherstack)
があります。わかりやすいGoのコードなのであとで読んでみようと思います。


そのため、次回はシェルスクリプトでidcf-compute-apiを使いsalt-masterとsalt-minionをプロビジョニングしてみます。
