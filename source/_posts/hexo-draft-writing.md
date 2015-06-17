title: 'Hexoでドラフトをプレビューする'
date: 2014-07-16 00:30:14
tags:
 - Hexo
 - NitrousIO
description: Hexoでは投稿を投稿しないでドラフト状態のままプレビューすることができます。思いついたことをメモしておいたり、これからやりたいことを忘れないためにToDo的にリストできます。いつも使い方を忘れてしまうのでこれもメモです。
---

Hexoでは投稿を投稿しないでドラフト状態のままプレビューすることができます。
思いついたことをメモしておいたり、これからやりたいことを忘れないためにToDo的にリストできます。

いつも使い方を忘れてしまうのでこれもメモです。

### ドラフトページの作成

[Writing](http://hexo.io/docs/writing.html)のページに書式があります。通常のページ作成コマンドに続いて、`draft`を追加するだけです。

``` bash
$ cd ~/workspace/blog
$ hexo new draft hexo-draft-writing
[info] File created at /home/action/workspace/blog/source/_drafts/hexo-draft-writing.md
```

### ドラフトページのプレビュー

こちらも通常のローカルサーバー起動コマンドに、`-d`オプションを追加します。
[Built-in Server](http://hexo.io/docs/server.html)のページに書式があります。
`_drafts/`のページも、`_posts/`にマージしてプレビューできます。

``` bash
$ hexo server -d
[info] Using drafts.
[info] Hexo is running at http://localhost:4000/. Press Ctrl+C to stop.
```

私はHexoはNitrous.IOを使って書いているので、メニューからブラウザが起動してドラフト画面が表示できます。

```
Preview -> Port 4000
```

### ドラフトの公開

ファイルをエディタで開いたままだと、保存したときに`_drafts`ディレクトリに再度ファイルが作成されてしまうので、
エディタからファイルをクローズしてから、`_posts`ディレクトリに移動します。

``` bash
$ mv source/_drafts/hexo-draft-writing.md source/_posts/
```

### まとめ

DockerでPaaSをインストールして遊んでいますが、Nitrous.IOの手軽さには及びません。
まだ無料プランで使っているのが申し訳ないくらい。
自分で作ったtrusuやDeisのインタフェースに使えるWebIDEが何か欲しいです。

