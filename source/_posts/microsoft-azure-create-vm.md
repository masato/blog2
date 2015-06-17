title: "Microsoft AzureにSalt環境を構築する - Part2: 仮想マシンを作成"
date: 2014-12-17 21:02:16
tags:
 - Azure
 - Salt
 - xplat-cli
 - jq
description: 以下のサイトを参考にしてMicrosof Azureに仮想マシンを作成します。Microsof AzureではSSH用の鍵を作成からはじまり、アフィニティグループの作成、クラウドサービスの作成、仮想ネットワークの作成、DNSサーバーの作成、ストレージアカウントの作成と、仮想マシンを作成する前にいろいろ準備が必要です。
---

以下のサイトを参考にしてMicrosof Azureに仮想マシンを作成します。Microsof AzureではSSH用の鍵を作成からはじまり、アフィニティグループの作成、クラウドサービスの作成、仮想ネットワークの作成、DNSサーバーの作成、ストレージアカウントの作成と、仮想マシンを作成する前にいろいろ準備が必要です。

* [Deploying Cloudera CDH on Windows Azure Virtual Machines using Cloudera Manager](http://blogs.msdn.com/b/tconte/archive/2013/06/14/deploying-cloudera-on-windows-azure-virtual-machines-using-cloudera-manager.aspx)
* [Hadoop on Linux on Azure – Step-by-Step: Build the Infrastructure (2)](http://blogs.technet.com/b/oliviaklose/archive/2014/06/18/hadoop-on-linux-on-azure-step-by-step-build-the-infrastructure-2.aspx)

<!-- more -->

### SSH用の鍵を作成

SSH接続用にX509 証明書にカプセル化された公開鍵を作成します。

``` bash
$ openssl req -x509 -nodes -days 3650 \
  -newkey rsa:2048 \
  -keyout azure.key \
  -out azure-cert.pem
```

作成した公開鍵と秘密鍵を移動します。

``` bash
$ mkdir ~/.ssh
$ chmod 700 ~/.ssh
$ mv azure* ~/.ssh
$ chmod 600 ~/.ssh/azure.key
```

~/.ssh/configにazureuserを設定します。

``` bash
$ cat >> ~/.ssh/config <<EOS
Host *.cloudapp.net
    User azureuser
    IdentityFile $HOME/.ssh/azure.key
EOS
```

### イメージの検索

戻り値のJSONをパースするためjqをインストールします。

``` bash
$ cd /usr/local/bin
$ wget -O jq http://stedolan.github.io/jq/download/linux64/jq
$ chmod u+x jq
```

Ubuntu 14.04.1のイメージをjqを使い検索します。

``` bash
$ azure vm image list --json | jq '.[] | select(.publisherName == "Canonical" and contains({label:"14.04.1"})) | {label,name,imageFamily}'
...
{
  "label": "Ubuntu Server 14.04.1 LTS",
  "name": "b39f27a8b8c64d52b05eac6a62ebad85__Ubuntu-14_04_1-LTS-amd64-server-20140927-en-us-30GB",
  "imageFamily": "Ubuntu Server 14.04 LTS"
}
{
  "label": "Ubuntu Server 14.04.1 LTS",
  "name": "b39f27a8b8c64d52b05eac6a62ebad85__Ubuntu-14_04_1-LTS-amd64-server-20141125-en-us-30GB",
  "imageFamily": "Ubuntu Server 14.04 LTS"
}
...
```

### 仮想ネットワークの作成

アフィニティグループを作成します。

```
$ azure account affinity-group create \
  --location "Japan East" \
  --label salt-affinity \
  salt-affinity
```

仮想ネットワークを作成します。
 
``` bash
$ azure network vnet create  \
  --address-space 10.0.0.0 \
  --cidr 8 \
  --subnet-name salt \
  --subnet-start-ip 10.0.0.0 \
  --subnet-cidr 24 \
  --affinity-group salt-affinity \
  saltvnet
info:    Executing command network vnet create
+ Getting network configuration
+ Getting or creating affinity group
+ Getting affinity groups
info:    Using affinity group salt-affinity
+ Updating Network Configuration
info:    network vnet create command OK
```

### Linuxの場合はDNSサーバーは不要

[Azure での Linux 入門](http://azure.microsoft.com/ja-jp/documentation/articles/virtual-machines-linux-introduction/)を読むと、Linuxイメージを使う場合は自分でDNSサーバーを作成する必要はなさそうです。

> 最初に Linux イメージのインスタンスをデプロイするときに、仮想マシンのホスト名を提供するように求められます。仮想マシンが実行されると、このホスト名はプラットフォーム DNS サーバーに公開されます。相互に接続された複数の仮想マシンがホスト名を使用して IP アドレス検索を実行できるようになります。

### ストレージアカウントを作成

ストレージアカウントを作成します。

``` bash
$ azure storage account create \
  --affinity-group salt-affinity \
  mysaltstorage
info:    Executing command storage account create
+ Creating storage account
info:    storage account create command OK
```

### クラウドサービスの作成

salt-masterとsalt-minion用のクラウドサービスを作成します。

``` bash
$ azure service create \
  --affinitygroup salt-affinity \
  masato-salt
info:    Executing command service create
+ Creating cloud service
data:    Cloud service name masato-salt
info:    service create command OK
$ azure service create \
  --affinitygroup salt-affinity \
  masato-minion1
cloud service
data:    Cloud service name masato-minion1
```

### 仮想マシンの作成

salt-master用の仮想マシンを作成します。

``` bash
$ azure vm create \
  --vm-size Small \
  --ssh 22 \
  --ssh-cert ~/.ssh/azure-cert.pem \
  --no-ssh-password \
  --virtual-network-name saltvnet \
  --subnet-names salt \
  --affinity-group salt-affinity \
  masato-salt \
  b39f27a8b8c64d52b05eac6a62ebad85__Ubuntu-14_04_1-LTS-amd64-server-20141125-en-us-30GB \
  azureuser
info:    Executing command vm create
- Looking up image b39f27a8b8c64d52b05eac6a62ebad85__Ubuntu-14_04_1-LTS-amd64-server-20141125-en-us-+0GB
+ Looking up virtual network
+ Looking up cloud service
warn:    --affinity-group option will be ignored
+ Getting cloud service properties
+ Retrieving storage accounts
+ Configuring certificate
+ Creating VM
info:    vm create command OK
```

仮想マシンの確認をします。

``` bash
$ azure vm show masato-salt
...
data:    VirtualIPAddresses 0 name "masato-saltContractContract"
data:    VirtualIPAddresses 0 isDnsProgrammed true
data:    Network Endpoints 0 localPort 22
data:    Network Endpoints 0 name "ssh"
data:    Network Endpoints 0 port 22
data:    Network Endpoints 0 protocol "tcp"
data:    Network Endpoints 0 virtualIPAddress "104.41.160.20"
data:    Network Endpoints 0 enableDirectServerReturn false
```

SSHで接続の確認をします。

``` bash
$ ssh -A azureuser@masato-salt.cloudapp.net -i .ssh/azure.key -o IdentitiesOnly=yes
...
Welcome to Ubuntu 14.04.1 LTS (GNU/Linux 3.13.0-40-generic x86_64)
...
azureuser@masato-salt:~$
```
