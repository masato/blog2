title: "Visual Studio Community 2013にCUDA Toolkit 6.5をインストールする"
date: 2015-02-19 19:20:17
tags:
 - DeepLearning
 - Torch
 - Thenao
 - CUDA
 - VisualStudioCommunity2013
 - GeForce
description: 最近話題のDeepLearningのフレームワーク、TorchやThenaoではGPUを使った計算をサポートしています。手元にあるノートパソコンのXPS 14プレミアムモデルはちょっと古いですがGeForce GT 630Mというグラフィックカードを搭載しているのでCUDAをサポートしていました。Maxwellを搭載したALIENWARE 15が発売されてすごく気になりますが、今の環境でもとりあえずCUDAの勉強をはじめることができそうです。
---

最近話題のDeepLearningのフレームワーク、[Torch](http://torch.ch)や[Thenao](http://deeplearning.net/software/theano/index.html)ではCUDAのGPUを使った計算をサポートしています。手元にあるノートパソコンの[XPS 14 プレミアムモデル](http://pc.watch.impress.co.jp/docs/column/nishikawa/20120731_550139.html)とちょっと古いですが[GeForce GT 630M](http://www.nvidia.co.jp/object/geforce-gt-630-jp.html)というグラフィックカードを搭載しているのでCUDAをサポートしていました。Maxwellを搭載した[ALIENWARE 15](http://www.dell.com/jp/p/alienware-15/pd)が発売されてすごく気になりますが、今の環境でもとりあえずCUDAの勉強をはじめることができそうです。

<!-- more -->

## XPS 14 と GeForce GT 630M

2年前に購入したノートPCですがまだまだ現役で使えて気に入っています。ゲームは全くやらないのでこれまで意識したことがなかったのですが[GeForce GT 630M](http://www.nvidia.co.jp/object/geforce-gt-630-jp.html)というグラフィックカードを搭載していました。ようやく出番がきた感じです。CUDA Toolkitをインストールするとデバイス情報を確認するdeviceQueryというツールが使えるので確認してみます。

* VRAMの容量: 1024 MBytes
* 搭載されている全GPUコア数: 98コア(プロセッサーの数: 2, プロセッサー当たりのコア数: 48)

``` dos
> cd C:\ProgramData\NVIDIA Corporation\CUDA Samples\v6.5\bin\win64\Release
> deviceQuery.exe
deviceQuery.exe Starting...

 CUDA Device Query (Runtime API) version (CUDART static linking)

Detected 1 CUDA Capable device(s)

Device 0: "GeForce GT 630M"
  CUDA Driver Version / Runtime Version          6.5 / 6.5
  CUDA Capability Major/Minor version number:    2.1
  Total amount of global memory:                 1024 MBytes (1073741824 bytes)
  ( 2) Multiprocessors, ( 48) CUDA Cores/MP:     96 CUDA Cores
  GPU Clock rate:                                1250 MHz (1.25 GHz)
  Memory Clock rate:                             2000 Mhz
  Memory Bus Width:                              64-bit
  L2 Cache Size:                                 131072 bytes
...
```

## Visual Studio Community 2013

[Visual Studio Community 2013](http://www.visualstudio.com/ja-jp/downloads/download-visual-studio-vs#d-community)は2014年末に発表されました。Professionalエディションと同等の機能が無償で使えます。個人で開発するにはとてもうれしいエディションです。


### インストール

[ダウンロードページ](http://www.visualstudio.com/ja-jp/downloads/download-visual-studio-vs#d-community)のリンクから、`Microsoft Visual Studio Community 2013 Update 4 - 英語`のインストーラーをダウンロードします。オプションをいろいろはずしても8GBの空き容量が必要です。インストールには1時間くらいかかりました。

* インストーラー: vs_community.exe
* 所要時間: 1時間

### 日本語化

Visual Studio 2013 Language Packは日本語版をインストールします。さきほどのダウンロードページとは別の[ページ](https://www.microsoft.com/ja-jp/download/details.aspx?id=40783)から日本語を選択してダウンロードします。こちらはインストールに30分くらいかかります。

* インストーラー: vs_langpack.exe
* 所要時間: 30分

インストール中にエラーが出ます。Visual Studioを再起動しても日本語化されていないので失敗したように見えます。

![vs2013-lang-warn.png](/2015/02/19/visual-stuidio-community-2013-cuda-65/vs2013-lang-warn.png)

環境設定から日本語を選択すると日本語のメニュー表示になりました。

* TOOLS > Options > Environment > International Settings > 日本語

### インストールをやり直す場合

一度Visual Stuidioのインストールをやり直しました。コントロールパネルのプログラムと機能からVisual Stuidioをアンインストールしても何か残っているようでインストーラーの起動に失敗してしまいます。

> Visual Studio Professional 2013 is currently installed on this machine. Please unisnstall Visual Studio Professional 2013 and retry.

Professional版はインストールしていないのでおかしなエラーメッセージです。[Visual Studio 2013 Language Pack でエラーが出た時の対処](http://silight.hatenablog.jp/entry/2014/12/05/020506)のページを参考にさせていただくと原因はLanguage Packにあるようです。

コマンドプロンプトから`uninstall`フラグを指定すると正常にアンインストールができて、Visual Studioのインストーラーがもう一度起動するようになりました。

```
vs_langpack.exe /uninstall
```

## CUDA Toolkit 6.5

[Download](https://developer.nvidia.com/cuda-downloads)ページから、Windows 7 64bit版の[インストーラー]( http://developer.download.nvidia.com/compute/cuda/6_5/rel/installers/cuda_6.5.14_windows_general_64.exe)をダウンロードしてインストールします。インストールには1時間くらいかかるので気長に待ちます。

* インストーラー: cuda_6.5.14_windows_general_64.exe
* 所要時間: 1時間

CUDAのインストールが完了しました。バージョンを確認します。

* ヘルプ > Microsoft Visual Studioのバージョン情報

![vs2013-version.png](/2015/02/19/visual-stuidio-community-2013-cuda-65/vs2013-version.png)

### Nsight Visual Studio Edition 4.5

CUDA Toolkit 6.5でインスト-ルされるNSIGHTのバージョンは`4.1.0.14204`です。新しい4.5のRCが利用可能になっていると[メッセージ](https://developer.nvidia.com/content/nsight-visual-studio-edition-45-release-candidate-1-available-download)が出るので、`Developer Program Membership`に登録して[ダウンロード](https://developer.nvidia.com/gameworksdownload#?dn=nsight-visual-studio-edition-4-5-0)してインストールします。Visual Studioのバージョンを確認するとなぜかNsightが1.0になっています。

![vs2013-nsight.png](/2015/02/19/visual-stuidio-community-2013-cuda-65/vs2013-nsight.png)

NSIGHTメニューからバージョンを確認すると4.5がインストールされているので成功したようです。

* NSIGHT > Help > About Nsight...

![vs2013-nsight-45.png](/2015/02/19/visual-stuidio-community-2013-cuda-65/vs2013-nsight-45.png)

### 開発者向けドキュメントとコンテンツ

[CUDA ZONE](https://developer.nvidia.com/cuda-zone)には開発者用ドキュメントがたくさんあります。[Getting Started](https://developer.nvidia.com/get-started-parallel-computing)など並列コンピューティングの勉強はここからはじめると良さそうです。

また別途登録が必要ですが[qwikLAB](https://nvidia.qwiklab.com/)が提供するオンライン学習コンテンツもあります。実際にEC2のGPUインスタンスとIPython Notebookを使いながらインタラクティブにクエストをクリアしていく形で勉強できます。一部は無料で試すことができます。

