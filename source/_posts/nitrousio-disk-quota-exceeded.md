title: "Nitrous.IOでDisk quota exceededがでてgit pushできない"
date: 2014-09-21 10:22:29
tags:
 - NitrousIO
 - GitHubPages
 - Hexo
description: Nitrous.IOで開発しているHexoのブログをGitHubにpushしようとするとDisk quota exceededエラーが発生するようになりました。GitHub Pagesにdeployできません。メッセージ内容からディスク容量が足りなくなっているようです。
---

Nitrous.IOで開発しているHexoのブログをGitHubにpushしようとすると`Disk quota exceeded`エラーが発生するようになりました。[GitHub Pages](https://pages.github.com/)にdeployできません。メッセージ内容からディスク容量が足りなくなっているようです。

``` bash
$ hexo deploy
[info] Start deploying: github
[info] Clearing .deploy folder...
[info] Copying files from public folder...
error: file write error (Disk quota exceeded)
fatal: unable to write sha1 file
fatal: unable to write new_index file
error: failed to write new configuration file .git/config.lock
error: failed to write new configuration file .git/config.lock
Everything up-to-date
Branch master set up to track remote branch master from github.
[info] Deploy done: github
```

ディスク使用量を計算すると、1000Mになっています。

``` bash
$ du -sh ~/
1000M   /home/action/
```

Get More N2Oのbonousページで、15 N2O獲得
FaebookやLinkedInとリンクさせて、ストレージで750MB相当を獲得

再度デプロイ

``` bash
$ hexo deploy
...
[info] Deploy done: github
```

ディスク使用量を確認すると、1005MBになりました。

``` bash
$ du -sh ~/
1005M   /home/action/
```

そろそろPaidプランに移行しないといけないです。


