---
layout: post
title: "Setting up Hikey 960 to use KVM and CoreSight"
---

## Summary
As stated [here](https://www.96boards.org/documentation/consumer/hikey/hikey960/downloads/Debian.md.html), ”Currently the snapshot does not support HDMI output and must be administered via LS-UART1”, which literally means that we have to recompile the kernel so that we can get graphical output.
The kernel we should recompile would be [hikey960-upstream-rebase](https://github.com/96boards-hikey/linux/tree/hikey960-upstream-rebase) kernel.
And it also supports CoreSight in [this commit](https://github.com/96boards-hikey/linux/commit/2932b25c8003e773a0b32b52814a5fd17b287567)

## Build the kernel

#### Downloading linaro toolchain
```
mkdir hikey960
cd hikey960
mkdir toolchain
wget https://releases.linaro.org/components/toolchain/binaries/latest-7/aarch64-linux-gnu/gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu.tar.xz
tar -xf gcc-*-x86_64_aarch64-linux-gnu.tar.xz -C ./toolchain --strip-components=1
```

#### Build the kernel
```
git clone https://github.com/96boards-hikey/linux
cd linux
git checkout -b remotes/origin/hikey960-upstream-rebase

export ARCH=arm64
export CROSS_COMPILE=../toolchain/bin/aarch64-linux-gnu-

make hikey960_defconfig
make -j$(nproc) bindeb-pkg # bindeb-pkg is optional
```
