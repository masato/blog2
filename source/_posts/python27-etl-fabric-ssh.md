title: "Python2.7でETL - Part2: FabricをPythonプログラムから使ってSFTPする"
date: 2014-07-20 21:37:23
tags:
 - Python
 - ETL
 - Fabric
description: Pythonでリモートホストへファイル転送を使う場合に、いくつか選択肢があります。os.systemでscpコマンドを使う subprocess.Popenでscpコマンドを使う Paramikoを使ったPure python scp module 今回はFabricのfabric.api.putモジュールをPythonのプログラムから利用してSFTP転送を試してみます。
---

Pythonでリモートホストへファイル転送を使う場合に、いくつか選択肢があります。

* os.systemでscpコマンドを使う
* subprocess.Popenでscpコマンドを使う
* Paramikoを使った[Pure python scp module](https://github.com/jbardin/scp.py)

今回はFabricの`fabric.api.put`モジュールをPythonのプログラムから利用してSFTP転送を試してみます。

<!-- more -->

### Pythonスクリプト

SSHの設定情報をPythonファイルにします。JSONやYAMLを使わなくてもPythonの辞書はそのまま設定ファイルになります。

``` python settings.py
# -*- coding: utf-8 -*-
import os

SSH = {
  "PRIVATE_KEY": "/home/masato/.ssh/id_rsa",
  "REMOTE_HOST": "remote_host",
  "USERNAME": "docker"
}
```

ファイル転送のスクリプトを用意します。

``` python deploy.py
# -*- coding: utf-8 -*-

import os
from fabric.context_managers import settings
from fabric.api import (run,put)
import settings as myconf

def deploy_csv(file_path):

    if not os.path.exists(file_path):
        return

    with settings(host_string=myconf.SSH["REMOTE_HOST"], user=myconf.SSH["USERNAME"] ,
                  key_filename=myconf.SSH["PRIVATE_KEY"]):
        put(file_path, '/home/{0}/csv_inbox'.format(myconf.SSH["USERNAME"]))
        os.remove(file_path)
```

### まとめ

Fabricは通常fabコマンドで実行するのですが、APIを使ってPythonのプログラムからも実行できます。
私は`Python ETL`のプログラムの中でファイル転送にFabicのAPIを利用しています。

PythonのSQLSoupでDBからクエリしてゴニョゴニョ前処理した後、CSVファイルを作成します。
作成したCSVを同じPythonプログラムの中から手軽にリモートホストにCSVを転送できると便利です。

Fabricを使っているので、CSVを転送した後、リモートホストで行いたい処理もrunで記述できます。

カジュアルな`Python ETL`の用途では十分だと思います。

