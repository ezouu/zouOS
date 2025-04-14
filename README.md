# STM32MP1 Custom Kernel Build & Deployment Guide

This guide documents the full process to build, deploy, and optionally repackage a custom Linux kernel for the STM32MP1 platform using ST's OpenSTLinux ecosystem.

## Prerequisites

- STM32MPU Developer Package and SDK installed
- Starter Package extracted
- Serial connection to the board
- STM32MP1 Evaluation Board (e.g., STM32MP157C-EV1)
- USB OTG cable or SD card reader
- Linux PC (host)

## Step 1: Initialize SDK Environment

```bash
source ~/STM32MPU_workspace2/Developer-Package/SDK/environment-setup-cortexa7t2hf-neon-vfpv4-ostl-linux-gnueabi
```

## Step 2: Extract and Patch Kernel Source

```bash
cd ~/STM32MPU_workspace2/Developer-Package/.../linux-stm32mp-6.6.48-stm32mp-r1-r0
tar xf linux-6.6.48.tar.xz
cd linux-6.6.48
for p in `ls -1 ../*.patch`; do patch -p1 < $p; done
```

## Step 3: Create Build Directory

```bash
export OUTPUT_BUILD_DIR=$PWD/../build
mkdir -p ${OUTPUT_BUILD_DIR}
```

## Step 4: Configure Kernel with ST Fragments

```bash
make O="${OUTPUT_BUILD_DIR}" defconfig fragment*.config
for f in `ls -1 ../fragment*.config`; do scripts/kconfig/merge_config.sh -m -r -O ${OUTPUT_BUILD_DIR} ${OUTPUT_BUILD_DIR}/.config $f; done
(yes '' || true) | make oldconfig O="${OUTPUT_BUILD_DIR}"
```

## Step 5: Build Kernel, DTBs, and Modules

```bash
export ARCH=arm
export IMAGE_KERNEL=uImage

make ${IMAGE_KERNEL} vmlinux dtbs LOADADDR=0xC2000040 O="${OUTPUT_BUILD_DIR}"
make modules O="${OUTPUT_BUILD_DIR}"
make INSTALL_MOD_PATH="${OUTPUT_BUILD_DIR}/install_artifact" modules_install O="${OUTPUT_BUILD_DIR}"

mkdir -p ${OUTPUT_BUILD_DIR}/install_artifact/boot/
cp ${OUTPUT_BUILD_DIR}/arch/${ARCH}/boot/${IMAGE_KERNEL} ${OUTPUT_BUILD_DIR}/install_artifact/boot/
find ${OUTPUT_BUILD_DIR}/arch/${ARCH}/boot/dts/ -name 'st*.dtb' -exec cp '{}' ${OUTPUT_BUILD_DIR}/install_artifact/boot/ \;
```

## Step 6: Copy to SD Card via USB Mass Storage (U-Boot)

On the board (U-Boot Prompt):

```bash
ums 0 mmc 0
```

On the Host (PC):

```bash
lsblk
sudo mount /dev/sdX8 /mnt/bootfs
sudo mount /dev/sdX10 /mnt/rootfs

cd ${OUTPUT_BUILD_DIR}/install_artifact
sudo cp -r boot/* /mnt/bootfs/
rm lib/modules/*/source lib/modules/*/build
sudo cp -r lib/modules/* /mnt/rootfs/lib/modules/

sudo umount /mnt/bootfs
sudo umount /mnt/rootfs
```

## Step 7: Finalize on the Board

```bash
depmod -a
sync
reboot
```

## Step 8 (Optional): Update Starter Package Images

```bash
export STARTER_DIR=~/STM32MPU_workspace2/Starter-Package/stm32mp1-openstlinux-6.6-yocto-scarthgap-mpu-v24.11.06

mkdir -p ${STARTER_DIR}/bootfs_mounted
sudo mount -o loop ${STARTER_DIR}/images/stm32mp1/st-image-bootfs-openstlinux-weston-stm32mp1.ext4 ${STARTER_DIR}/bootfs_mounted
sudo cp -vf ${OUTPUT_BUILD_DIR}/install_artifact/boot/${IMAGE_KERNEL} ${STARTER_DIR}/bootfs_mounted/
sudo cp -vf ${OUTPUT_BUILD_DIR}/install_artifact/boot/st*.dtb ${STARTER_DIR}/bootfs_mounted/
sudo umount ${STARTER_DIR}/bootfs_mounted
rmdir ${STARTER_DIR}/bootfs_mounted

mkdir -p ${STARTER_DIR}/rootfs_mounted
sudo mount -o loop ${STARTER_DIR}/images/stm32mp1/st-image-weston-openstlinux-weston-stm32mp1.ext4 ${STARTER_DIR}/rootfs_mounted
sudo cp -rvf ${OUTPUT_BUILD_DIR}/install_artifact/lib/modules/* ${STARTER_DIR}/rootfs_mounted/lib/modules
sudo umount ${STARTER_DIR}/rootfs_mounted
rmdir ${STARTER_DIR}/rootfs_mounted
```

## Verification

```bash
uname -r
lsmod
modinfo /lib/modules/$(uname -r)/kernel/<path-to>.ko
```

