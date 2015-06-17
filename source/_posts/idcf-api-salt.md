title: 'IDCFクラウドのCLIでSaltをプロビジョニングする'
date: 2014-05-29 03:40:51
tags:
 - IDCFクラウド
 - CLI
 - idcf-compute-api
 - Salt
 - jq
description: IDCFクラウドCLIのidcf-compute-apiを使って、インスタンスの作成とSaltをインストールしてみます。gcutilの--metadata_from_file=user-data:cloud-config.yamlみたいに初期設定をuser-dataを渡せると便利ですが、使えないのでインスタンスの起動後にSSHでログインしてコマンドを実行します。
---

* `Update 2014-09-06`: [Salt with Docker - Part1: Dockerインストール](/2014/09/06/salt-idcf-docker-states/)
* `Update 2014-09-07`: [Salt with Docker - Part2: WordPressインストール](/2014/09/07/salt-idcf-docker-wordpress/)


IDCFクラウドCLIの[idcf-compute-api](http://www.idcf.jp/cloud/docs/Getting%20Started)を使って、インスタンスの作成とSaltをインストールしてみます。
gcutilの`--metadata_from_file=user-data:cloud-config.yaml`みたいに初期設定をuser-dataを渡せると便利ですが、
使えないのでインスタンスの起動後にSSHでログインしてコマンドを実行します。

<!-- more -->

### salt-installコマンド

sshでコマンド実行しているだけなのでなんでもよいですが、いつものようにSaltをインストールするスクリプトを書きました。
idcf-compute-apiはvirtualenvにインストールしているため、最初にactivateしています。

作成するインスタンスの設定は、[昨日](/2014/05/28/idcf-api-jq/)と同じです。

* `--serviceofferingid 21` -> M4サイズ
* `--templateid 7183` -> Ubuntu Server 12.04.04
* `--zoneid 1` -> jp-east-t1ゾーン

``` bash ~/bin/salt-install
#!/bin/bash

activate () {
  . ~/.venv/Python-2.7.6/bin/activate
}

activate

if [ -z "$1" ]; then
  echo "hostname should be set"
  exit 1
fi

results=$(idcf-compute-api deployVirtualMachine \
          --serviceofferingid=21 --templateid=7183 --zoneid=1 \
          --name=$1 --displayname=$1 --keypair=salt)

jobid=$(echo ${results} | jq '.deployvirtualmachineresponse.jobid')

echo jobid: ${jobid}
if [ "${job_id}" = "null" ]; then
  echo ${results}
  exit 1
fi

while :
do
  job_status=$(idcf-compute-api queryAsyncJobResult --jobid=${jobid} \
              | jq '.queryasyncjobresultresponse.jobstatus')
  if [ ${job_status} -eq 0 ]; then
    echo -ne "."
    sleep 15s
  else
    json=$(idcf-compute-api queryAsyncJobResult --jobid=${jobid})
    echo ${json} | jq '.queryasyncjobresultresponse.jobresult.virtualmachine | {id,password,name,displayname}, {ipaddress: .nic[].ipaddress}'
    break
  fi
done

until [ `nmap --open -p 22 "$1" |grep -c "ssh"` -eq 1 ]
do
  sleep 4
done

SSH_OPTS="-o UserKnownHostsFile=/dev/null -o CheckHostIP=no -o StrictHostKeyChecking=no"
SSH_KEY="-i /home/mshimizu/.ssh/salt"

while :
do
  retval=$(ssh $SSH_OPTS $SSH_KEY root@$1 "echo ssh success")
  if [[ "$retval" =~ success$ ]]; then
    echo $retval
    break
  else
    echo -ne "."
    sleep 4
  fi
done

if [ "$2" = "master" ]; then
  ssh $SSH_OPTS $SSH_KEY -A root@$1 "curl -L http://bootstrap.saltstack.org | sh -s -- -M -X"
else
  ssh $SSH_OPTS $SSH_KEY -A root@$1 "curl -L http://bootstrap.saltstack.org | sh -s -- -X"
fi
```

### 使い方

スクリプトの第1引数にhostname、第2引数にmasterかminionのどちらをインストールするか指定します。

salt-masterのインスタンス作成とインストールの場合

``` bash
$ salt-install salt master
```

salt-minionのインスタンス作成とインストールの場合

``` bash
$ salt-install minion1 minion
```

salt-masterのホスト名はsaltにすると`deployVirtualMachine`の`--name`オプションでhostnameに指定ます。
IDCFクラウドでは、アカウント内は`cs29dcloud.internal`のドメイン名で名前解決できるので、
hostnameをsaltにしておくと、salt-minionが自動的にsalt-masterを探してくれます。


### SSH起動のタイミング

はまったのが、SSHがつかえる状態になるまで時間がかかることです。
* queryAsyncJobResultでジョブが終了してもOSの起動は終了していない
* OSが起動しても、sshdが開始していない。
* nmapで22ポートを確認しても、authorized_keysのコピーが終わっていない

もっとよい書き方がありそうですが、公開鍵認証のsshでテストコマンドを実行して、echoが返るまでポーリングすることにしました。

### まとめ

今回は単純にSSHでbootstrapコマンドを実行するだけでしたが、Ansibleの[wait_for](http://docs.ansible.com/wait_for_module.html)を使うとYAMLで[きれい](/2014/05/19/ansible-gce-salt/)に書けるかも知れません。
