title: 'DockerfileのADDとuseraddのPermission denied'
date: 2014-06-24 23:06:52
tags:
 - Docker
 - DockerDevEnv
 - Dockerfile
 - Go
 - Dart
---

[SSHで接続できる作業ユーザー](/2014/06/20/docker-devenv-add-user/)で作ったのですが、
それでもホームディレクトリに展開したGoやDartのSDKや、.emacs.dの`Permission denied`が解消されず、Dockerfileを見直しています。

作成したuseraddも修正して、ユーザー名を`docker`にしました。

[Guidance for Docker Image Authors](http://www.projectatomic.io/docs/docker-image-author-guidance/)にも書いてありますが、
Dockerが広まってくると、やはりrootで作業し続けるのはお行儀がよくありません。

PackerでOVAを作っているとイメージの再作成をすると時間がかかってしまい、ビルド開始した後にミスを見つけると呪いたくなります。
Dockerの場合は何回でも気軽にやり直せるので、開発環境を少しずつよくしていくことができます。断捨離またはdispose all the things。

<!-- more -->


あまり複数コマンドを一つのRUNに束ねると、エラーがわかりにくくなりますが、
ディレクトリ作成系のコマンドとパーミッション変更系のコマンドは同時にRUNしないと、`Permission denied`になります。

``` Dockerfile
RUN useradd -m -d /home/docker -s /bin/bash docker \
 && echo "docker:docker" | chpasswd \
 && mkdir /home/docker/.ssh \
 && chmod 700 /home/docker/.ssh \
 && cp /root/.ssh/authorized_keys /home/docker/.ssh \
 && chmod 600 /home/docker/.ssh/authorized_keys \
 && chown -R docker:docker /home/docker/.ssh

RUN echo "docker ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
```

### Go

ググっても、ENVで実行時に環境変数を設定する方法は書いてあっても、作業ユーザーのPATHを設定する方法が
よくわかりません。rootから`sudo - `してもENVは消えてしまいますし、とりあえず`~/.profile`に書いたらPATHが通りました。


``` Dockerfile
## Go

RUN apt-get install -y golang
RUN mkdir -p /home/docker/gocode/src /home/docker/gocode/bin /home/docker/gocode/pkg \
 && echo 'export GOPATH=/home/docker/gocode' >> /home/docker/.profile \
 && echo 'export PATH=${PATH}:/home/docker/gocode/bin' >> /home/docker/.profile \
 && chown -R docker:docker /home/docker/gocode
```

### Dart

Dartに関してはdart-sdkのzipにも[パーミッション問題](https://code.google.com/p/dart/issues/detail?id=7601)があるようで、グループ権限も疑っていました。今では以下のRUNで問題ないです。

``` Dockerfile
## Dart
RUN wget http://storage.googleapis.com/dart-archive/channels/stable/release/latest/sdk/dartsdk-linux-x64-release.zip -P /home/docker \
   && unzip /home/docker/dartsdk-linux-x64-release.zip -d /home/docker \
   && echo 'export PATH=${PATH}:/home/docker/dart-sdk/bin' >> /home/docker/.profile \
   && chown -R docker:docker /home/docker/dart-sdk
```

### ADD directories /home/docker

一番嵌まった原因は、ドットファイルをまとめたディレクトリをADDするところです。
rootの場合は問題がなかったのですが、非rootのホームディレクトリへADDするとファイルを編集できなくなります。

userid、groupidも1000で正しいのですが、ファイルを編集できなくなります。

```
$ id
uid=1000(docker) gid=1000(docker) groups=1000(docker)
$ cd .ssh    
-su: cd: .ssh: Permission denied
```

前回SSH接続ユーザー作成時の、[authorized_keys](https://github.com/dotcloud/docker/issues/1295)のADDに近い感じです。
結局、rootのドットファイルを個別にcpして、最後にchownすることにしました。


``` Dockerfile
## dotfiles
RUN mkdir -p /root/tmp /home/docker/tmp
ADD dotfiles /root/
#ADD dotfiles /home/docker/
RUN cp -R /root/.byobu /home/docker/ \
  && cp -R /root/.emacs.d /home/docker/ \
  && cp /root/.gitconfig /home/docker/ \
  && cp /root/.vimrc /home/docker/ \
  && chown -R docker:docker /home/docker
```

### まとめ

いくつかDokcerやAUFSのバグもありそうな感じもしますが、
作業ユーザーもできたので、とりあえず安心してプログラムができそうです。

