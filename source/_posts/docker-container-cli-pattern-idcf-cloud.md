title: "DockerでIDCFクラウドのCLIを実行する"
date: 2015-09-19 14:03:15
categories:
 - Docker
tags:
 - DockerCompose
 - CLI
 - IDCFクラウド
 - Python
---

[Dockerコンテナ上でCLIを実行するデザインパターン](/2015/04/22/docker-container-cli-pattern/)を以前調べましたが、今回は[IDCFクラウドのCLI](http://www.idcf.jp/cloud/spec/api.html)のイメージを作ってみます。[virtuanemv](https://virtualenv.pypa.io/en/latest/)で仮想環境を用意しても良いのですが、Dockerイメージでコマンドを配布するとホストマシンの環境を汚さずにお試しで実行できます。CLIの配布形式としてもDockerイメージは便利に使えます。

<!-- more -->

## プロジェクト

今回作成したリポジトリは[こちら](https://github.com/masato/docker-idcfcli)です。

### Dockerfile

ベースイメージはオフィシャルの[Python](https://hub.docker.com/_/python/)の2.7 ONBUILDを使います。ONBUILDでビルドに必要な処理をするためDockerfileはとても簡単です。

``` bash Dockerfile
FROM python:2-onbuild
```

### requirements.txt

IDCFの[リポジトリ](https://github.com/idcf/cloudstack-api)にはrequirements.txtが同梱されていません。依存するパッケージの追加と、cloudstack-apiもGitHub経由でインストールするように定義します。

```txt requirements.txt
httplib2
simplejson
argparse
prettytable==0.5
parsedatetime==0.8.7
lxml
-e git+https://github.com/idcf/cloudstack-api#egg=cloudstack-api
```

ONBUILDのベースイメージがrequirements.txtファイルのCOPYと`pip install`をしてくれます。

```bash Dockerfile
FROM python:2.7

RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

ONBUILD COPY requirements.txt /usr/src/app/
ONBUILD RUN pip install --no-cache-dir -r requirements.txt

ONBUILD COPY . /usr/src/app
```

### docker-compose.yml

IDCFクラウドコンソールの[API Key](https://console.idcfcloud.com/user/apikey)から環境変数を設定します。

* IDCF_COMPUTE_HOST: エンドポイント
* IDCF_COMPUTE_API_KEY: API Key
* IDCF_COMPUTE_SECRET_KEY: Secret Key

```yaml docker-compose.yml
idcfcli:
  build: .
  volumes:
    - /etc/localtime:/etc/localtime:ro
  environment:
    - IDCF_COMPUTE_HOST=
    - IDCF_COMPUTE_API_KEY=
    - IDCF_COMPUTE_SECRET_KEY=
  command: ["/usr/local/bin/cloudstack-api","listVirtualMachines","-t=id,name,state"]
```

## 使い方

リポジトリから`git clone`します。docker-compose.yml.defaultをdocker-compose.ymlにリネームして環境変数を設定します。

```bash
$ git clone https://github.com/masato/docker-idcfcli.git idcfcli
$ cd idcfcli
$ mv docker-compose.yml.default docker-compose.yml
$ vi docker-compose.yml
```

Docker Composeからコンテナを起動します。デフォルトのコマンドは[listVirtualMachines](http://docs.idcf.jp/cloud/api/virtual-machine/#listvirtualmachines)を実行しています。

```bash
$ docker-compose build
$ docker-compose run --rm idcfcli
+--------------------------------------+------+---------+
|                  id                  | name |  state  |
+--------------------------------------+------+---------+
| xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx | seed | Running |
+--------------------------------------+------+---------+
```

任意のコマンドは以下のように実行します。

```bash
$ docker-compose run --rm idcfcli cloudstack-api listVirtualMachines -t=id,name,state
+--------------------------------------+------+---------+
|                  id                  | name |  state  |
+--------------------------------------+------+---------+
| xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx | seed | Running |
+--------------------------------------+------+---------+
```

### エイリアスの作成

` ~/.bashrc`などにエイリアスを定義しておきます。

```bash ~/.bashrc
alias idcf-cli='docker-compose run --rm idcfcli cloudstack-api'
```

エイリアスを使うとよりCLIらしくなります。

```bash
$ idcf-cli listVirtualMachines -t=id,name,state
+--------------------------------------+------+---------+
|                  id                  | name |  state  |
+--------------------------------------+------+---------+
| xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx | seed | Running |
+--------------------------------------+------+---------+
```

