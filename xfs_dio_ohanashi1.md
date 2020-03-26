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
`git clone`したところで`wks`ディレクトリを作り、そこにマウントしました

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
```
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
```
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

# 調査開始
hoge
