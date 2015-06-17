title: 'DashingとTreasure Data - Part3: Rubyのtd-clientを使う'
date: 2014-05-31 10:52:41
tags:
 - Dashing
 - AnalyticSandbox
 - TreasureData
 - DockerEnv
 - Ruby
 - byobu
description: 昨日はTreasure Data Serviceへtdコマンドから簡単なクエリを実行しました。DashingはSinatraアプリなのでRubyでTreasure Data Serviceにクエリするプログラムを書きます。ちょっとしたDBアクセス用のライブラリも用意してみました。
---

[Part2](/2014/05/30/dashing-treasuredata-td/)は`Treasure Data Service`へtdコマンドから簡単なクエリを実行しました。
[Dashing](https://github.com/Shopify/dashing)はSinatraアプリなのでRubyで`Treasure Data Service`にクエリするプログラムを書きます。
ちょっとしたDBアクセス用のライブラリも用意してみました。

<!-- more -->

## Dockerコンテナ起動

-eオプションで環境変数を指定してコンテナを起動します。TD_API_KEYには、`td apikey:show`で確認した値を使います。

``` bash
$ ~/docker_apps/phusion/
$ docker run -t -i -p 3030:3030 -e TD_API_KEY=xxx -v ~/docker_apps/workspaces/dashing:/root/sinatra_apps masato/baseimage:1.0 /sbin/my_init /bin/bash
```

## 環境変数の確認
コンテナに環境変数がセットされていることを確認します。
SSH接続すると`-eオプション`で指定した環境変数が反映されないので、コンテナを起動した`/bin/bash`のまま作業します。

``` bash
root@67413c80299f:/# echo $TD_API_KEY
xxx
```

## byobu

サーバーアプリを開発する場合、Emacsのmulti-termだけを使うよりも、
マルチプレクサでEmacsとサーバーで別ウィンドウで作業すると便利なのでbyobuを起動します。
``` bash
$ byobu
$ byobu-config
$ byobu-select-backend 
$ byobu-ctrl-a
```

## Gemfile

このコンテナでも`-vオプション`を使い、Dockerホストの共有ディレクトリをマウントして作業します。
プロジェクトのディレクトリに移動して確認します。

``` bash
$ cd /root/sinatra_apps/cloud_td
$ ls
Gemfile       README.md  config.ru   history.yml  lib     sql      widgets
Gemfile.lock  assets     dashboards  jobs         public  test.rb
```

Gemfileに`td-client`を追加します。

``` ruby /root/sinatra_apps/cloud_td/Gemfile
gem 'td-client'
```

`bundle install`します。

``` bash
$ bundle install
```

## Dashingの起動確認

とりあえずDashingサーバーを起動して確認します。
``` bash
$ dashing server
```

デフォルトで同梱されているsampleダッシュボードの表示を確認します。
`-vオプション`と`-eオプション`を使って、少しずつ`Disposable Infrastructure`らしくなってきました。

http://{Dockerホスト}:3030/sample

## lib/db_client.rb 

`Treasure Data`に接続するコードを、`lib`ディレクトリに書きました。
API_KEYは、`docker run`の環境変数のオプションに渡している値を使います。一応initializeでオーバーライドできるようにしています。

``` ruby /root/sinatra_apps/cloud_td/lib/db_client.rb
# -*- coding: utf-8 -*-

require 'td-client'

module DBClient
  class TD
    def initialize(apikey: nil)
      apikey = ENV['TD_API_KEY'] unless apikey
      @client = TreasureData::Client.new apikey
    end
    attr_reader :headers

    def wait_finished db_name,sql,type, &block
      job = @client.query(db_name, sql,
                          result_url=nil, priority=nil, retry_limit=nil,
                          opts={type: type})

      puts "waiting until job_id:#{job.job_id} finished... "
      until job.finished?
        sleep 2
        job.update_progress!
      end
      job.update_status!
      puts "job_id:#{job.job_id} finished, status:#{job.status} "

      block.call(job)
    end

    def query(db_name, sql, type: nil)
      type ||= :hive
      results = []
      wait_finished(db_name,sql,type) do |job|
        job.result_each do |row|
          results << {label: row[0], value: row[1]}
        end
        @headers = results.map{|m| m[:label]} unless @headers
      end
      results
    end
  end
end
```

## job

[前回](/2014/05/30/dashing-treasuredata-td/)に`Treasure Data Toolbelt`で使ったのと同じSQLをヒアドキュメントに書きます。さきほど書いたlibディレクトリ下のコードをrequre_relativeします。

確認用に、取得したデータは、sampleダッシュボードの右側にある`data-id="buzzwords"`のウィジェットを使って表
示してみます。`data-view="List"`で`widgets/list`ウィジェットを使っているので、リスト表示のサンプルにちょうど良いです。

`send_event`の引数で`data-id`にIDとして`list`を使います。
`SCHEDULER.every '1d'`は、[rufus-scheduler](https://github.com/jmettraux/rufus-scheduler/)を使って、1日1回ジョブを実行する定義です。
`:first_in => 0`とすることで、初回のジョブがDashing起動後すぐに実行されるようにします。

``` ruby /root/sinatra_apps/cloud_td/jobs/access_count.rb

require_relative "../lib/db_client"

client = DBClient::TD.new

SCHEDULER.every '1d',:first_in => 0 do |job|
  db_name = 'clouddb'
  sql = <<-EOS
    SELECT TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST') AS day,
           COUNT(1) AS cnt
    FROM www_access
    GROUP BY TD_TIME_FORMAT(time, 'yyyy-MM-dd', 'JST')
    ORDER BY day DESC
  EOS

  results = client.query(db_name,sql)
  send_event('buzzwords', { items: results })
end
```

sampleダッシュボードのbuzzwordsのdata-idで、ウィジェットを定義している箇所を確認します。
``` ruby /root/sinatra_apps/cloud_td/dashboards/sample.rb
    <li data-row="1" data-col="1" data-sizex="1" data-sizey="2">
      <div data-id="buzzwords" data-view="List" data-unordered="true" data-title="STATUS COUNT" dat\
a-moreinfo="# of times said around the office"></div>
    </li>
```

## 確認

デフォルトのsampleダッシュボードの右側に、`Treasure Data`から取得したデータがリスト形式で表示されることを確認します。
http://{Dockerホスト}:3030/sample

## git push

Dockerホストから別のシェルを起動して、SSH接続するためにIPアドレスを確認します。

``` bash
$ docker inspect 67413c80299f | jq -r '.[0] | .NetworkSettings | .IPAddress'
172.17.0.2
```

Dockerコンテナには鍵をコピーしていないので、
Gitのリモートリポジトリに使っている鍵を、ssh-agentに追加して、コンテナにSSHで接続します。
Dockerホストと同じVLAN上に配置している、リモートリポジトリにコンテナ内部からpushします。

``` bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/private_key
$ ssh -A root@172.17.0.2
$ cd ~/sinatra_apps/cloud_td/
$ git add -A
$ git commit -m 'add td_client library'
$ git push origin master
```

## まとめ
今回はデフォルトのsampleダッシュボードを使って、`Treasure Data Service`からデータを取得して表示してみました。
簡単なDB接続用コードを書いて、複数のジョブからrequireして簡単にSQLを実行できるようにしました。

sample.rbを参考にしながら、ウィジェットを定義する自分のダッシュボードを作っていくと簡単にカスタマイズできます。

次回は、[fluentd](https://github.com/fluent/fluentd/)の安定版である[td-agent](http://docs.treasuredata.com/articles/td-agent)を使って、ApacheやTomcatのログをストリームでインポートしてみます。


