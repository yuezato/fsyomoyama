# XFSでのベンチマークで不思議なことが起こったので調べた話

## どういうベンチマークをしていた?
疑似コードで書くとこういうやつ
```
let file  = 起動オプション["ベンチマークで書き込み対象になるファイル"];
let start = 起動オプション["書き込み開始位置"];
let size  = 起動オプション["1回の書き込みバイト数"];
let count = 起動オプション["何回書き込むか"];

// ファイルをO_DIRECTモードで開く
// もし存在しなければ、新規に作成する
let fd = open(file).with_odirect(true);

fd.seek(start);

let buffer = allocate(size);

for(int i = 0; i < count; ++i) {
 write(fd, buffer);
} 
```
+ ファイルを新規に作成し、O_DIRECTモードで開き、
+ 起動オプションで指定された位置にずらし
+ 指定されたバイト数で、指定された回数だけシーケンシャルにデータを書き出す

## 何が起こった?
+ ファイルを新規に生成するパターンで、書き込み開始位置を先頭位置でないところからやると遅くなる

## 遅くなることを実際に確認してみる
上の疑似コードに対応するベンチマークプログラムがこれ: https://github.com/yuezato/diskbenchi

### まずはgit cloneしてxfsをマウント
今回は `/dev/sdb` が実験に使える環境だったので、これを使います。

```shell
yuezato@ubuntu:~$ git clone https://github.com/yuezato/diskbenchi; cd diskbenchi
yuezato@ubuntu:~/diskbenchi$ sudo mkfs.xfs -f /dev/sdb
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
yuezato@ubuntu:~/diskbenchi$ mkdir wks
yuezato@ubuntu:~/diskbenchi$ sudo mount /dev/sdb wks
yuezato@ubuntu:~/diskbenchi$ sudo chmod a+w wks
yuezato@ubuntu:~/diskbenchi$ lsblk /dev/sdb
NAME MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sdb    8:16   0 931.5G  0 disk /home/yuezato/diskbenchi/wks
yuezato@ubuntu:~/diskbenchi$ cat /etc/os-release
NAME="Ubuntu"
VERSION="19.10 (Eoan Ermine)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 19.10"
VERSION_ID="19.10"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=eoan
UBUNTU_CODENAME=eoan
yuezato@ubuntu:~/diskbenchi$ uname -ar
Linux ubuntu 5.0.0-38-generic #41-Ubuntu SMP Tue Dec 3 00:27:35 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
```
`git clone`したところで`wks`ディレクトリを作り、そこにマウントしました。

### /dev/sdbの中身は？
`/dev/sdb`の中身は次の通りで、論理512バイトセクタ物理4096バイトセクタのいわゆる4KiB AFT HDDというやつです:
```
yuezato@ubuntu:~/fsprof$ sudo hdparm -I /dev/sdb
/dev/sdb:

ATA device, with non-removable media
        Model Number:       ST1000DM003-1ER162
        Serial Number:      ***
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

### 実験開始
`cargo build --release`して
* 1回あたり2MiBの書き込みを2000回行う
* ファイル先頭からと、ファイル先頭3584バイトから行うので2種類
```shell
yuezato@ubuntu:~/diskbenchi$ rm -f wks/bench && time ./target/release/diskbenchi --of wks/bench --bs $((2*1024*1024)) --count 2000 --offset 0
Opt {
    bs: 2097152,
    count: 2000,
    of: "wks/bench",
    offset: Some(
        0,
    ),
    hugepool: false,
}

real    0m18.216s <--- だいたい18秒ね
user    0m0.000s
sys     0m0.661s
yuezato@ubuntu:~/diskbenchi$ rm -f wks/bench && time ./target/release/diskbenchi --of wks/bench --bs $((2*1024*1024)) --count 2000 --offset $((4096-512))
Opt {
    bs: 2097152,
    count: 2000,
    of: "wks/bench",
    offset: Some(
        3584,
    ),
    hugepool: false,
}

real    0m21.299s <--- 21秒は遅くない……？
user    0m0.008s
sys     0m0.584s
```

なんかちょっと差がついてない……？

書き込み回数を3000回にしてみよう
```shell
yuezato@ubuntu:~/diskbenchi$ rm -f wks/bench && time ./target/release/diskbenchi --of wks/bench --bs $((2*1024*1024)) --count 3000 --offset 0
Opt {
    bs: 2097152,
    count: 3000,
    of: "wks/bench",
    offset: Some(
        0,
    ),
    hugepool: false,
}

real    0m27.311s <--- 27秒ね
user    0m0.009s
sys     0m0.653s

yuezato@ubuntu:~/diskbenchi$ rm -f wks/bench && time ./target/release/diskbenchi --of wks/bench --bs $((2*1024*1024)) --count 3000 --offset $((4096-512))
Opt {
    bs: 2097152,
    count: 3000,
    of: "wks/bench",
    offset: Some(
        3584,
    ),
    hugepool: false,
}

real    0m32.029s <--- 32秒か……
user    0m0.004s
sys     0m0.665s
```
なんか差が開いてないですかね？

4000回にしてみると
```shell
yuezato@ubuntu:~/diskbenchi$ rm -f wks/bench && time ./target/release/diskbenchi --of wks/bench --bs $((2*1024*1024)) --count 4000 --offset 0
Opt {
    bs: 2097152,
    count: 4000,
    of: "wks/bench",
    offset: Some(
        0,
    ),
    hugepool: false,
}

real    0m36.392s <--- 36秒か ということは offset=0 だと 1000回を9秒になりそう
user    0m0.018s
sys     0m1.116s

yuezato@ubuntu:~/diskbenchi$ rm -f wks/bench && time ./target/release/diskbenchi --of wks/bench --bs $((2*1024*1024)) --count 4000 --offset $((4096-512))
Opt {
    bs: 2097152,
    count: 4000,
    of: "wks/bench",
    offset: Some(
        3584,
    ),
    hugepool: false,
}

real    0m52.200s <--- 52秒って遅すぎるやんけ！！
user    0m0.044s
sys     0m0.938s
```

## 調査開始
### bpftrace
bpftraceで書いた自作のI/Oレイテンシチェッカを使います。
幾つかバリエーションがあるんですが、今回はこれ: https://github.com/yuezato/fsprof/blob/master/req_lat.bt
何をするかというと
+ `/dev/sdb` 向けの I/Oリクエスト をターゲットにして https://github.com/yuezato/fsprof/blob/master/req_lat.bt#L6
+ I/OリクエストがデバイスドライバからHDDに向かって飛び出して、HDDがwrite completionを上げるまでの時間（レイテンシ）を測定

bpftrace( https://github.com/iovisor/bpftrace )は超便利なので、使ったことがない方は使ってみてください。
Ubuntuならインストールガイド https://github.com/iovisor/bpftrace/blob/master/INSTALL.md の通りやると、
サクッと入ると思います。

ただ、手元ではソースからビルドした次のバージョンを使っていて、
今回使うbpftraceプログラムは`apt`で入るバージョンだと動かないかもしれません。
```
yuezato@ubuntu:~/fsprof$ sudo bpftrace -V
bpftrace v0.9.4-38-gd079
```

### 早速使ってみる
まず `req_lat.bt` を sudo なりなんなりで起動して待機しておきます（bpftraceはroot権限が基本的に必要です）:
```shell
yuezato@ubuntu:~/fsprof$ sudo ./req_lat.bt
Attaching 3 probes...
```
その傍らで別のシェルプロセスで、`offset=0`にして100件書き込みしてみます:
```shell
yuezato@ubuntu:~/diskbenchi$ rm -f wks/bench && time ./target/release/diskbenchi --of wks/bench --bs $((2*1024*1024)) --count 100 --offset 0
（省略）
```

すると
``` shell
yuezato@ubuntu:~/fsprof$ sudo ./req_lat.bt
Attaching 3 probes...
sec: 3973320, size: 688128, segs: 168, time:1.858[ms]
sec: 3974664, size: 688128, segs: 168, time:1.698[ms]
sec: 3976008, size: 688128, segs: 168, time:1.698[ms]
sec: 3977352, size: 32768, segs: 8, time:0.264[ms]
sec: 3977416, size: 688128, segs: 168, time:1.667[ms]
sec: 3978760, size: 688128, segs: 168, time:1.416[ms]
sec: 3980104, size: 688128, segs: 168, time:2.212[ms]
sec: 3981448, size: 32768, segs: 8, time:0.242[ms]
sec: 3981512, size: 688128, segs: 168, time:1.727[ms]
sec: 3982856, size: 688128, segs: 168, time:1.716[ms]
sec: 3984200, size: 688128, segs: 168, time:1.745[ms]
sec: 3985544, size: 32768, segs: 8, time:0.222[ms]
sec: 3985608, size: 688128, segs: 168, time:1.658[ms]
sec: 3986952, size: 688128, segs: 168, time:1.461[ms]
sec: 3988296, size: 688128, segs: 168, time:1.731[ms]
sec: 3989640, size: 32768, segs: 8, time:0.221[ms]
sec: 3989704, size: 688128, segs: 168, time:27.129[ms]
sec: 3991048, size: 688128, segs: 168, time:1.821[ms]
sec: 3992392, size: 688128, segs: 168, time:1.475[ms]
sec: 3993736, size: 32768, segs: 8, time:1.247[ms]
sec: 3993800, size: 688128, segs: 168, time:2.559[ms]
sec: 3995144, size: 688128, segs: 168, time:1.452[ms]
sec: 3996488, size: 688128, segs: 168, time:1.449[ms]
sec: 3997832, size: 32768, segs: 8, time:2.99[ms]
sec: 3997896, size: 688128, segs: 168, time:1.636[ms]
sec: 3999240, size: 688128, segs: 168, time:1.447[ms]
sec: 4000584, size: 688128, segs: 168, time:1.448[ms]
sec: 4001928, size: 32768, segs: 8, time:1.265[ms]
...
```

100回の書き込みがI/O request化されたものが出てきました。
このrequest達はデバイスドライバによってデバイスに投げられる直前のもので、極めてlow layerな情報を持っています。
出力を眺めてみると、基本的には
```
sec: 3973320, size: 688128, segs: 168, time:1.858[ms]
sec: 3974664, size: 688128, segs: 168, time:1.698[ms]
sec: 3976008, size: 688128, segs: 168, time:1.698[ms]
sec: 3977352, size: 32768, segs: 8, time:0.264[ms]

sec: 3977416, size: 688128, segs: 168, time:1.667[ms]
sec: 3978760, size: 688128, segs: 168, time:1.416[ms]
sec: 3980104, size: 688128, segs: 168, time:2.212[ms]
sec: 3981448, size: 32768, segs: 8, time:0.242[ms]

sec: 3981512, size: 688128, segs: 168, time:1.727[ms]
sec: 3982856, size: 688128, segs: 168, time:1.716[ms]
sec: 3984200, size: 688128, segs: 168, time:1.745[ms]
sec: 3985544, size: 32768, segs: 8, time:0.222[ms]
```
という4つのrequestの繰り返しになっていることが分かります。
もともとは `2MiB = 2 * 1024 * 1024byte` の書き込みだったのですが、
これが`688128`バイトのrequest3つと、`32768`バイトのrequest1つに分割されています。
(これで2MiBになっているか確認: https://www.wolframalpha.com/input/?i=2+*+1024+*+1024+%3D+688128+*+3+%2B+32768&lang=ja )

なぜ 688128 という謎の数字が出てきているのかといったことも、
完全に説明がつくのですが、今回からは脱線してしまうのでちょっとお休みして……

次に`offset`をなぜか遅くなってしまう`3584`にしてlatencyを同様に採集してみると次のようになります:
```
yuezato@ubuntu:~/fsprof$ sudo ./req_lat.bt
Attaching 3 probes...
sec: 409800, size: 3584, segs: 1, time:0.248[ms]
sec: 409807, size: 688128, segs: 168, time:1.732[ms]
sec: 411151, size: 688128, segs: 168, time:1.694[ms]
sec: 412495, size: 688128, segs: 168, time:1.694[ms]
sec: 413839, size: 33280, segs: 9, time:0.219[ms]

sec: 413903, size: 512, segs: 1, time:0.144[ms]
sec: 413904, size: 687616, segs: 168, time:2.413[ms]
sec: 415247, size: 688128, segs: 168, time:1.458[ms]
sec: 416591, size: 688128, segs: 168, time:2.165[ms]
sec: 417935, size: 33280, segs: 9, time:0.229[ms]
...
...
sec: 434383, size: 512, segs: 1, time:0.442[ms] ★
sec: 434384, size: 687616, segs: 168, time:4.70[ms]
sec: 435727, size: 688128, segs: 168, time:1.734[ms]
sec: 437071, size: 688128, segs: 168, time:1.712[ms]
sec: 438415, size: 33280, segs: 9, time:0.230[ms]

sec: 438479, size: 512, segs: 1, time:0.162[ms]
sec: 438480, size: 687616, segs: 168, time:4.822[ms]
sec: 439823, size: 688128, segs: 168, time:1.497[ms]
sec: 441167, size: 688128, segs: 168, time:1.727[ms]
sec: 442511, size: 33280, segs: 9, time:0.215[ms]

sec: 442575, size: 512, segs: 1, time:0.158[ms]
sec: 442576, size: 687616, segs: 168, time:5.530[ms]
sec: 443919, size: 688128, segs: 168, time:1.838[ms]
sec: 445263, size: 688128, segs: 168, time:1.690[ms]
sec: 446607, size: 33280, segs: 9, time:0.217[ms]
sec: 446671, size: 512, segs: 1, time:0.154[ms]
```
`512 + 687616 + 688128*2 + 33280`のパターンが繰り返されているのですが、これって実は2MiBより大きいんですよね。512バイト多い。
（参考: https://www.wolframalpha.com/input/?i=2+*+1024+*+1024+%3C+512+%2B+687616+%2B+688128*2+%2B+33280&lang=ja )

2MiBの書き込みが512バイト余分に増えてたら、xfsのバグじゃん！！と思うのですが、実はそうではないんですなこれが……。
そうでないと言い切れる説明をしていきます。
その説明の前に、687616バイトの書き込みがどうも毎回遅いんですよね。他は2msぐらいなのに、688KB前後の書き込みで5ms秒ぐらいに膨れ上がっている。なぜ？
ここが遅くなる原因の気がします。

しかし遅くなる以前に、余分な512バイトの先に突き止めたいと思います。そうすると自ずと遅くなる原因が分かります。

とりあえずsecの増え方に注目してみます。説明のため、★をつけた`sec: 434383, size: 512, segs: 1, time:0.442[ms] ★`のところから
1. 434383セクタから512バイト=1セクタ書き込むと、434384セクタだ。実際に次はそこから書き込んでいるので、ヨシ
2. 434384セクタから687616バイト=1343セクタ書き込むと、435727。ヨシ
4. そこから688128バイト=1344セクタ書き込むと、437071。ヨシ
5. 次も688128バイト=1344セクタ書き込むと、438415。ヨシ
6. 次は33280バイト=65セクタ書き込むから、次は438415+65=438480から書き込む筈だ
7. あれ！？ 次は `sec: 438479, size: 512, segs: 1, time:0.162[ms]` になってる

ということは、33280バイト書いた最後の1セクタが、次の1セクタに上書きされている……？

上書きされているとすると、ここから次の2つが導けます
a. やっぱり本質的には2MiBしか書き込んでいない。
b. 33280バイト書き込みが完了するまで、次の512バイトでの上書きができず、この待ち合わせが（なぜか）687616の書き込みに現れている？

## 512バイトの上書きはどこから来るのか？
これは別のIOプロファイラを使うと説明がつきます: https://github.com/yuezato/fsprof/blob/master/iomap_dio_trace.bt
だいぶ細かい情報が出てしまうので、出力を整形したものを次に記載します:
```
vfs_write {
|new_sync_write {
||iomap_dio_rw (sync=1) {
|||iomap_apply (pos = 176164352, length = 2097152) {
||||iomap_dio_bio_actor(inode->i_size = 176164352, pos = 176164352, length = 512, dio->flags | DIO_WRITE = 1) {
|||||generic_make_request (bio.iter_size = 512);
||||}
|||}
|||iomap_apply (pos = 176164864, length = 2096640) {
||||iomap_dio_bio_actor(inode->i_size = 176164352, pos = 176164864, length = 2096640, dio->flags | DIO_WRITE = 1) {
|||||generic_make_request (bio.iter_size = 2096640);
|||||generic_make_request (bio.iter_size = 1409024);
|||||generic_make_request (bio.iter_size = 720896);
|||||generic_make_request (bio.iter_size = 32768);
|||||iomap_dio_zero (pos = 165678592) {
||||||generic_make_request (bio.iter_size = 512);
|||||}
||||}
|||}
||}
|}
}
```
これはwriteシステムコールのうちで、I/O requestの形成に関わる部分を抜粋するトレーサです。
上からみていくと
1. 2097152=2MiBの書き込みから、まず512バイトのrequestが作られ
2. 残り2096640バイトから687616バイトのrequestが作られ、
3. 残り1490924バイトから688128バイトのrequestが作られ、
4. 残り720896バイトから688128バイトのrequestが作られ、
5. 残った32768バイトのrequestを作った後に
6. `iomap_dio_zero`という関数の中で「追加で512バイト」のrequestが生成されます。

実は上で見た6512, 687616, 688128*2, 33280というI/Oリクエストは、
このカーネル内部で生成されたrequestの最後2つ（32768 + 512）が結合したものです。

`iomap_dio_zero`が謎の512バイトの追加requestを生成しなければ、全て丸く収まるはずです。

** iomap_dio_zeroとは……?
https://elixir.bootlin.com/linux/v5.0/source/fs/iomap.c#L1732 から呼び出される
`iomap_dio_zero` https://elixir.bootlin.com/linux/v5.0/source/fs/iomap.c#L1579 に焦点を絞ります。

といっても、`iomap_dio_zero`の実装ではなく、「なぜこれが呼び出されるか？」を説明する必要があります。
まあこれは https://elixir.bootlin.com/linux/v5.0/source/fs/iomap.c#L1721 からはじまる4行のコメントに答えが書いているのですが。

引用します
> 	/*
>	 * We need to zeroout the tail of a sub-block write if the extent type
>	 * requires zeroing or the write extends beyond EOF. If we don't zero
>	 * the block tail in the latter case, we can expose stale data via mmap
>	 * reads of the EOF block.
>	 */
> https://elixir.bootlin.com/linux/v5.0/source/fs/iomap.c#L1720
