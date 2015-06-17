title: "Dockerでデータ分析環境 - Part5: RStudio ServerとData-onlyコンテナ"
date: 2014-08-08 00:27:23
tags:
 - Docker
 - AnalyticSandbox
 - RStudioServer
 - R
 - DataVolumeContainer
 - docker-registry
description: RStudio Serverをdisposableに本番環境へデプロイする準備として、ホームディレクトリをData-onlyコンテナへ分離します。今回作成するData-onlyコンテナにはフットプリントの小さいbusyboxを使います。RStudio Server学習者に2コンテナずつセットで提供します。Data-onlyコンテナはアーカイブしてIDCFオブジェクトストレージにバックアップする予定です。
---

[RStudio Server](/2014/07/09/docker-analytic-sandbox-rstudio-server/)をdisposableに本番環境へデプロイする準備として、ホームディレクトリをData-onlyコンテナへ分離します。

今回作成するData-onlyコンテナにはフットプリントの小さいbusyboxを使います。`RStudio Server`学習者へRStudioとデータの2コンテナをセットで提供します。

Data-onlyコンテナはアーカイブしてIDCFオブジェクトストレージにバックアップする予定です。

<!-- more -->

### rstudioユーザーのUIDとGID


`RStudio Server`コンテナに作成したユーザーのUIDとGIDを確認します。

``` bash /etc/passwd
rstudio:x:1000:1001::/home/rstudio:
```

### ドットファイルのコピー

まずプロジェクトを作成します。

``` bash
$ mkdir ~/docker_apps/rstudio_data
$ cd !$
$ mkdir dotfiles
```

Data-onlyにrstudioユーザーのホームディレクトリを作成するので、基本的なドットファイルを`Rstudio Server`コンテナからコピーします。

``` bash
$ docker cp 6a373a56e271:/etc/skel/.bash_logout dotfiles
$ docker cp 6a373a56e271:/etc/skel/.bashrc dotfiles
$ docker cp 6a373a56e271:/etc/skel/.profile dotfiles
```

### Data-onlyコンテナの作成

busyboxをbaseimageに使い、Data-onlyコンテナのDockerfileを作成します。
VOLUMEディレクトリは`RStudio Server`の`/home/rstudio/`ディレクトリをマウントするので、
UIDとGIDを、上記で確認した1000:1001に合わせます。

``` bash ~/docker_apps/rstudio_data/Dockerfile
FROM busybox
MAINTAINER Masato Shimizu <ma6ato@gmail.com>
RUN mkdir /tmp/dotfiles
ADD dotfiles /tmp/dotfiles
RUN mkdir /home/rstudio \
  && cp /tmp/dotfiles/.* /home/rstudio \
  && chown -R 1000:1001 /home/rstudio
VOLUME /home/rstudio
CMD /bin/sh
```

イメージをビルドします。

``` bash
$ docker build -t masato/rstudio-data
```

Data-onlyコンテナを起動します。

```
$ docker run -i -t -d --name rstudio-data masato/rstudio-data
```

### ローカルのdocker-registryにpush

あとで本番環境にデプロイするため、ローカルのdocker-registryにpushします。

``` bash
$ curl localhost:5000
"docker-registry server (dev) (v0.7.3)"
$ docker tag masato/rstudio-data localhost:5000/rstudio-data
$ docker push localhost:5000/rstudio-data
...
Pushing tag for rev [710a7212b5b7] on {http://localhost:5000/v1/repositories/rstudio-data/tags/latest}
```

### ボリュームをマウントしてRStudio Serverコンテナを起動

``` bash
$ docker run --rm -i -t --volumes-from rstudio-data masato/rstudio-server /sbin/my_init bash
```

### ダミーファイルの作成

`RStudio Server`のrstudioユーザーのホームディレクトリにダミーファイルを作成します。

``` bash
# sudo - rstudio
$ echo hello > hello.txt
```

### Data-onlyコンテナのvfsを確認

Data-onlyコンテナのvfsのディレクトリの場所をinspectします。

``` bash
$ docker inspect 11a84ee4e500 | grep "vfs/dir"
        "/home/rstudio": "/var/lib/docker/vfs/dir/1fc752b41a7ef3d749152581e9482bedba9085817afe82da54accc9857668767"
```

Dockerホスト上から、Data-onlyコンテナに作成されたファイルを確認します。

``` bash
$ sudo ls -altr /var/lib/docker/vfs/dir/1fc752b41a7ef3d749152581e9482bedba9085817afe82da54accc9857668767
合計 80
-rw-r--r--   1 matedev admin   675  8月  7 22:56 .profile
-rw-r--r--   1 matedev admin  3637  8月  7 22:56 .bashrc
-rw-r--r--   1 matedev admin   220  8月  7 22:56 .bash_logout
drwx------ 455 root    root  57344  8月  7 23:05 ..
-rw-rw-r--   1 matedev admin     6  8月  7 23:06 hello.txt
drwxr-xr-x   2 matedev admin  4096  8月  7 23:06 .
```

### Data-onlyコンテナにアタッチして確認

Data-onlyコンテナに直接アタッチして、作成したファイルを確認します。

``` bash
$ docker start 11a84ee4e500
$ docker attach 11a84ee4e500
/ # cd /home/rstudio/
/home/rstudio # ls
R          hello.txt
```

### まとめ

Data-onlyボリュームのバックアップはServerFaultの[Docker volume backup and restore](http://serverfault.com/questions/576490/docker-volume-backup-and-restore)を参考にして試してみます。

[docker-backup](https://github.com/discordianfish/docker-backup)や、S3にバックアップする[docker-lloyd](https://github.com/discordianfish/docker-lloyd)といったライブラリが紹介されています。
