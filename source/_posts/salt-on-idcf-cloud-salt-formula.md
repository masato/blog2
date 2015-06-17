title: 'Salt on IDCF クラウド - Part2: Salt Formula'
date: 2014-11-18 23:53:20
tags:
 - Salt
 - IDCFクラウド
 - IDCFCloud
 - Ubuntu
 - SaltFormula
---

Saltで使われる用語は非英語圏だと直感的にわかりづらいものが多いですが、`Salt States`や`Salt State System`は、Saltの構成管理ツールとしての中心的な仕組みです。`Salt State`はサーバーの構成管理に伴う、パッケージのインストール、設定ファイルの作成、サービスの起動、ユーザー管理などを行います。Puppet の manifests や Chef の recipes に相当します。

<!-- more -->

### SLSファイル

`Salt Formula`の`SLS(SaLt State)`ファイルは拡張子が`.sls`のファイルにYAML形式です。SLSファイルには`Salt States`を記述して、masterノードのファイルシステムに配置します。SLSファイルにIDと一緒に定義したstatesは順番通り実行できますが、依存関係を宣言的に定義することもできます。

### Salt Formula

`Salt Formula`はmasterの`/srv/salt/`ディレクトリに配置します。saltコマンドから`state`モジュールを使い個別にSLSファイルを実行したり、SLSファイル全体を実行することができます。また、SLSファイル内の特定のIDをもったstateだけ実行することもできます。

### masterのディレクトリ構成

masterにFormulaを配置するディレクトリを作成します。

``` bash
$ mkdir -p /srv/salt
```

今回作成するFormulaのディレクトリ構造です。

``` bash
$ tree /srv/salt/
/srv/salt/
├── base.sls
├── emacs
│   └── init.sls
└── top.sls
```

### top.sls

top.slsは、`Salt States`のトップファイルです。このファイルが最初に実行されます。`'*'`に定義されたSLSファイルはすべてのminionで実行されます。この例ではGrainsのrolesの定義に従って適用するminionを決定します。`roles:dev`をGrainsで指定しているminion1とminion2にはemacsのstateが適用されます。

``` yml /srv/salt/top.sls
base:
  '*':
    - base
  'roles:salt-master':
    - match: grain
  'roles:dev':
    - match: grain
    - emacs
```

### base.sls

base.slsは`/srv/salt/`直下に配置します。共通の日本語環境を設定するため、`locale`と`timezone`stateの`system`functionを使いました。

``` yml /srv/salt/base.sls
ja_JP.UTF-8:
  locale.system

Asia/Tokyo:
  timezone.system
```

### emacs Formula

`/srv/salt/`の下にディレクトリを作成する場合はinit.slsを作成します。ディレクトリ名はsaltコマンドのstatesモジュールに渡すstateの名前になります。`pkg`stateの`installed`functionを使い、emacsのパッケージをインストールします。

``` yml /srv/salt/emacs/init.sls
emacs:
  pkg.installed:
    - names:
      - emacs24-nox
      - emacs24-el
```

### statesモジュールで個別のFormulaを適用する

`states`モジュールで個別のFormulaをminionに適用することができます。以下ではGrailsに定義したrolesを参照してminionを指定します。

``` bash
$ salt -G 'roles:dev' state.sls emacs
```

minionにインストールしたEmacsのバージョンを確認してみます。IDCFクラウドの場合はすでにemacs24-noxはインストール済みなので、emacs24-elパッケージのみインストールされました。

``` bash
$ salt 'minion1*' cmd.run 'emacs --version'
minion1.cs74idcfcloud.internal:
    GNU Emacs 24.3.1
    Copyright (C) 2013 Free Software Foundation, Inc.
    GNU Emacs comes with ABSOLUTELY NO WARRANTY.
    You may redistribute copies of Emacs
    under the terms of the GNU General Public License.
    For more information about these matters, see the file named COPYING.
```

### stateモジュールで全てのFormulaを適用する

`highstate`functionを使い、`Salt Formula`のすべてのSLSファイルをminionに適用します。`'*'`のワイルドカードを指定して、すべてのminionを対象にします。

``` bash
$ salt '*' state.highstate
```