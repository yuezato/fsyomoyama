# FS四方山話
業務でファイルシステムを作る過程で面白いことがたくさん起こるので、
それを書き留めていきます。

## XFSでのベンチマークで不思議なことが起こったので調べた話
https://github.com/yuezato/fsyomoyama/blob/master/xfs_dio_ohanashi.md

## 論理512バイト物理4096バイトHDD（XFS不思議ベンチの続き）
WIP https://github.com/yuezato/fsyomoyama/blob/master/AFT_HDD.md

## 謎の688128バイト、または謎の定数168
まだ

## 1発でHDDにでかいデータを送るには（と、それやる意味があるかどうか）
* Linuxでは、カーネルのバージョンによりますけど、頑張らない場合は
`write(4MB)`とかしても`4MB`単発のリクエストではなく、
分割された小さなリクエスト達がHDDに送られることになります。
* 実際にそれを確認しつつ、巨大なリクエストを送るにはどうすれば良いかと、
それやって何か良いことがあるかどうかを調べます。
* WIP: https://github.com/yuezato/fsyomoyama/blob/master/LargeRequest.md
