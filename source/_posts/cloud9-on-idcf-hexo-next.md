title: "Cloud9をIDCFクラウドで使う - Part2: HexoブログをGitHub Pagesにデプロイする"
date: 2015-06-17 14:39:56
tags:
tags:
 - Cloud9
 - Nodejs
 - IDCFクラウド
 - Hexo
 - DockerCompose
description: 先日IDCFクラウドのDocker上に構築したCloud9環境に、Node.js製blogframeworkのHexoを移設します。ほぼ1年間Nitrous.IO上のHexoでブログを書いていたので、ファイル数が増え無料プランだとMarkdownのコンパイルの時間が増えるようになりプレビューがストレスでした。Nitrous.IO LITE終了がきっかけでしたが、自分クラウドのDocker環境に移設してとても快適になり結果的によかったです。
---


[先日](/2015/06/08/cloud9-on-idcf-install/)IDCFクラウドのDocker上に構築した[Cloud9](https://github.com/c9/core/)環境に、Node.js製blogframeworkの[Hexo](https://hexo.io/)を移設します。ほぼ1年間Nitrous.IO上のHexoでブログを書いていたので、ファイル数が増え無料プランだとMarkdownのコンパイルの時間が増えるようになりプレビューがストレスでした。Nitrous.IO LITE終了がきっかけでしたが、自分クラウドのDocker環境に移設してとても快適になり結果的によかったです。

<!-- more -->

## 移設前のHexoの環境

パッケージの更新を怠っているのでバージョンは少し古く3.0.0を使っています。現在の最新版は`3.1.1`なのでこの機会にアップデートも行います。

```json:/workspace/blog/package.json
{
  "name": "hexo-site",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "hexo": "^3.0.0",
    "hexo-deployer-git": "0.0.4",
    "hexo-generator-archive": "^0.1.1",
    "hexo-generator-category": "^0.1.0",
    "hexo-generator-feed": "^1.0.1",
    "hexo-generator-index": "^0.1.0",
    "hexo-generator-sitemap": "^1.0.1",
    "hexo-generator-tag": "^0.1.0",
    "hexo-renderer-ejs": "^0.1.0",
    "hexo-renderer-marked": "^0.2.4",
    "hexo-renderer-stylus": "^0.2.0",
    "hexo-server": "^0.1.2"
  },
  "scripts": {"start": "node server.js"}
}
```

## Hexoのインストール

npmから[hexo-cli](https://github.com/hexojs/hexo-cli)をインストールします。Dockerコンテナ環境なのでnpmをグローバルにインストールしても環境が汚れることを気にしなくて済みます。旧環境の`blog/source`ディレクトリをGitHub上で管理しているので新環境にHexoをクリーンインストールした後、`git clone`してsourceディレクトリをコピーしようと思います。

### npm install

インストールの手順は[Hexo](https://hexo.io/)に書いてあるようにとても簡単です。Cloud9をWebブラウザから開いてコンソールを使います。

```bash
$ cd /workspace
$ npm install hexo-cli -g
$ hexo init blog
```

ここで一度commitします。

```bash
$ cd blog
$ git config --global user.email "ma6ato@gmail.com"
$ git config --global user.name "Masato Shimizu"
$ git init
$ git add -A
$ git commit -m 'first commit'
```

旧環境ではいくつかプラグインを使っているので追加でインストールします。

```bash
$ npm install hexo-deployer-git hexo-generator-feed hexo-generator-sitemap --save
```

最終的に以下のようなpackage.jsonになりました。

```json:/workspace/blog/package.json
{
  "name": "hexo-site",
  "version": "0.0.0",
  "private": true,
  "hexo": {
    "version": "3.1.1"
  },
  "dependencies": {
    "hexo": "^3.1.0",
    "hexo-deployer-git": "0.0.4",
    "hexo-generator-archive": "^0.1.2",
    "hexo-generator-category": "^0.1.2",
    "hexo-generator-feed": "^1.0.2",
    "hexo-generator-index": "^0.1.2",
    "hexo-generator-sitemap": "^1.0.1",
    "hexo-generator-tag": "^0.1.1",
    "hexo-renderer-ejs": "^0.1.0",
    "hexo-renderer-marked": "^0.2.4",
    "hexo-renderer-stylus": "^0.2.0",
    "hexo-server": "^0.1.2"
  }
}
```

デフォルトの状態でプレビューをして確認します。デフォルトは4000 portですが、Cloud9コンテナはDockerホストの5000 portにマップしているので`-p`フラグから指定します。

```bash
$ hexo server -p 5000
INFO  Hexo is running at http://0.0.0.0:5000/. Press Ctrl+C to stop.
```

WebブラウザからIDCFクラウドのDockerのCloud9コンテナで起動しているHexoサーバーに接続します。ややこしいです。

![hexo-default.png](/2015/06/17/cloud9-on-idcf-hexo-next/hexo-default.png)


ちゃんとページが表示されているので、ここでcommitしておきます。

```bash
$ git add -A
$ git commit -m 'plugin install'
```

## Cloud9のエディタ設定

HexoのブログはMarkdownで記述します。Cloud9は日本語も使えますが、エディタの設定を折り返しにしないととても書きにくいです。エディタの右下にあるギアアイコンをクリックして`Wrap lines`にチェックをいれます。

![markdown-ja.png](/2015/06/17/cloud9-on-idcf-hexo-next/markdown-ja.png)


## NexTテーマ

Hexoは[テーマ](https://github.com/hexojs/hexo/wiki/themes)を変更することでブログのテイストを変更することができます。先ほど確認した画面はデフォルトの[landscape](https://github.com/hexojs/hexo-theme-landscape)です。久しぶりにテーマのページを除いてみたらまた増えてました。好みのテーマを探すのも楽しいです。最近は[NexT](https://github.com/iissnan/hexo-theme-next)が気に入っています。

テーマのインストールも簡単で、GitHubからthemeディレクトリにcloneするだけです。

blogインスタンスのディレクトリをgitで管理しているのでサブモジュールとしてcloneします。


```bash
$ cd /workspace/blog
$ git submodule add https://github.com/iissnan/hexo-theme-next themes/next
```

プレビューするとこんな感じです。

![hexo-next.png](/2015/06/17/cloud9-on-idcf-hexo-next/hexo-next.png)

## 設定変更

Hexoのblogインスタンスの設定と、NexTテーマの設定をそれぞれ変更します。

### blog/_config.yml

設定方法は好みですが以下のdiffのように編集しました。YAMLの設定ファイルなのでわかりやすいです。

```bash
$ diff --git a/_config.yml b/_config.yml
index 57e708e..5de1f0e 100644
--- a/_config.yml
+++ b/_config.yml
@@ -3,16 +3,18 @@
 ## Source: https://github.com/hexojs/hexo/
 
 # Site
-title: Hexo
+title: "masato's blog"
 subtitle:
-description:
-author: John Doe
-language:
-timezone:
+description: "IoT, RaspberryPi, Arduino, Meshblu, Docker, Node.js, Clojure, ClojureScript"
+author: "Masato Shimizu"
+email: ma6ato@gmail.com
+language: default
+avatar: /images/profile.jpg
+timezone: Asia/Tokyo
 
 # URL
 ## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
-url: http://yoursite.com
+url: http://masato.github.io/
 root: /
 permalink: :year/:month/:day/:title/
 permalink_defaults:
@@ -34,14 +36,13 @@ titlecase: false # Transform title into titlecase
 external_link: true # Open external links in new tab
 filename_case: 0
 render_drafts: false
-post_asset_folder: false
+post_asset_folder: true
 relative_link: false
 future: true
 highlight:
   enable: true
   line_number: true
-  auto_detect: true
-  tab_replace:
+  tab_replace: true
 
 # Category & Tag
 default_category: uncategorized
@@ -52,7 +53,8 @@ tag_map:
 ## Hexo uses Moment.js to parse and display date
 ## You can customize the date format as defined in
 ## http://momentjs.com/docs/#/displaying/format/
-date_format: YYYY-MM-DD
+#date_format: YYYY-MM-DD
+date_format: MMM D YYYY
 time_format: HH:mm:ss
 
 # Pagination
@@ -63,9 +65,36 @@ pagination_dir: page
 # Extensions
 ## Plugins: http://hexo.io/plugins/
 ## Themes: http://hexo.io/themes/
-theme: landscape
+theme: next
 
 # Deployment
 ## Docs: http://hexo.io/docs/deployment.html
 deploy:
-  type:
+  type: git
+  repo: git@github.com:masato/masato.github.io.git
+
+sitemap:
+  path: sitemap.xml
+
+feed:
+  type: rss2
+  path: rss2.xml
+  limit: 20
+
+tag_generator:
+  per_page: 10
+
+archive_generator:
+  per_page: 10
+  yearly: true
+  monthly: true
+
+
+# Archives
+## 2: Enable pagination
+## 1: Disable pagination
+## 0: Fully Disable
+archive: 2
+category: 2
+tag: 2
```

### themes/next/_config.yml

NexTテーマも同様に編集します。こちらにはテーマ固有の設定を定義しています。

```text
diff --git a/_config.yml b/_config.yml
index feb2741..b5ade09 100755
--- a/_config.yml
+++ b/_config.yml
@@ -11,18 +11,18 @@ menu:
 favicon: /favicon.ico
 
 # Set default keywords (Use a comma to separate)
-keywords: "Hexo,next"
+keywords: "IoT,RaspberryPi,Arduino,Meshblu,Docker,Node.js,Clojure,ClojureScript"
 
 # Set rss to false to disable feed link.
 # Leave rss as empty to use site's feed link.
 # Set rss to specific value if you have burned your feed already.
-rss:
+rss: /rss2.xml
 
 # Icon fonts
 # Place your font into next/source/fonts, specify directory-name and font-name here
 # Avialable: default | linecons | fifty-shades | feather
-icon_font: default
-#icon_font: fifty-shades
+#icon_font: default
+icon_font: fifty-shades
 #icon_font: feather
 #icon_font: linecons
 
@@ -74,3 +74,7 @@ images: images
 
 # Theme version
 version: 0.4.3
+
+# Miscellaneous
+google_analytics: UA-xxx
+favicon: /favicon.ico
```

デフォルトの画像ファイルなどを入れ替えます。

```
/workspace/blog/themes/next/source/favicon.ico
/workspace/blog/themes/next/source/images/profile.jpg
/workspace/blog/themes/next/source/profile.jpg 
```

設定ファイルの編集が終わったのでcommitします。

```bash
$ git add -A
$ git commit -m 'next config edit'
```

### フォントの変更

font-familyも好みで変更します。NexTテーマの場合CSSフレームワークは[Styl](https://github.com/tj/styl)です。これはテーマを作る作者によって様々です。


```css:workspace/blog/themes/next/source/css/_variables/base.styl
// Font families.
//$font-family-sans-serif   = "Avenir Next", Avenir, Tahoma, Vendana, sans-serif
//$font-family-serif        = "PT Serif", Cambria, Georgia, "Times New Roman", serif
//$font-family-monospace    = "PT Mono", Consolas, Monaco, Menlo, monospace
//$font-family-chinese      = "Microsoft Jhenghei", "Hiragino Sans GB", "Microsoft YaHei"
//$font-family-base         = Lato, $font-family-chinese, sans-serif
//$font-family-headings     = Cambria, Georgia, $font-family-chinese, "Times New Roman", serif
//$font-family-posts        = $font-family-base

$font-family-sans-serif = 'Open Sans','Helvetica Neue','Helvetica','Arial','ヒラギノ角ゴ Pro W3','Hiragino Kaku Gothic Pro','メイリオ', Meiryo,'ＭＳ Ｐゴシック','MS PGothic',sans-serif
$font-family-serif = Georgia, "Times New Roman", serif
$font-family-monospace = "Source Code Pro", Consolas, Monaco, Menlo, Consolas, monospace

$font-family-headings     = Lato, $font-family-sans-serif
$font-family-posts        = Lato, $font-family-sans-serif
$font-family-base         = $font-family-posts
```

これでHexoの設定は終了です。コミットしておきます。

```bash
$ git add -A
$ git commit -m 'next theme config'
$ cd workspace blog
```

## Markdownのpostファイルのコピー


記述したMarkdownはすべてGitHub上で管理しています。IDCFクラウド上のDockerホストにSSH接続してDocker Composeを起動しているディレクトリに移動します。docker-compose.ymlではCloud9のworkspaceをホストのディレクトリにマウントしています。

```yaml:docker-compose.yml
cloud9:
  build: .
  ports:
    - 8080:80
    - 15454:15454
    - 3000:3000
    - 5000:5000
  volumes:
    - ./workspace:/workspace
  command: node /cloud9/server.js --port 80 -w /workspace --auth xxx:xxx
```

blogインスタンスのsourceディレクトリは一度削除してまるごとコピーします。最後に作り直したblogインスタンスのディレクトリをGitHubの新しいリポジトリにpushして移設は終了です。

## GitHub Pagesにデプロイ

blogインスタンスディレクトリの_config.ymlに[デプロイ](http://hexo.io/docs/deployment.html)の設定をします。

### SSHキーの作成

GitHubの[SSHキー作成](https://help.github.com/articles/generating-ssh-keys/)ページの手順で自分のプロファイルにキーを追加します。SSHキーもCloud9のコンソールから作成します。

```bash
$ cd ~/.ssh
$ ssh-keygen -t rsa -b 4096 -C "ma6ato@gmail.com"
```

### デプロイ

```yaml:/workspace/blog/_config.yml
 # Deployment
 ## Docs: http://hexo.io/docs/deployment.html
 deploy:
  type: git
  repo: git@github.com:masato/masato.github.io.git
```

Cloud9のコンソールからhexoコマンドを使ってデプロイします。この例では最初にコンパイル先のディレクトリをきれいにします。`--generate`フラグでMarkdownのコンパイルを行いデプロイしています。

```bash
$ cd /workspace/blog
$ hexo clean && hexo deploy --generate
```

Nitrous.IOの無料版では`cd`コマンドでさえ重いくらいになっていましたが、Markdownのコンパイルもとても速くなりさくさく動くようになりました。Cloud IDEでブログを書いて[GitHub Pages](https://pages.github.com/)にデプロイする環境が自分クラウドにできました。こちらがCloud9の新環境でデプロイしたHexoの[ページ](https://masato.github.io/2015/06/17/cloud9-on-idcf-hexo-next/)です。こちらがCloud9の新環境でデプロイしたHexoの[ブログ](https://masato.github.io/2015/06/17/cloud9-on-idcf-hexo-next/)です。