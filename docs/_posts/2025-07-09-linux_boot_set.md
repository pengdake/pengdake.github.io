---
layout: single
toc: true
toc_sticky: true
categories: [linux]
tags: [linux, fstab]
---


# linux分区挂载设置

## 背景
家里树莓派通过usb挂载的磁盘存储，有时会插拔存储对应的usb接口，磁盘分区名称可能会发生变化，从而导致开机引导失败，为了避免此种情况，需要在/etc/fstab中设置基于uuid的挂载

## uuid
这里的uuid是唯一标识磁盘分区的标识符，是 Linux 中识别磁盘设备最推荐的方式之一，尤其适用于多磁盘、USB 接口不固定的情况。在格式化分区的时候生成，除非格式化或通过其它方式人为改动，否则不会变化

## 操作示例
获取uuid
```
sudo blkid
/dev/sda1: LABEL_FATBOOT="boot" LABEL="boot" UUID="54E3-79CE" TYPE="vfat" PARTUUID="d5f8932e-01"
/dev/sda2: LABEL="rootfs" UUID="c6dd3b94-a789-4d57-9080-1472f721804b" TYPE="ext4" PARTUUID="d5f8932e-02"
```
更新fstab
```
cat /etc/fstab
proc            /proc           proc    defaults          0       0
/dev/sda1  /boot           vfat    defaults          0       2
/dev/sda2  /           ext4    defaults,noatime          0       1

sudo sed -i 's|^/dev/sda1|UUID=54E3-79CE|g' /etc/fstab
sudo sed -i 's|^/dev/sda2|UUID=c6dd3b94-a789-4d57-9080-1472f721804b|g' /etc/fstab
```






