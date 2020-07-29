---
layout: post
title: "ubuntu中挂载磁盘"
date:  2020-7-29 12:06:00  
categories: linux
---

在购买云服务器之后，后面由于业务的需要可能需要扩展磁盘，这个时候可以通过购买运营商的挂载盘来实现。

首先需要在运营商的服务中购买挂载盘(需要注意挂载盘的适配机房区域，否则可能不能使用)。

1. 使用命令`fdisk -l`查看磁盘的状态。

![https](/assets/image/fdisk_l.png)

可以看到，新的挂载盘的设备名是`/dev/vdb`。

2. 格式化该磁盘设备，`mkfs.ext4 /dev/vdb`

3. 找到一个挂载点，挂载该磁盘，这里我们使用`/meida/data`目录作为挂载点。`mount /dev/vdb /media/data`

##### 手动挂载的磁盘在下次系统重启的时候需要手动再次挂载，我们可以通过配置让磁盘自动挂载。

编辑/etc/fstab文件，添加下面行

`UUID=e2f3c605-efdf-46f0-bf95-3f7f20f6ea66 /media/data ext4 defaults 0 1`

格式如下：

`<file system> <mount point>   <type>  <options>       <dump>  <pass>`

`file system`: 设备的符号，可以使用blkid命名查看UUID或者直接使用/dev/vdb

`mount point`: 挂载点，这里是/media/data

`type`: 文件系统

`opitions`:挂载参数，一般是defaults

`dump`：磁盘检测，默认为0

`pass`: 磁盘检查，默认为0,不需要检查

配置完成之后，可以使用`mount -a`命令验证一下是否配置正确

