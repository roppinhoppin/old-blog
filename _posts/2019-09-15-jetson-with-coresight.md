---
layout: post
title: "Using Jetson TX2 with CoreSight"
---



このページでは，Jetpack 4.4を焼いたJetson TX2の環境を想定している．あと完全に自分用のメモ．

<!-- TOC -->

- [Building Custom Kernel](#building-custom-kernel)
- [Get trace output from perf](#get-trace-output-from-perf)
- [sysFSから生えてるデバイスから直接トレースしたデータを取得する](#sysfsから生えてるデバイスから直接トレースしたデータを取得する)

<!-- /TOC -->

### Building Custom Kernel

**Jetson TX2上で行う．**
L4Tカーネルのソースのダウンロードとビルドコンフィグの設定
```
mkdir 32.2.1
cd 32.2.1
wget -N https://developer.nvidia.com/embedded/dlc/r32-2-1_Release_v1.0/TX2-AGX/sources/public_sources.tbz2
tar xvf public_sources.tbz2
cd public_sources
tar xvf kernel_src.tbz2
cd kernel/kernel-4.9
zcat /proc/config.gz > .config
make menuconfig
```

menuconfigの画面にて，Kernel Hacking->CoreSight Tracing Supportの全ての要素をyに変更．

Denverコアを有効にするとコア数が2個増えてビルドが早くなって良さそう．平均的にビルド時間は40分くらいかかる．
```
make -j$(nproc)
sudo make install
sudo mv /boot/Image /boot/Image.old
sudo cp ./arch/arm64/boot/Image /boot/Image
```

そして再起動すればCoreSightのサポートが入っているカーネルに差し代わる．

### Get trace output from perf

```
ls /sys/bus/coresight/devices/ 
```

にてetfの番号を確認．筆者の環境では8030000.etfだったので，

```
cd path/to/kernel-4.9
make CORESIGHT=1 VF=1 -C tools/perf
sudo tools/perf/perf record -vvv -e cs_etm/@8030000.etf/u --cpu 0 uname
```

こんな感じでperfでもトレースが取れる．

```
sudo perf report --stdio --dump
```
このコマンドでダンプされたperf.dataが読めるがJetson TX2で試したところ

```
0x178 [0x70]: failed to process type: 70
Error:
failed to process sample
# To display the perf.data header info, please use --header/--header-only options.
#

0x178 [0x70]: event: 70
```

**とのエラーが出たのであまり使えなさそう．**

### sysFSから生えてるデバイスから直接トレースしたデータを取得する

ptm2humanをコンパイル
```
git clone https://github.com/hwangcc23/ptm2human
cd ptm2human 
./autogen.sh
./configure
make
```

sysFSに生えているデバイスを有効化
```
sudo -s

cd /sys/bus/coresight/devices/

echo 1 > 8030000.etf/enable_sink
echo 1 > 8050000.etr/enable_sink
echo 1 > 9840000.ptm/enable_source
```

```
cd path/to/your_workspace
dd if=/dev/8030000.etf of=trace.bin
ptm2human -i trace.bin
```
