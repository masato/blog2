title: "IPython Notebookの練習 - Part1: はじめに"
date: 2014-08-11 23:14:18
tags:
 - AnalyticSandbox
 - IPythonNotebook
 - Bluemix
 - Exercise
description: Dockerでデータ分析環境 - Part7 Ubuntu14.04のDockerにIPython NotebookをデプロイでようやくDocker上にデータ分析環境ができました。今日からIPython Notebookを使ったデータ分析の練習をしていきます。ここ数週間風邪をひいてしまいIBM Bluemix Challengeに間に合いませんでした。developerworksにRun IPython Notebook on IBM Bluemixという記事があったので、CloudFoundryで試したかったです。結局DockerとOpenRestyとxip.ioを使ってCloudFoundryでやりたかったことは実現できたのですが、あたらしい環境セットを考えると、BuildpackのようなBuildstepも使えるようにしていきたいです。
---

[Dockerでデータ分析環境 - Part7: Ubuntu14.04のDockerにIPython Notebookをデプロイ](/2014/08/10/docker-analytic-sandbox-ipython-notebook-deploy/)でようやくDocker上にデータ分析環境ができました。今日から`IPython Notebook`を使ったデータ分析の練習をしていきます。

ここ数週間風邪をひいてしまい`IBM Bluemix Challenge`に間に合いませんでした。
developerworksに[Run IPython Notebook on IBM Bluemix](http://www.ibm.com/developerworks/cloud/library/cl-ipython-app/index.html)という記事があったので、CloudFoundryで試したかったです。

結局DockerとOpenRestyとxip.ioを使ってCloudFoundryでやりたかったことは実現できたのですが、あたらしい環境セットを考えると、BuildpackのようなBuildstepも使えるようにしていきたいです。

<!-- more -->


### IPython の起動

IPythonのrunit起動スクリプトは、`--pylab inline`を使っていないので、このままでは画面上にグラフを表示することができません。

``` bash ~/docker_apps/ipython/sv/ipython/run
#!/bin/sh

exec 2>&1
exec /root/anaconda/bin/ipython notebook --no-browser --profile nbserver --ip=0.0.0.0 --port 8080 --notebook-dir=/notebook
```

### notebook を書く

先ほどのBluemixのサンプルです。`--pylab inline`フラグで暗黙的にモジュールをしないので、
notebook内で明示的に`%matplotlib inline`を使います。

``` python
%matplotlib inline

import matplotlib.pyplot as plt
import numpy as np

x = np.arange(0, 4*np.pi, 0.05)
y = [np.sin(i) for i in x]

plt.plot(x, y)
```

{% img center /2014/08/11/ipython-notebook-exercise-part1/ipython-plot.png %}
