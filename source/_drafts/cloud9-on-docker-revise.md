title: "Cloud9 on Docker - Part4: Node.jsとHexoのバージョンアップ"
date: 2017-01-10 20:05:32
tags:
 - Cloud9
 - Hexo
 - Node.js
description: 1年以上アップデートせずに放置していたCloud9の環境を再構築して使い直します。
---

　1年以上アップデートせずに放置していたCloud9の環境を再構築して使い直します。あわせてDockerfileも見直してシンプルにしました。


<!-- more -->

## プロジェクト

　リポジトリは[こちら](https://github.com/masato/docker-cloud9)です。

### Dockerfile

　Dockerfileは[公式のNode.js](https://hub.docker.com/_/node/)の7を使います。

```Dockerfile
FROM node:7

RUN git clone https://github.com/c9/core.git /cloud9 && \
    cd /cloud9 && ./scripts/install-sdk.sh

RUN npm install hexo-cli -g

WORKDIR /workspace
```


### docker-compose.yml

　docker-compose.ymlファイルもversion 2のフォーマットに更新します。`cloud-user:secret`はCloud9にログインするときのBasic認証になります。`ユーザー名:パスワード`になるので任意に設定します。

```docker-compose.yml
version: '2'
services:
  app:
    build: .
    restart: always
    ports:
      - 80:80
      - 4000:4000
    volumes:
      - ./workspace:/workspace
      - ~/.ssh/id_rsa:/root/.ssh/id_rsa:ro
      - ~/.gitconfig:/root/.gitconfig
      - /etc/localtime:/etc/localtime:ro
    command: node /cloud9/server.js --port 80 -w /workspace -l 0.0.0.0 --auth cloud-user:secret
```

