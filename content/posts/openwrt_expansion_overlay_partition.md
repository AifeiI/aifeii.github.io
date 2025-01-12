+++
title = '硬路由器上的 Openwrt 扩展 Overlay 分区'
date = 2025-01-12T22:59:19+08:00
draft = false
+++

## 前提
---
本文基于 `OpenWRT 23.05.5`，路由器是 `ramips/mt7621` 平台

需要用到的命令

- fdisk
- mkfs.ext4
- mount

需要用到的包（ipa）
- fdisk
- kmod-usb-storage
- e2fsprogs
- kmod-fs-ext4
- block-mount

## 操作
---
### 1. 安装必要的包

```shell
# 更新包列表索引
opkg update
# 安装分区工具与USB设备识别驱动、ext4文件系统
opkg install fdisk kmod-usb-storage e2fsprogs kmod-fs-ext4
# 安装挂载点配置，可以在 Luci 上进行挂载操作
opkg install block-mount

# 安装完之后需要直接重启，否则 USB 设备无法立即被识别
reboot
```

### 2. 对 USB 进行分区

首先确认 USB 设备正确识别

```shell
fdisk -l

# 查看输出内容末尾，是否有对应容量的设备被识别出来：
# Device    Start      End  Sectors  Size Type
# /dev/sda1   2048 15542271 15540224  7.4G Microsoft basic data
```

开始分区
- 按照我自己的需求，是需要分出三个分区：扩展 Overlay 空间的分区（ext4）、扩展自定义数据分区（ext4）、交换分区（swap，可选）
- 添加 SWAP 分区是用于缓解内存较为紧张的场景，但如果USB接口是2.0，可能效果就不明显，甚至出现反效果
- 自定义数据，可以包含一些对存储需求较大的程序，例如：aria2、smartDNS、ftp等等

```shell
# 假定识别出来的设备是 sda，那么就开始对整个设备进行分区与格式化，也可以按照自己需要进行调整
fdisk /dev/sda
```

`fdisk` 是提供交互式输入进行分区，所以可以按照提示与帮助进行操作
假如设备之前存在分区，现在需要删除重新划分分区，则在命令输入：`d`，然后回车，会显示：
```shell
Command (m for help): d
Selected partition 1
Partition 1 has been deleted.
```

然后输入 `n` 则表示创建新的分区

```shell
Command (m for help): n
# 这里是表示标记当前分区序号，之后会在 /dev 目录下看到对应的设备，例如：/dev/sda1
Partition number (1-128, default 1): 1
# 这里是选择分区起始扇区，按照默认即可直接回车
First sector (34-15544286, default 2048):
# 这里是选择分局大小，输入 +2G 表示创建 2GB 大小的空间
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-15544286, default 15542271): +2G
Created a new partition 1 of type 'Linux filesystem' and of size 2 GiB.
Partition #1 contains a vfat signature.
# 之前有创建过分区会提示这个是否移除签名，如果需要保留原有的文件，则不要输入 Y
Do you want to remove the signature? [Y]es/[N]o: Y
The signature will be removed by a write command.

# 按照自己需求划分好分区后，或者所有分区都划分完毕后，可以直接输入 w 命令保存分区表
Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

按照自己需求设置好分区表后，就可以进行格式化

```shell
# 将第一个分区格式化为 ext4 文件系统，执行完需要记录一下 UUID ，便于后面进行挂载管理
mkfs.ext4 /dev/sda1
# 将第二个分区也格式化为 ext4 文件系统，同上
mkfs.ext4 /dev/sda2
# 将第三个分区格式化为 swap 文件系统
mkswap /dev/sda3
# 启用交换分区
swapon /dev/sda3
```

开始正题，扩展 Overlay 空间

```shell
# 先将创建的扩展分区挂载
mkdir /mnt/extspace
mount /dev/sda1 /mnt/extspace

# 然后是将原有的 Overlay 分区内的数据打包复制到新的分区上
# 先确认原有的 Overlay 目录，部分设备在 OpenWRT 上的 Overlay 分区挂载点是有差异的，通过 df -hl 命令进行确认
tar -C /overlay -cvf - . | tar -C /mnt/extspace -xf -

# 等执行完之后执行数据存盘以及取消新扩展分区的挂载
sync
umount /mnt/extspace
```

### 3. 配置自动挂载规则

在 Luci 上访问 `系统（System） -> 挂载点（mounts）`

#### 3.1 添加挂载点

##### 替换 Overlay
1. 在`挂载点`栏目点击`添加`
2. 勾选`已启用`
3. 选择`根据UUID匹配`
4. 然后在下拉列表中，根据之前进行格式化所记录的UUID，找到用于扩展 Overlay 分区的项
5. 在`挂载点`中选择`作为外部Overlay分区使用（/overlay）`
6. 点击保存

##### 挂载自定义数据分区
1. 在`挂载点`栏目点击`添加`
2. 勾选`已启用`
3. 选择`根据UUID匹配`
4. 然后在下拉列表中，根据之前进行格式化所记录的UUID，找到用于自定义数据分区的项
5. 在`挂载点`中，手动输入`/mnt/USB`
6. 点击保存

##### 添加交换分区

1. 在 `交换分区` 栏目点击 `添加`
2. 勾选 `已启用`
3. 选择 `设备`
4. 然后在下拉列表中，找到最后创建的 sda3 设备
6. 点击保存

执行完以上步骤后，点击最下方的`保存并应用`

### 4. 配置自定义数据分区与对应程序存储目录的关联

在 /etc/rc.local 里面增加自动添加软连接的操作：

```shell
# 将需要扩展存储空间的程序目录，通过软连接方式，转到自定义数据分区上
ln -s /mnt/USB/aria2 /tmp/etc/aria2
```

添加之后进行重启，然后就可以愉快的开始折腾

## 验证
---

查看 Luci 的概览（/cgi-bin/luci/admin/status/overview），就可以看到存储空间被拓展到 2GB，内存栏目也出现了交换分区

同时可以使用 `df -hl` 命令查看新增的分区挂载情况