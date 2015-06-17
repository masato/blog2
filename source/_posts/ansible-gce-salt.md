title: "AnsibleでGCEにSalt Masterをプロビジョニング"
date: 2014-05-19 19:58:23
tags:
 - Ansible
 - Salt
 - GCE
description: GCEにSaltをインストールしましたが、前にAnsibleのCloud Modulesを使っていたのを思い出しました。今回はバージョンは1.6.1の、gceモジュールでSalt MasterをGCEにプロビジョニングしてみます。
---

[GCEにSalt](/2014/05/15/gce-salt-cloud/)をインストールしましたが、前に[Ansible](https://github.com/ansible/ansible)の[Cloud Modules](http://docs.ansible.com/list_of_cloud_modules.html)を使っていたのを思い出しました。
今回はバージョンは1.6.1の、[gce](http://docs.ansible.com/gce_module.html)モジュールで`Salt Master`をGCEにプロビジョニングしてみます。

<!-- more -->

### はじめに

AnsibleはChefやSaltと違ってクライアントのインストールが不要です。ローカルへの実行もできます。
[Guide](http://docs.ansible.com/guide_gce.html)と[ソースコード](https://github.com/ansible/ansible/blob/devel/library/cloud/gce)を読みながら作業を進めていきます。

記述するファイルは少ないのですが、ディレクトリ構造を覚える必要があります。

### GCE用のユーザー

gceuserにAnsibleの実行環境を作っていきます。プロジェクトを作成します。

``` bash
$ sudo su - gceuser
$ mkdir -p ansible_apps/gce
$ cd !$
```

### Ansibleのインストール

Ansibleとlibcloudをインストールします。libcloudは0.14+を使います。

``` bash
$ sudo pip install ansible apache-libcloud
$ pip search ansible
ansible                   - Radically simple IT automation
  INSTALLED: 1.6.1 (latest)
$ pip search apache-libcloud
  INSTALLED: 0.14.1 (latest)
```

### GCE用クライアントIDの作成

[Google Developers Console](https://console.developers.google.com/project)へログインし、現在使っているプロジェクトIDを開きます。
APIと認証 -> 認証情報　-> 新しいクライアントIDを作成 -> サービスアカウント -> と進み、クライアントIDを作成します。
salt-cloudで使うサービスアカウントの秘密鍵をダウンロードして、xxx-privatekey.p12の名前で保存します。

サービスアカウントの鍵を[PKCS12フォーマット](http://en.wikipedia.org/wiki/PKCS_12)からコンバートします。

``` bash
$ cd ~/ansible_apps/gce
$ openssl pkcs12 -in  ./xxx-privatekey.p12 -passin pass:notasecret \
-nodes -nocerts |openssl rsa -out ./service_account_private_key
MAC verified OK
writing RSA key
```

### Ansibleプロジェクトディレクトリ

Ansibleは複数ファイルを書いて構成するので、ディレクトリ構造が重要です。

``` bash
$ cd ~/ansible_apps
$ tree gce
gce
├── xxx-privatekey.p12
├── gce.yml
├── hosts
├── roles
│   └── salt
│       └── tasks
│           └── main.yml
└── service_account_private_key
```

インベントリファイルを作成します。localhostへはローカル接続するように指定します。

```  ~/ansible_apps/gce/hosts
[localhost]
localhost ansible_connection=local
```

GCEプロビジョニング用のPlaybookです。ちょっと長いのでvars.ymlなどは分割した方がよさそうですが、
わかりやすいように、一つのファイルにしました。ZeroMQのポートも開けています。

``` yaml ~/ansible_apps/gce/gce.yml
---
- name: Create a Sault Master instance
  hosts: localhost
  gather_facts: no
  connection: local

  vars:
    machine_type: n1-standard-1
    image: debian-7
    service_account_email: xxx.gserviceaccount.com
    pem_file: service_account_private_key
    project_id: xxx-xxx-456

    names: salt-master
    zone: asia-east1-a

  tasks:
    - name: Launch instances
      local_action: gce instance_names"&#123;&#123; names &#125;&#125;"
                    machine_type="&#123;&#123; machine_type &#125;&#125;"
                    image="&#123;&#123; image &#125;&#125;"
                    zone="&#123;&#123; zone &#125;&#125;"
                    service_account_email="&#123;&#123; service_account_email &#125;&#125;"
                    pem_file="&#123;&#123; pem_file &#125;&#125;"
                    project_id="&#123;&#123; project_id &#125;&#125;"
      tags: salt-master
      register: gce

    - name: Wait for SSH to come up
      local_action: wait_for host="&#123;&#123; item.public_ip &#125;&#125;"
                    port=22
                    delay=10
                    timeout=60
                    state=started
      with_items: gce.instance_data

- name: Configure instance(s)
  hosts: launched
  sudo: True
  roles:
    - salt

- name: Open port for ZeroMQ
  hosts: localhost
  connection: local
  gather_facts: False
  tasks:
    - name: Allow ZeroMQ port
      local_action:
        module: gce_net
        name: default
        allowed: tcp:4505,4506
        fwname: all-zeromq
        service_account_email: "&#123;&#123; service_account_email &#125;&#125;"
        pem_file: "&#123;&#123; pem_file &#125;&#125;"
        project_id: "&#123;&#123; project_id &#125;&#125;"
```

Salt Masterをインストールするロールを書きます。


``` yaml ~/ansible_apps/gce/roles/salt/tasks/main.yml
---

- name: install Salt Master
  command: curl -L http://bootstrap.saltstack.org | sh -s -- -M -N git develop
- name: install libcloud
  pip: name=apache-libcloud
```

Playbookの実行をします。

``` bash
$ ansible-playbook -i hosts gce.yml
```

作成したGCEのインスタンスにSSH接続します。

``` bash
$ gcutil ssh salt-master
Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
gceuser@salt-master:~$
```

### まとめ

久しぶりにAnsibleを使いましたが、Saltに比べファイル分割が自分にはわかりにくい気がします。
最近はDockerfileに慣れているので、Ansible流にモジュールを覚えるのが、やはりChefやPuppetみたいでなんか面倒です。

`Docker on GCE with Salt`でプロビジョニングするの方が使いやすいようです。
