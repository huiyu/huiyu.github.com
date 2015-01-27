---
layout: post
title: "CentOS使用本地ISO镜像作为yum源"
categories: Linux
tags: [CentOS, yum]
---

CentOS采用yum（Yellow dog Updater, Modified）作为RMP包管理器，默认需要联网才能更新安装软件包，在离线状态下非常麻烦。这里采用修改yum源指向本地挂载ISO镜像，可以在离线状态下使用yum命令更新软件包。

## 挂载

创建iso存放目录和挂载目录：

```bash
# mkdir /mnt/iso
# mkdir /mnt/cdrom
```

将镜像scp到`/mnt/iso`目录，然后将`/mnt/iso`下的镜像文件挂载到`/mnt/cdrom`文件夹下：

```bash
# mount -o loop /mnt/iso/CentOS-6.5-x86_64-bin-DVD1.iso /mnt/cdrom/
```

使用`df -h`命令查看是否挂载成功：

```bash
# df -h
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup-lv_root
                      146G   20G  119G  15% /
tmpfs                 1.9G     0  1.9G   0% /dev/shm
/dev/sda1             485M   38M  423M   9% /boot
/dev/mapper/VolGroup-lv_home
                      7.9G  146M  7.4G   2% /home
/mnt/iso/CentOS-6.5-x86_64-bin-DVD1.iso
                      4.2G  4.2G     0 100% /mnt/cdrom
```

## 配置yum指向本地挂载

yum源以`*.repo`格式文件，存放在`/etc/yum.repos.d/`目录下，在配置前先把该文件夹下的所有文件备份。

配置自己的`centos.repo`，内容如下：

```bash
[base]
name=CentOS
baseurl=file:///mnt/cdrom
enabled=1
gpgckeck=0
gpgkey=file:///mnt/cdrom/RPM-GPG-KEY-CentOS-6
```
