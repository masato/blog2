title: "Salt on IDCF クラウド - Part4: psutilとpsモジュールで監視"
date: 2014-11-20 22:52:00
tags:
 - Salt
 - IDCFクラウド
 - IDCFCloud
 - psutil
description: Salt チュートリアル - Part4 Saltを使った監視インフラで少し書きましたが、Saltのpsモジューを使うと、監視システムに必要なメトリクスをminionから集めることができます。今回はSLSファイルを作成しないで、psutilをpipモジュールを使いインストールしてみます。
---

[Salt チュートリアル - Part4: Saltを使った監視インフラ](/2014/09/17/salt-tutorials-monitoring/)で少し書きましたが、Saltの[psモジュール](http://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.ps.html#module-salt.modules.ps)を使うと、監視システムに必要なメトリクスをminionから集めることができます。今回はSLSファイルを作成しないで、psutilをpipモジュールを使いインストールしてみます。

<!-- more -->

### python-devのインストール

[psモジュール](http://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.ps.html#module-salt.modules.ps)の実行にはPythonの[psutil](https://github.com/giampaolo/psutil)が必要です。psutilのビルドにはpython-devパッケージが必要になるため、common.slsに追加します。

``` yml /srv/salt/common.sls
pkg-core:
  pkg.installed:
    - pkgs:
      - git
      - python-pip
      - python-dev
```

state.highstateを実行して、minionに適用します。

``` bash
$ salt '*' state.highstate
```

### pipモジュール

[pipモジュール](http://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.pip.html)を使いpsutilをインストールします。

``` bash
$ salt '*' pip.install upgrade=True psutil
```

### psモジュール

Pythonのpsutilをインストールしたので、Saltの[psモジュール](http://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.ps.html#module-salt.modules.ps)が使えるはずですが、なぜか利用できません。

``` bash
$ salt 'minion1*' ps.disk_usage "/"
minion1.cs74idcfcloud.internal:
    'ps.disk_usage' is not available.
```

pipモジュールのlistファンクションで確認すると、psutilモジュールはインストール済みです。

``` bash
$ salt 'minion*' pip.list psutil
minion2.cs74idcfcloud.internal:
    ----------
    psutil:
        2.1.3
minion1.cs74idcfcloud.internal:
    ----------
    psutil:
        2.1.3
```


### salt-minionのリスタート


psutilを有効にするために、salt-minionのrestartが必要のようです。以下のSLSファイルを作成します。

``` yml /srv/salt/misc.sls
restart_minion:
  cmd.run:
    - name: |
        nohup /bin/sh -c 'sleep 10 && salt-call --local service.restart salt-minion'
    - python_shell: True
    - order: last
```

メンテナンス用なのでtop.slsには追加せず、直接SLSファイルを指定します。

``` bash
$ salt -G 'roles:dev' state.sls_id  restart_minion misc
```

### ps.disk_usageの実行

``` bash
$ salt -G 'roles:dev' ps.disk_usage "/"
minion1.cs74idcfcloud.internal:
    ----------
    free:
        12889178112
    percent:
        12.8
    total:
        15717089280
    used:
        2005934080
minion2.cs74idcfcloud.internal:
    ----------
    free:
        12889124864
    percent:
        12.8
    total:
        15717089280
    used:
        2005987328

```
