title: "iPhoneのヘルスケアデータから歩数を日別に集計してCSVファイルにする"
date: 2016-12-12 16:46:26
tags:
 - Docker
 - Python
 - iPhone
 - ヘルスケア
description: Android WearやApple Watchが出始めの頃は楽しんで着けていたのですが、やはり時計は気に入ったものをしたいので活動量計としても使わなくなってしまいました。
---

　Android WearやApple Watchが出始めの頃は楽しんで着けていたのですが、やはり時計は気に入ったものをしたいので活動量計としても使わなくなってしまいました。活動量計を着けなくても日常持ち歩いているiPhoneには標準でヘルスケアデータを記録できるアプリがインストールされています。データも溜まってきたのでiPhoneから書き出してデータ分析用に使ってみたいと思います。
　

<!-- more -->

## iPhoneからヘルスケアデータを書き出す

　iPhoneアプリのヘルスケアを開き右上のプロファイルアイコンをタップします。

![health-1.png](/2016/12/12/iphone-health-data-steps-csv/health-1.png)

　プロファイルページにある`ヘルスケアデータを書き出す`をタップします。
　
![healt-2.png](/2016/12/12/iphone-health-data-steps-csv/health-2.png)
　
　確認ダイアログの`書き出す`をタップします。
　
![healt-3.png](/2016/12/12/iphone-health-data-steps-csv/health-3.png)


　ヘルスケアデータを書き出したいサービスをタップします。
　
![healt-4.png](/2016/12/12/iphone-health-data-steps-csv/health-4.png)

　
　iCloud Driveを選択すると同期しているPCのiCloud Driveフォルダに`書き出したデータ.zip`のファイル名でアーカイブが保存されます。
　
## CSVコンバーター

　ヘルスケアデータは`書き出したデータ.zip`の中にあるXML形式の`書き出したデータ.xml`ファイルです。歩数データはエクセルで管理しているのでコピー＆ペーストしやすいようにCSVにコンバートするスクリプトを書きました。
　
　
　使い方は最初に[ここ](https://github.com/masato/health-data-csv.git)からリポジトリをcloneします。


```bash
$ git clone https://github.com/masato/health-data-csv.git
$ cd health-data-csv
```

　`書き出したデータ.zip`ファイルをcloneしたディレクトリにコピーします。macOSの場合iCloud Driveは以下のディレクトリになります。パスに半角スペースがあるためダブルクォートします。

```bash
$ cp "$HOME/Library/Mobile Documents/com~apple~CloudDocs/書き出したデータ.zip" .
```

　`convert.py`はZipファイルからヘルスケアデータのXMLを取り出し歩数を日別に集計してCSVファイルに出力するPythonスクリプトです。`type`を`HKQuantityTypeIdentifierStepCount`に指定して`Record`要素から歩数データだけ抽出しています。[Pythonによるデータ分析入門 ―NumPy、pandasを使ったデータ処理](https://www.amazon.co.jp/dp/4873116554/)を勉強しているところなのでデータ分析ツールの[pandas](http://pandas.pydata.org/)を使い集計とCSVへの書き出しを実装してみます。
　
　[Python 3 で日本語ファイル名が入った zip ファイルを扱う](http://qiita.com/methane/items/8493c10c19ca3584d31d)の記事によると`書き出したデータ.xml`のように日本語ファイル名は`cp437`でデコードされるようです。


```python convert.py
# -*- coding: utf-8 -*-

from lxml import objectify
import pandas as pd
from pandas import DataFrame
from dateutil.parser import parse
from datetime import datetime
import zipfile
import argparse
import sys, os

def main(argv):

    parser = argparse.ArgumentParser()
    parser.add_argument('-f', '--file',
                        default='書き出した.zip',
                        type=str,
                        help='zipファイル名 (書き出した.zip)')
    parser.add_argument('-s', '--start',
                        action='store',
                        default='2016-01-01',
                        type=str,
                        help='開始日 (2016-12-01)')

    args = parser.parse_args()

    if not os.path.exists(args.file):
        print('zipファイル名を指定してください。')
        parser.print_help()
        sys.exit(1)

    zipfile.ZipFile(args.file).extractall()

    parsed = objectify.parse(open('apple_health_export/書き出したデータ.xml'
                                  .encode('utf-8').decode('cp437')))

    root = parsed.getroot()

    headers = ['type', 'unit', 'startDate', 'endDate', 'value']

    data = [({k: v for k, v in elt.attrib.items() if k in headers})
            for elt in root.Record]

    df = DataFrame(data)
    df.index = pd.to_datetime(df['startDate'])

    # 歩数だけ
    steps = df[df['type'] == 'HKQuantityTypeIdentifierStepCount'].copy()
    steps['value'] = steps['value'].astype(float)

    # 開始日が条件にある場合スライス
    if args.start:
        steps = steps.loc[args.start:]

    # 日別にグループ化して集計
    steps_sum = steps.groupby(pd.TimeGrouper(freq='D')).sum()

    steps_sum.T.to_csv('./steps_{0}.csv'.format(datetime.now().strftime('%Y%m%d%H%M%S')),
                       index=False, float_format='%.0f')

if __name__ == '__main__':
    main(sys.argv[1:])
```

## Pythonスクリプトの実行

　スクリプトの実行はDockerイメージは[continuumio/anaconda3](https://github.com/ContinuumIO/docker-images/tree/master/anaconda3)を使います。データ分析に[Anaconda](https://www.continuum.io/downloads)を使うDockerイメージです。[Jupyter](http://jupyter.org/)もインストールされています。
　
　Pythonスクリプトは`-f`フラグでヘルスケアから書き出したカレントディレクトリにあるzipファイル名を指定します。`-s`フラグはCSVにコンバートするレコードの開始日を指定できます。


```bash
$ docker pull continuumio/anaconda3
$ docker run -it --rm \
  -v $PWD:/app \
  -w /app \
  continuumio/anaconda3 \
  python convert.py -f 書き出したデータ.zip -s 2016-12-01
```


　カレントディレクトリに「steps_xxx.csv」のような歩数を日別に集計したCSVファイルが作成されました。

```bash
$ cat steps_20161212013800.csv
2016-12-01,2016-12-02,2016-12-03,2016-12-04,2016-12-05,2016-12-06,2016-12-07,2016-12-08,2016-12-09,2016-12-10,2016-12-11
7217,8815,2260,1828,3711,6980,7839,5079,7197,7112,2958
```
