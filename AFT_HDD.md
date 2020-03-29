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

## まず512バイト書き込み
```
loop {
 write(512);
 seek(4096 - 512);
}
```
をやります。実際に使うのはこれ: https://gist.github.com/yuezato/3f5b5fdba962defbe4b02f17b38d1212#file-w512-c

```
root@ubuntu:~/simpledd# time ./w512 --of=/root/wks/bench --count=20000
count = 20000

real    0m11.799s
user    0m0.129s
sys     0m0.896s
```

`iostat`すると
```
# iostat -x -d -t -p /dev/sdb 1
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
sdb           3102.00 3102.00  12408.00  12408.00     0.00  3102.00   0.00  50.00    0.14    0.14   0.00     4.00     4.00   0.16 100.00
```

https://github.com/yuezato/fsprof/blob/master/req_lat.bt を使うと
```
yuezato@ubuntu:~/fsprof$ sudo ./req_lat.bt
Attaching 3 probes...
^C

@mq_microtime_stats[4096]: count 3, average 224, total 672
@mq_microtime_stats[512]: count 20000, average 532, total 10644391
```
1クエリ600μsということなんで、1秒では
```
512 byte * (1sec / 532microsec) to kB = 962kB/s
```
iostatと微妙に言ってることが違うけどまあええか……

## 次に4096バイト書き込み
```
loop {
 write(4096);
 seek(4096);
}
```
をやります。seekを入れてるのは512でもseekしているから。
実際に使うプログラムはこれ: https://gist.github.com/yuezato/3f5b5fdba962defbe4b02f17b38d1212#file-w4096-c

実行結果
```
# time ./w4096 --of=/root/wks/bench --count=20000
count = 20000

real    0m5.902s
user    0m0.076s
sys     0m0.332s
```

req_latの方の結果は
```
yuezato@ubuntu:~/fsprof$ sudo ./req_lat.bt
Attaching 3 probes...
^C

@mq_microtime_stats[4096]: count 20002, average 272, total 5451022
```

ということで4096バイトの方が一応速かったです。

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
本来は1回の書き込みのつもりだったので、結構厳しいですね…

このことを実際に確認してみましょう。

## セクタ先頭から512バイト位置で書き出していく
```
  512バイト目（セクタ0の512バイト目)から4KiB書く;
  2セクタ目の先頭までseekする;
  8092+512バイト目（セクタ2の512バイト目）から4KiB書く;
  4セクタ目の先頭までseekする;
  16384+512バイト目（セクタ4の512バイト目）から4KiB書く;
  6セクタ目の先頭までseekする;
  ...
```
という感じでいきます。

実際に使うプログラム: https://gist.github.com/yuezato/3f5b5fdba962defbe4b02f17b38d1212#file-w4096_512-c

```
# time ./w4096_512 --of=/root/wks/bench --count=20000
count = 20000

real    0m15.536s
user    0m0.100s
sys     0m0.578s

yuezato@ubuntu:~/fsprof$ sudo ./req_lat.bt
Attaching 3 probes...
^C

@mq_microtime_stats[4096]: count 20003, average 718, total 14374364
```

上で位置とバイト数が4096バイトアラインメントだと平均で272μsだったことを考えると、
ちょい遅い気もします。とはいえμ秒なんですけどね……

でもまあrealでの実行結果はそこそこ差が出ているのでヨシ。

# 冒頭の例をやる
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
これをやってみましょう。
書き込みは4KiBアラインメントしておいて、ただしオフセットは中途半端な3584(=4096 - 512)バイトにします。

```
# time ./w136k --of=/root/wks/bench --count=5000
count = 5000

real    0m3.166s
user    0m0.023s
sys     0m0.361s

$ sudo ./req_lat.bt
Attaching 3 probes...
^C

@mq_microtime_stats[4096]: count 1, average 261, total 261
@mq_microtime_stats[139264]: count 5000, average 551, total 2755857
```

136KiBの書き込みで大体551μ秒です。

それでは4KiB + 132KiBの分割書き込みだと……？
```
# time ./w4k_132k --of=/root/wks/bench --count=5000
count = 5000

real    0m21.262s
user    0m0.046s
sys     0m0.762s

yuezato@ubuntu:~/fsprof$ sudo ./req_lat.bt
Attaching 3 probes...
^C

@mq_microtime_stats[4096]: count 5004, average 1925, total 9635536
@mq_microtime_stats[135168]: count 5000, average 2133, total 10669231
```
**おっそい！！** ミリ秒の世界に突入してるやんけ……

なんで……？


## 考察
136KiBの書き込みで551μ秒というのは、4096バイト倍数位置で512バイトずつ書いていっているのとほぼ同じ時間になっています。
条件的には、セクタ先頭から512バイトずらした位置から4KiBずつ書いていく718μ秒の方になるかと思うのですが、
この違いはどこから来るのでしょうか？

136KiBの書き込みではseekを行っていないため、HDD内部のキャッシュが効いて、
HDDの仕事としては

# 結論
512e HDD上でDBやストレージシステムを動かす場合は
書き込み開始位置も4096バイト揃えにして、
書き込むバイト数も4096バイトの倍数の、
徹底した4096バイトアライメントが必要になります。

DBやストレージシステム自体にIOスケジューラを実装している場合は、
カーネルのIOスケジューラを抜けた後に大体いい感じになることを
ちゃんと確認しておいた方が良いかもしれません。
