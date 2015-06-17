title: "Packerを使いWindows上でOVAを作成する - Part11: PackerのOVAをIDCFクラウドで動かす"
date: 2014-06-22 11:31:09
tags:
 - PackerOVA
 - Packer
 - VMwarePlayer
 - idcf-compute-api
 - jq
 - IDCFクラウド
 - IDCFオブジェクトストレージ
description: PackerでOVA作成シリーズです。Part10でようやくOVAが作成できたので、IDCFオブジェクトストレージにPUTしたあと、IDCFクラウドにデプロイして動作確認します。その後IDCFクラウドからエクスポートしたOVAと同じ方法で、IDCFオブジェクトストレージからダウンロード、ローカルのVMware Player 6.02にもう一度OVAをデプロイしてみます。

---
[PackerでOVA作成](http://masato.github.io/tags/PackerOVA/)シリーズです。
[Part10](/2014/06/19/packer-windows-nested-vmware-player-create-ova/)でようやくOVAが作成できたので、IDCFオブジェクトストレージにPUTしたあと、IDCFクラウドにデプロイして動作確認します。

その後[IDCFクラウドからエクスポートしたOVA](/2014/06/21/idcf-storage-knox-stream-upload/)と同じ方法で、IDCFオブジェクトストレージからダウンロード、ローカルの`VMware Player 6.02`にもう一度OVAをデプロイしてみます。


いろいろと苦労しましたが、ようやく[PackerでOVA作成](http://masato.github.io/tags/PackerOVA/)のシリーズが終わりました。備忘のためリンクです。

* [Packerを使いWindows上でOVAを作成する - Part1: VMware Workstation ビルド環境](/2014/06/08/packer-windows-vmware-iso-ova-buildenv/)
* [Packerを使いWindows上でOVAを作成する - Part2: VMware Workstation ビルド実行](/2014/06/08/packer-windows-vmware-iso-ova-build/)
* [Packerを使いWindows上でOVAを作成する - Part3: IDCFクラウド](/2014/06/09/packer-idcf-ova/)
* [Packerを使いWindows上でOVAを作成する - Part4: Nested ESXi5 on VirtualBox ESXi5 Vagrant](/2014/06/11/packer-windows-vagrant-mingw-mintty/)
* [Packerを使いWindows上でOVAを作成する - Part5: Nested ESXi5 on VirtualBox インストール](/2014/06/12/packer-windows-vagrant-nested-esxi5/)
* [Packerを使いWindows上でOVAを作成する - Part6: Nested ESXi5 on VirtualBox リモートビルド](/2014/06/13/packer-windows-vagrant-remote-esxi5-build/)
* [Packerを使いWindows上でOVAを作成する - Part7: Nested ESXi5 on VirtualBox パーティション作成](/2014/06/14/packer-windows-vagrant-remote-esxi5-partedUtil/)
* [Packerを使いWindows上でOVAを作成する - Part8: Nested VirtualBox デスクトップ環境のディスク拡張](/2014/06/15/packer-windows-vagrant-nested-vmware-player/)
* [Packerを使いWindows上でOVAを作成する - Part9: VMwarePlayerでNested VT-x](/2014/06/16/packer-windows-nested-vmware-palyer/)
* [Packerを使いWindows上でOVAを作成する - Part10: Nested VT-x VMwarePlayerでOVA作成](/2014/06/19/packer-windows-nested-vmware-player-create-ova/)
* [Packerを使いWindows上でOVAを作成する - Part11: PackerのOVAをIDCFクラウドで動かす](/2014/06/22/packer-windows-idcf-cloud/)

<!-- more -->

### OVAをESXi4用に再パッケージして準備する

Packerで作成したOVAをIDCFクラウドで動作させるため、再パッケージします。
* マニフェストファイルを除外
* OVFファイルのVirtualSystemTypeを`vmx-07`を確認する

OVFのVirtualSystemTypeは指定したように`vmx-07`になっています。

``` xml packer_vmware-iso_vmware.ovf
        <vssd:VirtualSystemType>vmx-07</vssd:VirtualSystemType>
```

tmpディレクトリを作成、マニフェストを除いてOVAを作り直します。
``` bash
$ mkdir -p ~/packer_apps/tmp
$ cd ~/packer_apps
$ mv packer_vmware-iso_vmware-disk1.vmdk tmp
$ mv packer_vmware-iso_vmware-disk1.ovf tmp
$ cd tmp
$ tar cvf ../packer_vmware-iso_vmware_07_nomf.ova packer_vmware*
```

### OVAをIDCFオブジェクトストレージへPUT

s3cmdを使い、IDCFオブジェクトストレージへOVAをPUTします。
``` bash
$ cd ~/packer_apps
$ s3cmd put -c ~/.s3cfg.idcf --acl-public packer_vmware-iso_vmware_07_nomf.ova s3://ova/
```

### OVAからインスタンスを作成

IDCFオブジェクトストレージへPUTしたOVAからテンプレートを作成します。

まず、registerTemplateコマンドで使うテンプレートの`OS Types`を確認します。
今回は`Othre Ubuntu (64-bit)`の100を使います。

``` bash
$ idcf-compute-api listOsTypes -c=id,description | egrep "Ubuntu.*64-bit"
100,"Other Ubuntu (64-bit)"
126,"Ubuntu 10.04 (64-bit)"
130,"Ubuntu 8.04 (64-bit)"
129,"Ubuntu 8.10 (64-bit)"
128,"Ubuntu 9.04 (64-bit)"
127,"Ubuntu 9.10 (64-bit)"
```

registerTemplateコマンドを使い、IDCFオブジェクトストレージのURLからテンプレートを登録します。
あとでOVAをエクスポートするので、`isextractable=true`にします。
``` bash
$ idcf-compute-api registerTemplate \
    --name=packer-07-nomf-v2  \
    --displaytext=packer-07-nomf-v2 \
    --format=OVA \
    --hypervisor=VMware \
    --ostypeid=100 \
    --url=http://ova.ds.jp-east.idcfcloud.com/packer_vmware-iso_vmware_07_nomf.ova \
    --zoneid=1 \
    --isextractable=true \
    --passwordenabled=true  | jq '.registertemplateresponse.template | .[0] | {id,name}'
{
  "name": "packer-07-nomf-v2",
  "id": 8390
}
```


インスタンスタイプのidを確認します。今回はM4を使うので、idは21です。
``` bash
$ idcf-compute-api listServiceOfferings -c=id,name,displaytext | grep M4
21,"M4","M4 ( 2CPU / 4GB )"
```

waitjobのヘルパースクリプトを用意します。

``` bash ~/bin/waitjob
#!/usr/bin/env bash
while :
do
  json=$(idcf-compute-api queryAsyncJobResult --jobid=$1)
  status=$(echo ${json} | jq '.queryasyncjobresultresponse.jobstatus')

  if [ ${status} -eq 0 ]; then
    echo -ne "."
    sleep 10s
  else
    echo -e "\n"
    echo ${json} | jq ".queryasyncjobresultresponse | {jobid: $1,jobresult}"
    break;
  fi
done
```

registerTemplateコマンドで登録したtemplateidを指定してインスタンスを作成します。
``` bash
$ waitjob $(idcf-compute-api deployVirtualMachine \
  --serviceofferingid=21 \
  --templateid=8390 \
  --zoneid=1 \
  --name=nomf-07-v2 \
  --displayname=nomf-07-v2 \
  --keypair=google_compute_api \
  --group=mshimizu | jq '.deployvirtualmachineresponse | .jobid')
```

作成したインスタンスにSSHで接続して確認します。
* login: packer
* password: packer

``` bash
$ ssh packer@nomf-07-v2
packer@ubuntu-1404:~$
```

### OVAをダウンロードする

registerTemplateコマンドで登録したOVAをローカルの`VMware Player`にデプロイして確認するためダウンロードします。

[昨日作成した](/2014/06/21/idcf-storage-knox-stream-upload/)のシェルスクリプトを使います。

``` bash ~/idcf_apps/extract_template.sh
#!/usr/bin/env bash

if [ -z "$1" ] || [ -z "$2" ]; then
  echo "Usage: extract_template template_id object_name"
  exit -1
fi

extract_template() {
  waitjob $(idcf-compute-api extractTemplate --id=$1 \
              --mode=HTTP_DOWNLOAD --zoneid=1\
             | jq '.extracttemplateresponse.jobid')
}

extract_template_status() {
  templateid=$(echo $1 | jq 'if (.jobresult | has("template")) then .jobresult.template.id else false end')
  idcf-compute-api listTemplates  \
               --templatefilter=self --id="${templateid}" \
             | jq '.listtemplatesresponse.template[0].isready'
}

extract_url() {
  echo $1 | jq 'if (.jobresult | has("template")) then .jobresult.template.url else .jobresult end'  \
            | sed -e 's/%2F/\//g' \
            | sed -e 's/"//g'
}

json=`extract_template $1`

while :
do
  template_isready=`extract_template_status "${json}"`
  if [ ${template_isready} == true ]; then
    echo -e "\n"
    url=`extract_url "${json}"`
    if [[ "${url}" == http* ]]; then
      node ./upload_stream.js ${url} $2
    else
      echo "extract template failed: " ${url}
    fi
    break;
  else
    echo -ne "."
    sleep 10s
  fi
done
```

同様にknox-mpuを使った、IDCFオブジェクトストレージにマルチアップロードするためのNodo.jsのスクリプトファイルです。
``` node.js ~/idcf_apps/upload_stream.js

#!/usr/bin/env node

process.env['NODE_TLS_REJECT_UNAUTHORIZED'] = '0';

var url = process.argv[2]
  , fileName = process.argv[3];

var knox = require('knox')
  , MultiPartUpload = require('knox-mpu')
  , https = require('https')
  , util = require('util')
  , request = require('request');

var bucketName = 's3-streaming-upload'
  , endpointName = 'ds.jp-east.idcfcloud.com';

var uploadUrl = util.format('https://%s.%s/%s',
                             bucketName,endpointName,fileName);

console.log('extract ova from: ' + url, '\nto IDCF Obeject Storage: '+ uploadUrl);

var client = knox.createClient({
  key: '{ACCESS_KEY}'
, secret: '{SECRET_KEY}'
, bucket: bucketName
, endpoint: endpointName
});

var stream = request(url);
upload = new MultiPartUpload(
  {
    client: client
  , objectName: fileName
  , stream: stream
  },function(err,body){
      console.log(body);
  }
);
```

registerTemplateコマンドで登録したtemplateidを使い、OVAをIDCFオブジェクトストレージにPUTします。
第2引数はPUT先のオブジェクト名です。

``` bash
$ time ./extract_template.sh 8390 packer-v07-v2.ova
extract ova from: https://xxx.realhostip.com/userdata/xxx.ova ,
to IDCF Obeject Storage: https://s3-streaming-upload.ds.jp-east.idcfcloud.com/packer-v07-v2.ova
{ Location: 'http://s3-streaming-upload.s3.amazonaws.com/packer-v07-v2.ova',
  Bucket: 's3-streaming-upload',
  Key: 'packer-v07-v2.ova',
  ETag: '7OX4lu3aR4uc9-S5Q7G8uA==',
  size: 597514240 }

real    0m44.926s
user    0m14.265s
sys     0m3.597s
```

ACLをpublicに設定してダウンロードできるようにします。

``` bash
$ s3cmd -c ~/.s3cfg.idcf setacl --acl-public s3://s3-streaming-upload/packer-v07-v2.ova
s3://s3-streaming-upload/packer-v07-v2.ova: ACL set to Public  [1 of 1]
```

### ダウンロードしたOVAをVMXに変換できない

以下のURLからダウンロードできるようになります。
https://s3-streaming-upload.ds.jp-east.idcfcloud.com/packer-v07-v2.ova

Minttyを起動して、OVAをwgetします。ここでも証明書の検証ができません。。。
```
$ mkdir exp_tmp
$ wget --no-check-certificate https://s3-streaming-upload.ds.jp-east.idcfcloud.com/packer-v07-v2.ova -P ./exp_tmp
```

`VMware Player`にOVAをインポートするためovftoolでVMXに変換したいのですが、失敗します。
``` bash
$ cd ./exp_tmp
$ ovftool.exe packer-v07-v2.ova packer-v07-v2.vmx
Error: Did not find an .ovf file at the beginning of the OVA package.
Completed with errors
```

どうもIDCFクラウドにデプロイするためにマニフェストファイルを削除してOVAを作成したのが問題のようです。

### OVFの署名に失敗する

ダウンロードしたOVAをtarで展開します。
``` bash
$ cd exp_tmp
$ tar xvf packer-v07-v2.ova
packer_vmware-iso_vmware-disk1.vmdk
packer_vmware-iso_vmware.ovf
```

VirtualSystemTypeのバージョンを`vmx-09`に編集します。
``` bash
$ vim  packer_vmware-iso_vmware.ovf
        <vssd:VirtualSystemIdentifier>packer-vmware-iso</vssd:VirtualSystemIdentifier>
<!--
        <vssd:VirtualSystemType>vmx-07</vssd:VirtualSystemType>
-->
        <vssd:VirtualSystemType>vmx-09</vssd:VirtualSystemType>
      </System>
```

マニフェストを作成するため、自己署名の証明書を用意します。
``` bash
$ openssl.exe genrsa 2048 > server.key
$ openssl.exe req -new -key server.key > server.csr
$ openssl.exe x509 -days 3650 -req -signkey server.key < server.csr > server.crt
```

VMDKとOVFからマニフェストを作成します。
``` bash
$ openssl.exe sha1 *.vmdk *.ovf > packer_vmware-iso_vmware.mf
$ cat packer_vmware-iso_vmware.mf
SHA1(packer_vmware-iso_vmware-disk1.vmdk)= 99b3d41e577ff876c5f4d82c906ad3c932dfc9cb
SHA1(packer_vmware-iso_vmware.ovf)= a129f064db70c6a91d6a2ac9af7aeccd2642fb9e
```

ovftoolを使い、OVFの署名したいのですが失敗してしまいます。

``` bash
$ ovftool.exe --privateKey=server.key packer_vmware-iso_vmware.ovf packer_vmware-iso_vmware_signed.ovf
Opening OVF source: packer_vmware-iso_vmware.ovf
The manifest validates
Opening OVF target: packer_vmware-iso_vmware_signed.ovf
Writing OVF package: packer_vmware-iso_vmware_signed.ovf
Transfer Completed
Error: Failed to read the certificate from server.key
Error: Invalid argument
Completed with errors
```

いろいろ試したのですが、手詰まりなのでIDCFクラウドのOVAをそのまま`VMware Player 6.02`にインポートするのは諦めました。

### OVFをVMware Playerにインポートする
`VMware Player 6.02`にOVFを引数にしてインポートを開始します。

``` bash
$ /c/Program\ Files\ \(x86\)/VMware/VMware\ Player/vmplayer.exe packer_vmware-iso_vmware.ovf
```

インポートが終了したら、`VMware Player 6.02`のコンソールからUbuntu14.04のゲストOSにログインします。
* login: packer
* password: packer

``` bash
Ubuntu 14.04 LTS ubuntu-1404 tty1

ubuntu-1404 login: packer
Password:
packer@ubuntu-1404:~$
```

#### まとめ
IDCFクラウドからエクスポートしたOVAをVMXに変換して`VMware Player 6.02`へインポートするのは失敗してしまいました。

しかし、tarで展開した後にVirtualSystemIdentifierをvmx-09に戻したOVFをインポートすることができたので、とりあえず成功とします。

これまでPackerでOVAを作ることを試してみましたが、Dockerを使い普段開発していると、OVA作成の環境構築やビルド、イメージの可搬性はとても厄介に感じます。

Dockerから公式のOSイメージをpullしてきて、その後はDockerfileを随時手直ししつつ、ソースコードはGitで管理して、自分のイメージを作っていくのがこれからのモダンな環境のようです。



