title: "Microsoft AzureにSalt環境を構築する - Part3: SaltとDockerをインストール"
date: 2014-12-18 00:28:12
tags:
 - Azure
 - Salt
 - Docker
description: Microsoft Azureに作成した仮想マシンにsalt-masterをインストールします。同様の手順でsalt-minion用の仮想マシンも作成してインストールをします。このSaltクラスタにはmasterとminionのどちらもDockerをインストールします。Saltの設定ファイルのひな形はsalt-configのリポジトリにpushしてあります。
---

Microsoft Azureに作成した[仮想マシン](2014/12/17/microsoft-azure-create-vm/)にsalt-masterをインストールします。同様の手順でsalt-minion用の仮想マシンも作成してインストールをします。このSaltクラスタにはmasterとminionのどちらもDockerをインストールします。Saltの設定ファイルのひな形は[salt-config](https://github.com/masato/salt-config)のリポジトリにpushしてあります。

<!-- more -->

### SSHで接続する

``` bash
$ ssh -A azureuser@masato-salt.cloudapp.net -i .ssh/azure.key -o IdentitiesOnly=yes
...
Welcome to Ubuntu 14.04.1 LTS (GNU/Linux 3.13.0-40-generic x86_64)
...
azureuser@masato-salt:~$
```

### /etc/hostsに追加

[Error message when I run sudo: unable to resolve host (none)](http://askubuntu.com/questions/59458/error-message-when-i-run-sudo-unable-to-resolve-host-none)を参考にして、/etc/hostsに自分のhostnameを追加して名前解決ができるようにします。


``` bash
$ sudo vi /etc/hosts
127.0.0.1 localhost
127.0.1.1 masato-salt
```


### Gitのインストール

Gitをインストールします。

``` bash
$ sudo apt-get update
$ sudo apt-get install -y git
```

### Salt設定リポジトリのclone

Salt構築用のリポジトリをcloneします。

``` bash
$ sudo  git clone --recursive https://github.com/masato/salt-config.git /srv/salt/config
$ cd /srv/salt/config
```

### salt-masterのインストール

### Pillarの編集

Pillarのディレクトリに移動します。

``` bash
$ cd /srv/salt/config/pillar
```

s3cmdのPillarを記入します。

``` bash
$ sudo cp s3cmd.sls.sample s3cmd.sls
$ sudo vi s3cmd.sls
s3cmd:
  access_key: xxx
  secret_key: xxx
```

usersのPillarを記入します。

``` bash
$ cd /srv/salt/config/pillar/users
$ sudo cp init.sls.sample init.sls
$ sudo vi init.sls
users:
  mshimizu:
    fullname: Masato Shimizu
    password: xxx
    sudouser: True
    sudo_rules:
      - ALL=(ALL) NOPASSWD:ALL
    groups:
      - docker
    ssh_auth:
      - ssh-rsa AAAA...
```


### /etc/salt/master.dの作成

/etc/salt/master.dのディレクトリを作成し、master.confをコピーします。

``` bash
$ sudo mkdir -p /etc/salt/master.d
$ sudo cp /srv/salt/config/etc/salt/master.d/master.conf /etc/salt/master.d
$ cat /etc/salt/master.d/master.conf
pillar_roots:
  base:
    - /srv/salt/config/pillar

file_roots:
  base:
    - /srv/salt/config/salt
    - /srv/salt/config/formulas/users-formula
```

### /etc/salt/minion.dの作成

/etc/salt/minion.dを作成し、grains.confを作成します。

``` bash
$ sudo mkdir -p /etc/salt/minion.d
$ sudo sh -c 'cat <<EOF >  /etc/salt/minion.d/grains.conf
grains:
  roles:
    - salt-master
EOF'
```


同居しているsalt-minionがsalt-masterを探せるように指定します。

``` bash
$ sudo mkdir -p /etc/salt/minion.d
$ sudo sh -c 'echo "master: masato-salt.masato-salt.l2.internal.cloudapp.net" > /etc/salt/minion.d/master.conf'
```

### salt-masterのインストール

salt-masterをインストールします。このノードにはsalt-minionも同時にインストールします。

``` bash
$ curl -L http://bootstrap.saltstack.com | sudo sh -s -- -M
...
 *  INFO: Running install_ubuntu_check_services()
 *  INFO: Running install_ubuntu_restart_daemons()
salt-minion stop/waiting
salt-minion start/running, process 4609
salt-master stop/waiting
salt-master start/running, process 4618
 *  INFO: Running daemons_running()
 *  INFO: Salt installed!
```

### salt-minionの公開鍵の承認

salt-minionの公開鍵を承認します。

``` bash
$ sudo salt-key -L
Accepted Keys:
Unaccepted Keys:
masato-salt
Rejected Keys:
$ sudo salt-key -a masato-salt
The following keys are going to be accepted:
Unaccepted Keys:
masato-salt
Proceed? [n/Y] Y
Key for minion masato-salt accepted.
```

salt-minionの公開鍵が登録されたことを確認します。

``` bash
$ sudo salt-key -L
Accepted Keys:
masato-salt
Unaccepted Keys:
Rejected Keys:
```

### state.highstate

state.highstateを実行して、git cloneしたSalt Formulaを適用します。

``` bash
$ salt '*' state.highstate
...
Summary
-------------
Succeeded: 23 (changed=16)
Failed:     0
-------------
Total states run:     23
```

### Dockerの確認

dockerグループのユーザーになります。

``` bash
$ sudo su - mshimizu
```

Dockerのバージョンを確認します。

``` bash
$ docker version
Client version: 1.4.1
Client API version: 1.16
Go version (client): go1.3.3
Git commit (client): 5bc2ff8
OS/Arch (client): linux/amd64
Server version: 1.4.1
Server API version: 1.16
Go version (server): go1.3.3
Git commit (server): 5bc2ff8
```

docker psを確認します。

``` bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

### salt-minionのインストール

salt-minionの仮想マシンを作成します。

``` bash
$ azure vm create \
  --vm-size Small \
  --ssh 22 \
  --ssh-cert ~/.ssh/azure-cert.pem \
  --no-ssh-password \
  --virtual-network-name saltvnet \
  --subnet-names salt \
  --affinity-group salt-affinity \
  masato-minion1 \
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

SSH接続します。

``` bash
$ ssh -A azureuser@masato-minion1.cloudapp.net -i .ssh/azure.key -o IdentitiesOnly=yes
```

/etc/salt/minion.dを作成し、master.confとgrains.confを編集します。

``` bash
$ sudo mkdir -p /etc/salt/minion.d
$ sudo sh -c 'echo "master: masato-salt.masato-salt.l2.internal.cloudapp.net" > /etc/salt/minion.d/master.conf'
$ sudo sh -c 'cat <<EOF >  /etc/salt/minion.d/grains.conf
grains:
  roles:
    - dev
EOF'
```

salt-minionをbootstrapからインストールします。

``` bash
$ curl -L https://bootstrap.saltstack.com | sudo sh
```

### salt-masterでsalt-minionの公開鍵を承認

salt-masterにログインして、salt-minionの公開鍵のを承認します。

``` bash
$ sudo salt-key -L
$ sudo salt-key -A
```

### state.highstate

state.highstateを実行して、git cloneしたSalt Formulaを適用します。

``` bash
$ salt '*' state.highstate
...
Summary
-------------
Succeeded: 23 (changed=16)
Failed:     0
-------------
Total states run:     23
```