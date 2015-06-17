title: "MoinMoinをDigitalOceanのCoreOSに移設する - Part2: Dockerイメージの作成"
date: 2014-10-13 03:36:05
tags:
 - MoinMoin
 - sed
 - Ubuntu
 - Nginx
 - uWSGI
 - DataVolumeContainer
description: MoinMoinの移設作業は開発環境でDockerイメージを作成するところから始めます。現在はCentOS6.5 + Apache + mod_wsgiで動いていますが、Ubuntu + Nginx + uWSGIに移行してみました。データボリュームコンテナはdocker-lloydを使ってIDCFオブジェクトストレージにバックアップしたかったのですが、コードを読んでも動作がわからなかったので、これまで通りテンポラリのコンテナを経由してアーカイブの操作をします。
---

* `Update 2014-10-18`: MoinMoin in Production on CoreOS - Part4: Staging to IDCF Cloud](/2014/10/18/docker-moinmoin-idcf-coreos/)
* `Update 2014-10-15`: [Docker data volume container backing up with docker-lloyd](/2014/10/15/docker-data-volume-container-docker-lloyd/)
* `Update 2014-10-14`: [MoinMoinをDigitalOceanのCoreOSに移設する - Part3: docker-backupでデータボリュームコンテナのバックアップとリストア](/2014/10/14/docker-data-volume-container-docker-backup/)

MoinMoinの移設作業は開発環境でDockerイメージを作成するところから始めます。現在はCentOS6.5 + Apache + `mod_wsgi`で動いていますが、Ubuntu + Nginx + uWSGIに移行してみました。

データボリュームコンテナは[docker-lloyd](https://github.com/discordianfish/docker-lloyd)を使ってIDCFオブジェクトストレージにバックアップしたかったのですが、コードを読んでも動作がわからなかったので、これまで通りテンポラリのコンテナを経由してアーカイブの操作をします。

<!-- more -->

### プロジェクトの作成

MoinMoinのイメージを作成するプロジェクトの構成です。

```
$ tree ~/docker_apps/moinmoin/
~/docker_apps/moinmoin/
├── Dockerfile
├── common.css
├── nginx.conf
├── run.sh
├── screen.css
└── user
```

### Dockerfile

最初に作成したDockerfileです。dperson/moinmoinの[Dockerfile](https://github.com/dperson/moinmoin/blob/master/Dockerfile)を参考にしました。

``` bash ~/docker_apps/moinmoin/Dockerfile
FROM ubuntu:14.04
MAINTAINER Masato Shimizu <ma6ato@gmail.com>
ENV HOME /root
ENV WORKDIR /root
RUN sed -i~ -e 's/archive.ubuntu.com/ftp.jaist.ac.jp/' /etc/apt/sources.list \
 && apt-get update
RUN DEBIAN_FRONTEND=noninteractive apt-get install -qy language-pack-ja
ENV LANG ja_JP.UTF-8
RUN locale-gen $LANG && update-locale $LANG && \
    ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
RUN DEBIAN_FRONTEND=noninteractive \
  apt-get install -qy nginx uwsgi uwsgi-plugin-python python wget unzip
ADD nginx.conf /etc/nginx/sites-available/default
RUN wget http://static.moinmo.in/files/moin-1.9.7.tar.gz -P /tmp && \
    tar xzf /tmp/moin-1.9.7.tar.gz -C /tmp && \
    cd /tmp/moin-1.9.7 && \
    python setup.py install --force --prefix=/usr/local && \
    sed -e '/sitename/s/Untitled //' \
        -e '/page_front_page.*Front/s/#\(page_front_page\)/\1/' \
        -e '/superuser/ { s/#\(superuser\)/\1/; s/YourName/ScottTiger/ }' \
        -e '/page_front_page/s/#u/u/' \
        -e '/acl_rights_before/s/.*/    acl_rights_default = u"ScottTiger:read,write,delete,revert,admin"\n&/'\
        -e '/theme_default/s/modernized/europython/' \
        -e '/theme_default/s/.*/    tz_offset = 9.0\n&/' \
       /usr/local/share/moin/config/wikiconfig.py > \
       /usr/local/share/moin/wikiconfig.py && \
       chown -Rh www-data. /usr/local/share/moin/data \
                           /usr/local/share/moin/underlay && \
    wget https://bitbucket.org/thesheep/moin-europython/get/3995afe116b0.zip -P /tmp && \
    unzip /tmp/3995afe116b0.zip -d /tmp && \
    cd /tmp/thesheep* && \
    cp europython.py /usr/local/share/moin/data/plugin/theme && \
    mkdir /usr/local/lib/python2.7/dist-packages/MoinMoin/web/static/htdocs/europython && \
    cp -R css /usr/local/lib/python2.7/dist-packages/MoinMoin/web/static/htdocs/europython && \
    cp -R img /usr/local/lib/python2.7/dist-packages/MoinMoin/web/static/htdocs/europython
ADD screen.css /usr/local/lib/python2.7/dist-packages/MoinMoin/web/static/htdocs/europython/css/screen.css
ADD common.css /usr/local/lib/python2.7/dist-packages/MoinMoin/web/static/htdocs/europython/css/common.css
ADD user /usr/local/share/moin/data/user/1338720549.58.37773
ADD run.sh /run.sh
VOLUME ["/usr/local/share/moin/data/pages"]
CMD ["bash", "/run.sh"]
EXPOSE 80
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

### sedでwikiconfig.pyの編集

moin-1.9.7.tar.gzをダウンロードしたあと、wikiconfig.pyの編集をsedで編集します。
dperson/moinmoinの[Dockerfile](https://github.com/dperson/moinmoin/blob/master/Dockerfile)を参考にしました。
こういったsedの使い方は初めて見ました。奥が深いです。とても良い勉強になりました。

### Europythonのテーマ

MoinMoinのテーマは[Europython](http://moinmo.in/ThemeMarket/Europython)を使います。[pypi](https://pypi.python.org/pypi)と同じテーマなので気に入っています。
ローカルでzipを解凍してデフォルトのfont-familyを修正してからADDします。

screen.cssのfont-familyを編集します。

``` css ~/docker_apps/moinmoin/screen.css
html {
    font-family:  "lucida grande",tahoma,verdana,arial,'ヒラギノ角ゴ Pro W3', 'Hiragino Kaku Gothic Pro', 'Osaka', 'Meiryo UI','メイリオ', 'ＭＳ Ｐゴシック', sans-serif;
}
div#sidebar ul#navibar li a {
  /*font-family: Verdana,Geneva,"Bitstream Vera Sans",Helvetica,sans-serif;*/
}
```

common.cssのfont-familyを編集します。

``` css ~/docker_apps/moinmoin/common.css
h1, h2, h3, h4, h5 {
    /*font-family: serif;*/
    font-weight: normal;
    letter-spacing: 0.05em;
}
```

### Nginx

/etc/nginx/sites-available/defaultにコピーするnginx.confです。
デフォルトの設定ファイルからIPv6と`try_files`をコメントアウトします。
uWSGIとドメインソケットで通信する設定をします。

``` bash ~/docker_apps/moinmoin/nginx.conf
server {
        listen 80 default_server;
        #listen [::]:80 default_server ipv6only=on;
        root /usr/share/nginx/html;
        index index.html index.htm;
        # Make site accessible from http://localhost/
        server_name localhost;
        location / {
                #try_files $uri $uri/ =404;
                # Uncomment to enable naxsi on this location
                # include /etc/nginx/naxsi.rules
                include uwsgi_params;
                uwsgi_modifier1 30;
                uwsgi_pass unix:/tmp/moin.sock;
        }
        location /moin_static197 {
                alias /usr/local/lib/python2.7/dist-packages/MoinMoin/web/static/htdocs;
        }
}
```

### uWSGIの起動

Dockerコンテナの起動スクリプトを用意します。uWSGIをデーモンで起動したあと、Nginxをフォアグラウンドで起動します。
以前はruintやSupervisordでinitを書いていましたが、最近はこういった起動スクリプトを用意する方がDockerらしい気がします。

またデータコンテナをアタッチするときにpagesディレクトリの権限を修正するため、起動前に`www-data:www-data`に所有者を変更します。

``` bash ~/docker_apps/moinmoin/run.sh
#!/bin/bash
chown -R www-data:www-data /usr/local/share/moin/data/pages
uwsgi --uid www-data \
    -s /tmp/moin.sock \
    --plugins python \
    --pidfile /var/run/uwsgi-moinmoin.pid \
    --wsgi-file server/moin.wsgi \
    -M -p 4 \
    --chdir /usr/local/share/moin \
    --python-path /usr/local/share/moin \
    --harakiri 30 \
    --die-on-term \
    --daemonize /var/log/uwsgi/app/moinmoin.log
nginx -g 'daemon off;'
```

### 起動と確認

イメージをビルドしてコンテナを起動します。

``` bash
$ docker build -t masato/moinmoin .
$ docker tag masato/moinmoin localhost:5000/moin
$ docker push localhost:5000/moin
$ docker run --name wiki -d masato/moinmoin
$ docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" wiki
172.17.0.178
```

ブラウザで確認します。

{% img center /2014/10/13/docker-moinmoin-data-volume-container-nginx-uwsgi/moin-default.png %}



### docker-backupdのバックアップとリストアは断念

[docker-backup](https://github.com/discordianfish/docker-backup)を試してみましたが、Goのコードを読んでも挙動がよくわかりませんでした。Goはもっと勉強しないと。


### データボリュームコンテナの作成

データボリュームコンテナを作成する前に、起動しているmoinのコンテナを削除します。

``` bash
$ docker kill wiki && docker rm wiki
```

データボリュームコンテナを作成します。

``` bash
$ docker run -v /usr/local/share/moin/data/pages --name wiki-vol busybox true
```

既存のさくらVPSで動かしているMoinMoinのpagesは[Dropboxにバックアップ](/2014/04/29/dropbox-uploader-moinmoin/)しています。
データボリュームコンテナにDropboxからダウンロードしたpagesのアーカイブをuntarします。

``` bash
$ docker run --rm --volumes-from wiki-vol -v $(pwd):/backup ubuntu tar zxfv /backup/141012171613_pages.tar.gz --strip-components=1 -C /usr/local/share/moin/data/pages
```

データコンテナをアタッチしてwikiを起動

``` bash
$ docker run --name wiki -d --volumes-from wiki-vol masato/moinmoin
$ docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" wiki
172.17.0.201
```

ブラウザで確認するとACLが機能して、ログインしないとコンテンツが表示されません。

{% img center /2014/10/13/docker-moinmoin-data-volume-container-nginx-uwsgi/moin-acl.png %}

### 管理者ユーザーの作成

この設定ではユーザーは管理者ユーザー(ScottTiger)のみ操作が可能です。

``` python /usr/local/share/moin/wikiconfig.py
acl_rights_default = u'ScottTiger:read,write,delete,revert,admin'
```

従って既存のデータを移行する場合は、`/usr/local/share/moin/data/user/`のディレクトリにあるユーザー定義ファイルをコピーして追加しないと、何も操作ができなくなります。

``` bash
ADD user /usr/local/share/moin/data/user/1338720549.58.37773
```

予め登録してある管理者ユーザーでログインするとWikiページが表示されました。

{% img center /2014/10/13/docker-moinmoin-data-volume-container-nginx-uwsgi/moin-acl-admin.png %}
