title: "OpenRestyからRedisに問い合わせるAPIのLuaのサンプル"
date: 2015-05-18 11:41:07
tags:
 - OpenResty
 - Docker
 - DockerCompose
 - Nginx
 - Lua
---

Nginxを拡張してLuaが便利なOpenRestyを使って簡単なAPIのサンプルを書いてみます。バックエンドのRedisに問い合わせてJSON形式でレスポンスを返します。LuaのRedisクライアントは[lua-resty-redis](https://github.com/openresty/lua-resty-redis)を使っているようです。Luaで組み込みのプログラムが書けるので、リバースプロキシにも使うOpenRestyのところで、バックエンドのアプリのちょっとしたAPIの差異を吸収することができます。

<!-- more -->

## プロジェクト

今回もサービスのオーケストレーションにはDocker Composeを使います。プロジェクトのディレクトリを作成してdocker-compose.ymlとnginx.confを作成します。

``` bash
$ cd ~/openresty_apps
$ tree
.
├── docker-compose.yml
└── nginx
    └── nginx.conf
```

OpenRestyのイメージは[tenstartups/openresty](https://registry.hub.docker.com/u/tenstartups/openresty/)を使います。このイメージでは`/entrypoint`という起動スクリプトがENTRYPOINTに設定されています。nginx.confを編集したあとにHUPを送って再読み込みしたいので、docker-compose.ymlで上書きます。

``` yaml ~/openresty_apps/docker-compose-yml
openresty:
  restart: always
  image: tenstartups/openresty
  ports:
    - "80:80"
  volumes:
    - ./nginx:/etc/nginx
  links:
    - redis
  entrypoint: ["nginx", "-c", "/etc/nginx/nginx.conf"]
redis:
  restart: always
  image: redis
rediscli:
  image: redis
  links:
    - redis
```

nginx.confで環境変数を読み込むために`env`ディレクティブを使い、`links`で指定されたRedisnサービスのIPアドレスとポートを設定します。クエリ文字列に`uuid`を指定します。OpenRestyからRedisに問い合わせて対応するtokenを取得してJSONを返す簡単なサンプルです。

``` bash ~/openresty_apps/nginx/nginx.conf
daemon off;
worker_processes  1;

env REDIS_PORT_6379_TCP_ADDR;
env REDIS_PORT_6379_TCP_PORT;

events {
    worker_connections  256;
}

http {
    server {
        listen 80;
        access_log /proc/self/fd/1;
        error_log /proc/self/fd/2;

        location /api/v1/tokens {
            default_type text/html;
            content_by_lua '
                local cjson = require "cjson"
                local redis = require "resty.redis"
                local red = redis:new()
                local args = ngx.req.get_uri_args()

                local ok, err = red:connect(os.getenv("REDIS_PORT_6379_TCP_ADDR"), tonumber(os.getenv("REDIS_PORT_6379_TCP_PORT")))

                if not ok then
                    ngx.say("failed to connect: ", err)
                    return
                end

                local res, err = red:get("users:" .. args.uuid)

                if not res then
                    ngx.say("failed to get token: ", err)
                    return
                end

                ngx.header.content_type = "application/json; charset=utf-8"
                ngx.say(cjson.encode({token = res}))
                red:close()
            ';
        }
    }
}
```

Docker Composeから`openresty`サービスをupします。

``` bash
$ docker-compose up openresty
docker-compose up openresty
Recreating openrestyapps_redis_1...
Recreating openrestyapps_openresty_1...
Attaching to openrestyapps_openresty_1
```

依存関係にある`redis`サービスも起動しますが`rediscli`サービスは起動しません。

``` bash
$ docker-compose ps
      Name             Command             State              Ports
-------------------------------------------------------------------------
openrestyapps_op   nginx -c /etc/ng   Up                 0.0.0.0:80->80/t
enresty_1          inx/nginx.conf                        cp
openrestyapps_re   /entrypoint.sh     Up                 6379/tcp
dis_1              redis-server
```

## redis-cliでレコード作成

`rediscli`サービスでテスト用のレコードをRedisに作成します。

``` bash
$ docker-compose run --rm rediscli bash -c 'redis-cli -h $REDIS_PORT_6379_TCP_ADDR set users:123 8Nyo7D9a'
OK
Removing openrestyapps_rediscli_run_1...
```

getからレコードの作成を確認します。

``` bash
$ docker-compose run --rm rediscli bash -c 'redis-cli -h $REDIS_PORT_6379_TCP_ADDR get users:123'
"8Nyo7D9a"
Removing openrestyapps_rediscli_run_1...
```

## curlでテスト

`openresty`サービスにHUPを送ってnginx.confを再読込します。

``` bash
$ docker-compose kill -s HUP openresty
Killing openrestyapps_openresty_1...
```

Dockerホストからcurlを使いJSON形式でtokenが取得できました。

``` bash
$ curl  "http://localhost/api/v1/tokens?uuid=123"
{"token":"8Nyo7D9a"}
```


