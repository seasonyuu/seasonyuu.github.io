---
title: "解压 Android 镜像"
date: 2020-12-01T17:16:06+08:00
categories:
- AOSP 准备工作
keywords:
- vendor 分区
- system 分区
- product 分区
---

<!-- toc -->

> 以下内容适用于 Android 10 的固件镜像解压及挂载

## 为什么需要解压镜像

对于一些 OEM 定制的私有库，通常开源的 ROM 源码中是不可能包含的，而设备本身要运行起三方系统时，这些厂商定制的私有内容就需要作为预编译项添加到编译过程中。

在 Project-Treble 被推行之后，Android 系统中指定了 `product` 分区和 `vendor` 分区来存放一些厂商指定的、设备定制的内容，通常我们需要从 `system` 分区和这两个分区中拉取一些指定的文件。

## 开始

我们需要的镜像文件有三个： `system.img`, `vendor.img` 和 `product.img`。

通常我们从官方网站中下载线刷包会更便于提取对应的镜像文件，而有些时候可能并不存在线刷宝提供下载，那我们就要自己转换得到对应的 `*.img` 文件。

### 从卡刷包提取 img

在我忘了哪个 Android 版本之后开始，卡刷包中的镜像分成了两个文件，例如如果是 system 的话

```
system.transfer.list
system.new.dat
```

其中 `system.new.dat` 可能是 `system.new.dat.br` ，这种情况下需要使用 Google 官方工具 [`brotli`](https://github.com/google/brotli) 进行转换

Debian 系可以直接这样安装

```shell
sudo apt update
sudo apt install brotli
```

安装成功后，执行

```shell
brotli -d system.new.dat.br
```

执行完则可以得到 `system.new.dat` 文件了。

得到这三个文件之后，使用 [dat2img](https://github.com/danielmmmm/dat2img) （要求 `python` 环境）转换后即可得到 `system.img` 镜像文件

```shell
./sdat2img.py system.transfer.list system.new.dat system.img
```

上面以 `system.img` 举例，`vendor` 和 `product` 参考替换对应文件名即可。

## 分区介绍

得到各个镜像后，Linux 中需要将镜像 mount 到某个文件夹，我们首先 mount `system` 分区。

挂载成功后，我们会看根目录中又存在一个 `system` 目录，这个目录才是我们需要的最终目录。

![](/images/factory_image_extracted.png)

在刷机之后，通过 root 你也可以看到实际上 `system` 分区种就是这样的结构了，当然了，其中的 `vendor` 和 `product` 实际上 link 到对应的分区而已。

## 配合 `proprietary-files.txt` 使用

在挂在镜像成功之后，我们还需要将这些文件全都拉出来，形成一个和 ROM 中目录结构类似的目录。目录可以参考前面的截图。

device tree 中通常需要准备好 `setup-makefiles.sh` 和 `extract-files.sh` 脚本，如果这两份脚本是确认可用了的话，以前面截图的目录位置为例子。

```sh
./setup-makefiles.sh /miui_12.0.9/system
./extract-files.sh /miui_12.0.9/system
```

这样执行后，脚本就会根据 `proprietary-files.txt` 定义的规则创建或更新对应的 vendor 仓库了。