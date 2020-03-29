# 導入
次のことは捻くれた考え方をしなければまあ成立するのですが
```ruby
(136*1024) * 3000 = (4*1024 + 132*1024) * 3000
```
これと似たことをHDDでやるとどうなるかという話をします。
```ruby
time(
3000.times {
  write(136KiB);
})
 =?
time(
3000.times {
 write(4KiB);
 write(132KiB);
}) 
```

# 実験に使う環境
まずOSの情報
```
yuezato@ubuntu:~$ cat /etc/os-release
NAME="Ubuntu"
VERSION="19.10 (Eoan Ermine)"
yuezato@ubuntu:~$ uname -r
5.0.0-38-generic
```

実験で使うHDDの情報
```
yuezato@ubuntu:~$ sudo hdparm -I /dev/sdb
[sudo] yuezato のパスワード:

/dev/sdb:

ATA device, with non-removable media
        Model Number:       ST1000DM003-1ER162
        Firmware Revision:  HP51
        Transport:          Serial, SATA 1.0a, SATA II Extensions, SATA Rev 2.5, SATA Rev 2.6, SATA Rev 3.0
Standards:
        Used: unknown (minor revision code 0x001f)
        Supported: 9 8 7 6 5
        Likely used: 9
Configuration:
        Logical         max     current
        cylinders       16383   16383
        heads           16      16
        sectors/track   63      63
        --
        CHS current addressable sectors:    16514064
        LBA    user addressable sectors:   268435455
        LBA48  user addressable sectors:  1953525168
        Logical  Sector size:                   512 bytes
        Physical Sector size:                  4096 bytes
```

## 512e HDD
一番下の2行に注目してもらいたいのですが、
このHDDは論理では1セクタ512バイトですが（＝Linux側からは1セクタ512バイトに見える）
物理では1セクタ4096バイトというセクタサイズを持つものになっています。

このようなHDDは「Advanced FormatなHDD」とか「512e HDD」と呼ばれるようです。

参考: https://www.seagate.com/jp/ja/tech-insights/advanced-format-4k-sector-hard-drives-master-ti/

重要な挙動としては次のことがあります（上のseageteのリンク先でも紹介されています）。

物理的には1セクタ、すなわち最小書き込み単位が4096バイトなので、
従来どおりの512バイト書き込みでは次のような多段の処理が必要になります:
1. 512バイト書き込むには、後続(4096-512)バイトのデータを結合して書き込む必要がある
（何も考えないで書き込みたい512バイトの後ろに3584個の0をつけたりするとデータが消える）
2. そこで一旦4096バイトのデータを読み込む
3. 読み込んだデータの先頭512バイトを（HDDのメモリ上で）差し替えて
4. 更新された4096バイトのデータを書き込む

要するに **readが1回分増える** ということになります。

# 512バイト書き込み vs 4096バイト書き込み
## xfs情報
今回はxfsを使います
```
# xfs_info wks
meta-data=/dev/sdb               isize=512    agcount=4, agsize=61047662 blks
         =                       sectsz=4096  attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=0
data     =                       bsize=4096   blocks=244190646, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=119233, version=2
         =                       sectsz=4096  sunit=1 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

## まず512バイト書き込み
`iostat`すると
```
# iostat -x -d -t -p /dev/sdb 1
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
sdb           3102.00 3102.00  12408.00  12408.00     0.00  3102.00   0.00  50.00    0.14    0.14   0.00     4.00     4.00   0.16 100.00
```

https://github.com/yuezato/fsprof/blob/master/req_lat.bt を使うと
```
@mq_microtime_stats[512]: count 99999, average 596, total 59670039
```
1クエリ600μsということなんで、1秒では
```
512 byte * (1sec / 600microsec) to kB = 853kB/s
```
iostatと微妙に言ってることが違うけどまあええか……

## 次に4096バイト書き込み
```
# iostat -x -d -t -p /dev/sdb 1
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
sdb              0.00 17690.00      0.00  70764.00     0.00     0.00   0.00   0.00    0.00    0.04   0.00     0.00     4.00   0.06 100.00
```

req_latの方の結果は
```
yuezato@ubuntu:~/fsprof$ sudo ./req_lat.bt
Attaching 3 probes...
^C

@mq_microtime_stats[4096]: count 100000, average 37, total 3736434
```

ということで4096バイトの方がやはり圧倒的に速かったです。

# 開始位置も4096バイトアライメントにするべき
無駄な読み込みを発生させないためには4096バイト書き込みの方が良いのですが、
書き込みのoffsetにも注意する必要があります。

物理セクタが1セクタ4KiBということなので、
セクタ開始位置でないところから書き始めると、結局1回のreadが必要になります。

例えば、HDDの512バイト目から4KiB書き込もうとした場合には
1. 4KiBを前半3584バイトと後半512バイトに分けて、
2. HDDの先頭1セクタを読み込み、そのsuffixの3584バイトを差し替えて、先頭セクタを上書きし
3. HDDの2セクタ目を読み込み、そのprefixの512バイトを差し替えて、上書き
ということで2回の読み込みと2回の書き込みが発生します。

このことを実際に確認してみましょう。

## hoge
