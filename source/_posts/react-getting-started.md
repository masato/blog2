title: "React入門 - Part1: Getting Started"
date: 2014-12-26 12:32:00
tags:
 - React
 - ReactiveProgramming
 - JavaScript
 - Emacs
 - Cask
 - Om
description: ReactはFacebookがオープンソースで開発しているJavaScripライブラリです。ドキュメントサイトのGetting StartedとTutorialから勉強をはじめます。きっかけはClojureを勉強していて、OmというReactのClojureScriptインタフェースを紹介したInfoQの記事で興味を持ちました。Web UIのデータバインディングをリアクティブに書く方法を取り入れたいです。
---

[React](http://facebook.github.io/react)はFacebookがオープンソースで開発しているJavaScripライブラリです。ドキュメントサイトの[Getting Started](http://facebook.github.io/react/docs/getting-started.html)と[Tutorial](http://facebook.github.io/react/docs/tutorial.html)から勉強をはじめます。きっかけはClojureを勉強していて、[Om](https://github.com/swannodette/om)というReactのClojureScriptインタフェースを紹介した[InfoQの記事](http://www.infoq.com/jp/news/2014/02/om-react)で興味を持ちました。Web UIのデータバインディングをリアクティブに書く方法を取り入れたいです。

<!-- more -->

### Reactの特徴

[React](http://facebook.github.io/react)のページに書いてある特徴を簡単にまとめます。

* JUST THE UI
MVCのVだけComponentとして提供する

* VIRTUAL DOM
VirtualDOMを使い差分だけ更新するため高速に動作する

* DATA FLOW
一方向のリアクティブなデータバインディング

* JSX
JavaScript内にXML風に記述でき、DOM構造がわかりやすい

### 開発用コンテナ

JavaScriptの開発環境はStrongLoopを追加した[masato/baseimage](https://registry.hub.docker.com/u/masato/baseimage/)イメージを使います。`insecure_key`は[phusion/baseimage-docker](https://github.com/phusion/baseimage-docker)からダウンロードして使います。

``` bash
$ wget -P ~/.ssh https://raw.githubusercontent.com/phusion/baseimage-docker/master/image/insecure_key
$ chmod 600 ~/.ssh/insecure_key
$ docker run -d -it --name dev masato/baseimage /sbin/my_init bash
$ IP_ADDRESS=$(docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" dev)
$ ssh -A docker@$IP_ADDRESS -o IdentitiesOnly=yes -i ~/.ssh/insecure_key
```

### Node.jsの準備

SSHで接続後、nvmを有効にしてnodeとnpmコマンドが使えるようにします。

``` bash
$ source ~/.profile
$ nvm use v0.10
Now using node v0.10.35
$ npm --version
2.1.16
```

### Emacsの設定

JSX用のEmacsの設定を、[Emacs - Setup JSX mode, JSX Syntax checking and Suggestion](http://truongtx.me/2014/03/10/emacs-setup-jsx-mode-and-jsx-syntax-checking/)を参考にして、web-modeとflycheckをCaskファイルに追加します。

``` el:~/emacs.d/Cask
;; web-mode
(depends-on "web-mode")
(depends-on "flycheck")
;; javascript
(depends-on "js2-mode")
(depends-on "ac-js2")
```

npmコマンドでjsxhintパッケージをインストールします。

``` bash
$ npm install -g jsxhint
```

caskコマンドでパッケージをインストールします。

``` bash
$ cd ~/.emacs.d
$ cask install
```

JSX用のweb-modeとflucheckの設定を追加します。

``` el ~/.emacs.d/inits/10-jsx-flycheck.el
(add-to-list 'auto-mode-alist '("\\.jsx$" . web-mode))
(defadvice web-mode-highlight-part (around tweak-jsx activate)
  (if (equal web-mode-content-type "jsx")
      (let ((web-mode-enable-part-face nil))
        ad-do-it)
    ad-do-it))

(require 'flycheck)
(flycheck-define-checker jsxhint-checker
  "A JSX syntax and style checker based on JSXHint."

  :command ("jsxhint" source)
  :error-patterns
  ((error line-start (1+ nonl) ": line " line ", col " column ", " (message) line-end))
  :modes (web-mode))
(add-hook 'web-mode-hook
          (lambda ()
            (when (equal web-mode-content-type "jsx")
              ;; enable flycheck
              (flycheck-select-checker 'jsxhint-checker)
              (flycheck-mode))))
```

js2-modeの設定をします。JavaScriptファイルは2タブになるようにします。

``` el ~/.emacs.d/inits/11-js2-mode.el
(autoload 'js2-mode "js2-mode" nil t)
(add-to-list 'auto-mode-alist '("\\.js$" . js2-mode))
(add-hook 'js2-mode-hook
          '(lambda ()
             (setq js2-basic-offset 2)))

(add-hook 'js-mode-hook 'js2-minor-mode)
(add-hook 'js2-mode-hook 'ac-js2-mode)
```

### Hello World

ライブラリはCDNから取得してかんたんに`Hello World`をはじめます。最初にプロジェクトを作成します。

``` bash
$ mkdir ~/helloworld
$ cd !$
```

index.htmlを作成します。JSXをオンラインで変換するためにJSXTransformer-0.12.2.jsをロードします。JSXを記述したJavaScriptファイルは、`text/jsx`を指定してロードすると自動的に変換してくれます。

``` html ~/helloworld/index.html
<!DOCTYPE html>
<html>
  <head>
    <script src="http://fb.me/react-0.12.2.js"></script>
    <script src="http://fb.me/JSXTransformer-0.12.2.js"></script>
    <script type="text/jsx" src="helloworld.jsx"></script>
  </head>
  <body>
    <div id="example"></div>
  </body>
</html>
```

`<h1>`要素のところがJSXです。今回はComponentを作成していないので、Top Levelコンポーネントに入ります。

``` js ~/helloworld/helloworld.jsx
React.render(
  <h1>Hello, world!</h1>,
  document.getElementById('example')
);
```

### Webサーバーを起動して確認

プロジェクトのディレクトリに移動して、PythonのSimpleHTTPServerを起動します。

``` bash
$ cd ~/helloworld
$ python -m SimpleHTTPServer 8080
Serving HTTP on 0.0.0.0 port 8080 ...
```

IDCFクラウド上の仮装マシンにDocker開発環境があります。外部からコンテナに接続するために、ngrokを使ってトンネルをつくります。

``` bash
$ docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" dev
172.17.1.157
$ docker run -it --rm wizardapps/ngrok:latest ngrok 172.17.1.157:8080
```

ngrokで取得したURLをブラウザで確認すると`Hello World!`が表示されました。56110a8cはランダムで変わります。

http://56110a8c.ngrok.com/

### ChromeのReact Developer Tools 

Chromeの場合、[React Developer Tools](http://fb.me/react-devtools)をインストールすると、Reactのデバッグが便利になります。デベロッパーツールに`React`というタブが追加され、以下のようにComponentsを確認することができます。


{% img center /2014/12/26/react-getting-started/hello-world.png %}

