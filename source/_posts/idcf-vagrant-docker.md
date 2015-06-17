title: 'IDCFクラウドのVagrant1.6でDocker Providerを使う'
date: 2014-05-17 21:54:49
tags:
 - IDCFクラウド
 - Vagrant
 - DockerProvider
description: Vagrant 1.6からDockerをProviderに使えるようになりました。この前はVagrantのProviderにLXCを使いましたが、DockerをVagrantから使える方が便利な気がします。vagrant sshコマンドが使えるようにDockerのイメージにはphusion/baseimageを使います。

---

`Vagrant 1.6`からDockerをProviderに使えるようになりました。
[この前](/2014/05/11/idcf-vagrant-lxc/)はVagrantのProviderにLXCを使いましたが、DockerをVagrantから使える方が便利な気がします。
`vagrant ssh`コマンドが使えるようにDockerのイメージには[phusion/baseimage](https://github.com/phusion/baseimage-docker)を使います。

<!-- more -->

### 準備

Vagrantfileを保存するプロジェクトを作成します。
``` bash
$ mkdir ~/docker_apps/phusion
$ cd !$
```

`phusion/baseimage`で使うinsecure_keyをダウンロードします。
``` bash
$ curl -o insecure_key -fSL https://github.com/phusion/baseimage-docker/raw/master/image/insecure_key
$ chmod 600 insecure_key
```

### vagrant up

Vagrantfileを作成します。ここでは明示的にportを22に指定します。
``` ruby  ~/docker_apps/phusion/Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.define "phusion" do |v|
    v.vm.provider "docker" do |d|
      d.cmd = ["/sbin/my_init", "--enable-insecure-key"]
      d.image = "phusion/baseimage:latest"
      d.has_ssh = true
    end
    v.ssh.username = "root"
    v.ssh.private_key_path= "insecure_key"
  end
  config.ssh.port = 22
end
```

providerオプションにdockerを指定してupします。
``` bash
$ vagrant up --provider=docker
Bringing machine 'phusion' up with 'docker' provider...
==> phusion: Creating the container...
    phusion:   Name: phusion_phusion_1400332643
    phusion:  Image: phusion/baseimage:latest
    phusion:    Cmd: /sbin/my_init --enable-insecure-key
    phusion: Volume: /home/matedev/docker_apps/phusion:/vagrant
    phusion:   Port: 2222:22
    phusion:
    phusion: Container created: d419de25cc4a
==> phusion: Starting container...
==> phusion: Waiting for machine to boot. This may take a few minutes...
    phusion: SSH address: 172.17.0.2:22
    phusion: SSH username: root
    phusion: SSH auth method: private key
    phusion: Warning: Connection refused. Retrying...
==> phusion: Machine booted and ready!
```

### コンテナの確認

Dockerコンテナが起動していることを確認するため、psします。
``` bash
$ docker ps
CONTAINER ID        IMAGE                      COMMAND                CREATED             STATUS              PORTS                  NAMES
d419de25cc4a        phusion/baseimage:latest   /sbin/my_init --enab   4 minutes ago       Up 4 minutes        0.0.0.0:2222->22/tcp   phusion_phusion_1400328566
matedev@matedesk ~/docker_apps/phusion $
```

### SSH接続いろいろ

`vagrant up`したDokerコンテナに、SSH接続をいろいろな方法で試してみます。
まずは、ssh-configでVagrantのSSH設定を確認します。
``` bash
$ vagrant ssh-config
Host phusion
  HostName 172.17.0.2
  User root
  Port 22
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /home/matedev/docker_apps/phusion/insecure_key
  IdentitiesOnly yes
  LogLevel FATAL
```

コンテナのIPアドレスへ22ポートへ接続できます。
``` bash
$ ssh -i insecure_key root@172.17.0.2
Warning: Permanently added '172.17.0.2' (ECDSA) to the list of known hosts.
Last login: Sat May 17 12:09:33 2014 from 172.17.42.1
root@d419de25cc4a:~#
```

localhostにポートフォワードされた2222ポートで接続します。
``` bash
$ ssh -l root -i insecure_key -p 2222 localhost
Warning: Permanently added '[localhost]:2222' (ECDSA) to the list of known hosts.
Last login: Sat May 17 12:16:34 2014 from 172.17.42.1
root@d419de25cc4a:~#
```

`vagrant global-status`でVMとしてコンテナが起動しているのを確認できます。

``` bash
$ vagrant global-status
id       name    provider state   directory
----------------------------------------------------------------------
8bdfaed  phusion docker running /home/matedev/docker_apps/phusion
```
### コンテナを破棄する

`vagrant global-status`で確認したidを使ってdestroyします。ここではVMが1つしか起動していないので、最初の文字の8だけでidが一意にできます。
``` bash
$ vagrant destroy 8
    default: Are you sure you want to destroy the 'default' VM? [y/N] y
==> default: Deleting the container...
==> default: Removing built image...
```

`agrant destroy`でVMを破棄すると、dockerコンテナの正常に破棄されるので便利です。
``` bash
$  docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS         
```

### まとめ
`Vagrant 1.6`からサポートされたDocker Providerは開発環境として気軽に`vagrant up`と`vagrant destroy`できます。
Dockerの便利なところで、自分でDockerfileを書いたり、Dockerインデックスからいろいろなイメージをpullもそのままです。
また、コンテナのIDを指定して、`docker stop`と`docker rm`をしなくてもコンテナを破棄できるのも便利です。

