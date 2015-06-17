title: 'Docker開発環境 - Ubuntu14.04のSSH接続ユーザー作成'
date: 2014-06-20 23:34:17
tags:
 - Docker
 - Dockerfile
 - DockerDevEnv
 - Ubuntu
description: 何かのコードを動かしたいときにさっと使えるdisposableな環境なので、今までrootで作業していました。ただrootだとbundlerに文句を言われたり、npm installが微妙だったりします。そこでadduserSSH接続できるように、adduserした後authorized_keysを追加するのですが、パーミッションは正しくてもSSH接続できなくていろいろと試しました。
---

* `Update 2014-06-24`: [DockerfileのADDとuseraddのPermission denied](/2014/06/24/docker-devenv-adduser-add-permission-denied/)

何かのコードを動かしたいときにさっと使えるdisposableな環境なので、今までrootで作業していました。
ただrootだとbundlerに文句を言われたり、`npm install`が微妙だったりします。

そこで`adduser`SSH接続できるように、adduserした後`authorized_keys`を追加するのですが、
パーミッションは正しくてもSSH接続できなくていろいろと試しました。

<!-- more -->

### baseimage-docker
いつもの[baseimage-docker](https://github.com/phusion/baseimage-docker)を使います。
rootの場合、authorized_keysの追加方法はREADMEに書いてあります。
``` Dockerfile
## Install an SSH of your choice.
ADD your_key /tmp/your_key
RUN cat /tmp/your_key >> /root/.ssh/authorized_keys && rm -f /tmp/your_key
```
ubuntuという名前で作業ユーザーを追加します。

``` Dockerfile
FROM phusion/baseimage:0.9.10

# Set correct environment variables.
ENV HOME /root

# Regenerate SSH host keys.
RUN /etc/my_init.d/00_regen_ssh_host_keys.sh

## Install an SSH of your choice.
ADD mykey.pub /tmp/your_key
RUN cat /tmp/your_key >> /root/.ssh/authorized_keys \
  && chmod 600 /root/.ssh/authorized_keys

# Add User
RUN adduser --disabled-password --gecos "" ubuntu
RUN mkdir -m 700 /home/ubuntu/.ssh
RUN chown ubuntu:ubuntu /home/ubuntu/.ssh
RUN cat /tmp/your_key >> /home/ubuntu/.ssh/authorized_keys \
  && chmod 600 /home/ubuntu/.ssh/authorized_keys \
  && rm -f /tmp/your_key
RUN echo "ubuntu ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers \
  && echo "ubuntu:ubuntu" | chpasswd
RUN chown -R ubuntu:ubuntu /home/ubuntu/

CMD ["/sbin/my_init"]

RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

ビルドと実行をします。
/home/ubuntu/.sshはubuntuユーザー所有になっていますが、なぜか権限がなく読めなくなります。

``` bash
$ docker build -t masato/test .
$ docker run -i -t --rm masato/test /sbin/my_init /bin/bash
# su - ubuntu
$ ls -l /home/ubuntu/.ssh/
ls: cannot open directory /home/ubuntu/.ssh/: Permission denied
$ sudo ls -l /home/ubuntu/.ssh/
total 12
drwx------ 2 ubuntu ubuntu 4096 Jun 21 01:30 .
drwxr-xr-x 6 ubuntu ubuntu 4096 Jun 21 01:30 ..
-rw------- 1 ubuntu ubuntu  398 Jun 21 01:30 authorized_keys
```

### authorized_keysの追加を修正する

ググると[coreos-vagrantでDockerしてみてわかったこととかハマったこととか](http://iakio.hatenablog.com/entry/2013/11/28/002657)に
同じような現象が書いてあったので参考にして修正していきます。

mkdirとchownを同じRUNに書くか、
``` Dockerfile
RUN mkdir -m 700 /home/ubuntu/.ssh \
  && chown ubuntu:ubuntu /home/ubuntu/.ssh
RUN cat /tmp/your_key >> /home/ubuntu/.ssh/authorized_keys \
  && chmod 600 /home/ubuntu/.ssh/authorized_keys \
  && rm -f /tmp/your_key
```

直接authorized_keysをADDします。
``` Dockerfile
ADD mykey.pub /home/ubuntu/.ssh/authorized_keys
RUN chmod 700 /home/ubuntu/.ssh \
  && chmod 600 /home/ubuntu/.ssh/authorized_keys \
  && chown -R ubuntu:ubuntu /home/ubuntu/.ssh
```

コンテナにSSHで接続して確認します。
```
$ docker run -d -t  masato/test /sbin/my_init
3aab3c7492f7495ca695283f9dcda569c6892d1453c136051be5ac53b7be58ce
$ docker inspect 3aab3c7492 | jq -r '.[0] | .NetworkSettings | .IPAddress'
172.17.0.7
$ eval `ssh-agent`
$ ssh-add ~/.ssh/mykey
$ ssh ubuntu@172.17.0.7
ubuntu@3aab3c7492f7:~$
```

### まとめ
DartSDKをubuntuユーザーのディレクトリに配置できなかったのも、これが原因のような気がします。
Goもrootに配置していたので、あわせてほかの開発環境も修正していきます。
