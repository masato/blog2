title: "Spark on Dockerで分散型機械学習を始める - Part2: UbuntuでIPython Notebookを使う"
date: 2015-01-18 00:01:55
tags:
 - Docker
 - Spark
 - PySpark
 - IPythonNotebook
 - SequenceIQ
description: sequenceiq/sparkのDockerイメージはCentOS 6.5を使っているので、そのままではIPythonのインスト-ルに失敗してしまいます。SequenceIQにはUbuntuのbaseimageもあります。これから自分でSparkのDockerfileを作ろうと思いましたが、ちょどよいイメージがLogBaseInc/docker-spark-ipythonにありました。
---

[sequenceiq/spark](https://registry.hub.docker.com/u/sequenceiq/spark/)のDockerイメージはCentOS 6.5を使っているので、そのままではIPythonのインスト-ルに失敗してしまいます。SequenceIQには[Ubuntuのbaseimage](https://github.com/sequenceiq/docker-pam/blob/master/ubuntu-14.04/Dockerfile)もあります。これから自分でSparkのDockerfileを作ろうと思いましたが、ちょどよいイメージが[LogBaseInc/docker-spark-ipython](LogBaseInc/docker-spark-ipython)ありました。

<!-- more -->

## CentOS 6.5の場合

[sequenceiq/spark](https://registry.hub.docker.com/u/sequenceiq/spark/)

前回作成したDockerコンテナはCentOS 6.5の[baseimage](https://github.com/sequenceiq/docker-pam/blob/master/centos-6.5/Dockerfile)を使っています。

``` bash
$ cat /etc/redhat-release
CentOS release 6.5 (Final)
```

Pythonのバージョンは2.6.6です。

``` bash
$ python -V
Python 2.6.6
```

Python 2.7以上を使わないとiPythonがインストールできません。

``` bash
$ curl https://bootstrap.pypa.io/get-pip.py -o - | sudo python
$ sudo pip install ipython
Collecting ipython
  Downloading ipython-2.3.1.tar.gz (11.9MB)
    100% |################################| 11.9MB 1.1MB/s
    ERROR: IPython requires Python version 2.7 or 3.3 or above.
    Complete output from command python setup.py egg_info:
    ERROR: IPython requires Python version 2.7 or 3.3 or above.

    ----------------------------------------
    Command "python setup.py egg_info" failed with error code 1 in /tmp/pip-build-bzXfTe/ipython
```

## Ubuntu 14.04.1の場合

[LogBaseInc/docker-spark-ipython](LogBaseInc/docker-spark-ipython)を使いDockerコンテナを起動します。

``` bash
$ docker pull logbase/spark-ipython
$ docker run -d --name spark-ipython -p 8888:8888 logbase/spark-ipython
```

シェルを起動してバージョンを確認します。

```
$ docker exec -it spark-ipython /bin/bash
$ cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=14.04
DISTRIB_CODENAME=trusty
DISTRIB_DESCRIPTION="Ubuntu 14.04.1 LTS"
```

Pythonのバージョンは2.7.6、IPythonは2.3.1です。

``` bash
$ python -V
Python 2.7.6
$ ipython -V
2.3.1
```

## IPython Notebookを使う

ngrokを使いDockerホストの8888ポートをトンネルします。

``` bash
$ docker run -it --rm wizardapps/ngrok:latest ngrok $(docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" spark-ipython):8888
```

ブラウザでIPython Notebookを開いて簡単なテストで動作確認します。

https://6e7e14f6.ngrok.com/

{% img center /2015/01/18/spark-on-docker-ipython-notebook/ipython-notebook.png %}

