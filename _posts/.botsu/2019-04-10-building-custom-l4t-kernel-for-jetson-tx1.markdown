---
layout: post
title: "Building custom L4T kernel for Jetson TX1"
---

This page won't cover how to flash an initial firmware to Jetson otherwise this is for those who want to build a custom L4T kernel **on Jetson**

#### Grab a kernel source
Make sure to check which version of L4T you're using on [Jetpack page](https://developer.nvidia.com/embedded/jetpack). It's Jetpack 3.3 for my environment so L4T 28.2 is the correct one.
After doing that, on [L4T archive](https://developer.nvidia.com/embedded/linux-tegra-archive), select your suitable one ([28.2 for me](https://developer.nvidia.com/embedded/linux-tegra-r282)).
Finally you'll see the source package under *Source Packages*

Bash script below here is an example which is for my settings.
```
wget -N https://developer.nvidia.com/embedded/dlc/l4t-tx1-sources-28-2-ga
tar xvf tx1_sources.tbz2
cd public_release
tar xvf kernel_src.tbz2
cd kernel/kernel-4.4
```

Or just execute a bash script located in Jetpack package.
```
cd path_to_jetpack/64_TX1/Linux_for_Tegra/
./source_sync.sh -t tegra-l4t-r28.2.1
```

#### Build and deploy it

```
# in kernel-4.4 directory
zcat /proc/config.gz > .config

make prepare

make -j$(nproc) Image

sudo cp arch/arm64/boot/Image /boot/Image
```

#### Enable KVM/ARM support

```
# on your host machine

# download your compatible gcc toolchain here
sudo mkdir /opt/l4t-gcc-4.8.5-aarch64
sudo chown $USER:$USER /opt/l4t-gcc-4.8.5-aarch64
cd /opt/l4t-gcc-4.8.5-aarch64
wget https://developer.nvidia.com/embedded/dlc/l4t-gcc-toolchain-64-bit-28-1
mv l4t-gcc-toolchain-64-bit-28-1 gcc-4.8.5-aarch64.tgz
tar -xvf gcc-4.8.5-aarch64.tgz

# Building kernel here
export DEVDIR=/path/to/Linux_for_Tegra

mkdir -p $DEVDIR/images/modules
mkdir -p $DEVDIR/images/packages
mkdir -p $DEVDIR/images/dtb

export KERNEL_MODULES_OUT=$DEVDIR/images/modules
export TEGRA_KERNEL_OUT=$DEVDIR/images

export CROSS_COMPILE=/opt/l4t-gcc-4.8.5-aarch64/install/bin/aarch64-unknown-linux-gnu-
export ARCH=arm64

make O=$TEGRA_KERNEL_OUT tegra21_defconfig
# enable Virtulization in menuconfig
make O=$TEGRA_KERNEL_OUT menuconfig

make O=$TEGRA_KERNEL_OUT zImage -j$(nproc)

```
