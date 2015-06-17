title: "Dockerで開発環境をつくる - F#のインストール"
date: 2014-12-29 00:53:46
tags:
 - Dockerfile
 - DockerDevEnv
 - F#
 - Mono
 - Ubuntu
 - Xamarin
 - OCaml
 - Haskell
description: 先月.NETのクロスプラットフォーム化が発表されました。そのうちMacとLinuxに移植されるそうですが、これにはXamarinとMonoが大きな役割を担っているようです。Xamarinの名前も最近よく聞くようになりました。XamarinではC#でiOSとAndroidのクロスプラットフォームの開発ができますが、F#でもアプリを書けるようです。F#はOCamlやHaskellの影響を受けている関数型言語です。オブジェクト指向的な書き方もできるみたいで、以前から興味があったのですが、良い機会なのでDocker開発環境をつくって勉強してみようと思います。
---

先月.NETのクロスプラットフォーム化が[発表されました](http://jp.techcrunch.com/2014/11/13/microsoft-takes-net-open-source-and-cross-platform/)。そのうちMacとLinuxに移植されるそうですが、これにはXamarinとMonoが大きな役割を担っているようです。[Xamarin](http://xamarin.com/)の名前も最近よく聞くようになりました。XamarinではC#でiOSとAndroidのクロスプラットフォームの開発ができますが、F#でもアプリを書けるようです。F#はOCamlやHaskellの影響を受けている関数型言語です。オブジェクト指向的な書き方もできるみたいで、以前から興味があったのですが、良い機会なのでDocker開発環境をつくって勉強してみようと思います。

<!-- more -->

### Dockerfile

[masato/docker-fsharp](https://registry.hub.docker.com/u/masato/docker-fsharp/)はいつもの様に[phusion/baseimage](https://index.docker.io/u/phusion/baseimage/)を使ってF#の開発環境のイメージをビルドしました。GitHubのリポジトリは[masato/docker-fsharp](https://github.com/masato/docker-fsharp)です。


``` bash ~/docker_apps/docker-fsharp/Dockerfile
ROM phusion/baseimage:0.9.15
MAINTAINER Masato Shimizu <ma6ato@gmail.com>
# Set correct environment variables.
ENV HOME /root
# Regenerate SSH host keys.
RUN /etc/my_init.d/00_regen_ssh_host_keys.sh
## Enabling the insecure key permanently
RUN /usr/sbin/enable_insecure_key
## Japanese Environment
ENV LANG ja_JP.UTF-8
RUN locale-gen $LANG && update-locale $LANG && \
    ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive \
    apt-get install -y python emacs24-nox emacs24-el git byobu wget curl unzip tree elinks
## Mono/F#
RUN apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF
RUN echo "deb http://download.mono-project.com/repo/debian wheezy main" > /etc/apt/sources.list.d/mono-xamarin.list && \
    apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y mono-devel fsharp mono-vbnc nuget
RUN mozroots --machine --import --sync --quiet
## Development Environment
RUN update-alternatives --set editor /usr/bin/vim.basic
ENV USERNAME docker
ENV HOME /home/${USERNAME}
# Add Un-Root User
RUN useradd -m -d ${HOME} -s /bin/bash ${USERNAME} && \
    echo "${USERNAME}:${USERNAME}" | chpasswd && \
    mkdir ${HOME}/.ssh ${HOME}/tmp && \
    chmod 700 ${HOME}/.ssh && \
    cp /root/.ssh/authorized_keys ${HOME}/.ssh && \
    chmod 600 ${HOME}/.ssh/authorized_keys && \
    chown -R ${USERNAME}:${USERNAME} ${HOME}/.ssh && \
    echo "export LANG=ja_JP.UTF-8" >> ${HOME}/.profile && \
    echo "docker ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
ADD dotfiles ${HOME}
RUN chown -R ${USERNAME}:${USERNAME} ${HOME}
## Cask
RUN curl -fsSkL https://raw.github.com/cask/cask/master/go | python && \
    echo export PATH='${HOME}/.cask/bin:$PATH' >> ${HOME}/.profile && \
    /bin/bash -c 'source ${HOME}/.profile && cd ${HOME}/.emacs.d && cask install'
USER root
ENV HOME /root
WORKDIR /root
CMD ["/sbin/my_init"]
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

### Emacs

[Cask](https://github.com/cask/cask)ファイルに[fsharp-mode](https://github.com/emacsmirror/fsharp-mode/tree/master/emacs)を追加します。

``` el ~/docker_apps/docker-fsharp/dotfiles/.emacs.d/Cask
...
;; fsharp
(depends-on "fsharp-mode")
...
```

initsディレクトリにfsharp-modeの設定を書きます。

``` el ~/docker_apps/docker-fsharp/dotfiles/.emacs.d/inits/12-fsharp-mode.el
(require 'fsharp-mode)
```

### Dockerコンテナの起動

使い捨てのコンテナを起動します。dockerユーザーにスイッチして作業ディレクトリを作成します。

``` bash
$ docker run -it --rm masato/docker-fsharp /sbin/my_init bash
$ su - docker
$ mkdir fsharp_apps
$ cd !$
```

### バージョンの確認

monoのバージョンは3.10.0です。

``` bash
$ mono --version
Mono JIT compiler version 3.10.0 (tarball Wed Nov  5 12:50:04 UTC 2014)
Copyright (C) 2002-2014 Novell, Inc, Xamarin Inc and Contributors. www.mono-project.com
        TLS:           __thread
        SIGSEGV:       altstack
        Notifications: epoll
        Architecture:  amd64
        Disabled:      none
        Misc:          softdebug
        LLVM:          supported, not enabled.
        GC:            sgen
```

F#のバージョンは3.1です。 

``` bash
$ fsharpi

F# Interactive for F# 3.1 (Open Source Edition)
Freely distributed under the Apache 2.0 Open Source License

For help type #help;;

>
```

### コンパイルして実行する

`Hello World`を作成します。

``` fs ~/fsharp_apps/hello_world.fs
[<EntryPoint>]
let main argv =
  printfn "ハロー、ワールド"
  0
```

コンパイルします。

``` bash
$ fsharpc hello_world.fs
F# Compiler for F# 3.1 (Open Source Edition)
Freely distributed under the Apache 2.0 Open Source License
```

コンパイルされたexeファイルには実行権限がついていますが、そのまま実行するとエラーになります。

``` bash
$ ./hello_world.exe
-su: ./hello_world.exe: cannot execute binary file: Exec format error
```

exeファイルを実行する場合は、monoコマンドを使います。

``` bash
$ mono hello_world.exe
ハロー、ワールド
```

### インタプリタを実行する

fsharpiコマンドでインタプリタを開始します。 

``` bash
$ fsharpi

F# Interactive for F# 3.1 (Open Source Edition)
Freely distributed under the Apache 2.0 Open Source License

For help type #help;;

> 
```

リテラルを入力した後は、セミコロンを2つ入力してEnterキーで実行します。

``` bash
> "hello world";;
val it : string = "hello world"
> 1 + 2;;
val it : int = 3
```

`#help;;`で表示されるように、インタプリタを終了する場合は`#quit;;`を実行します。

``` bash
> #help;;

  F# Interactive directives:

    #r "file.dll";;        Reference (dynamically load) the given DLL
    #I "path";;            Add the given search path for referenced DLLs
    #load "file.fs" ...;;  Load the given file(s) as if compiled and referenced
    #time ["on"|"off"];;   Toggle timing on/off
    #help;;                Display help
    #quit;;                Exit

  F# Interactive command line options:

      See 'fsharpi --help' for options


> #quit;;
- Exit...
```

