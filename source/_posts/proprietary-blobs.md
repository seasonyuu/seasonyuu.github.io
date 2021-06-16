---
title: "proprietary-files.txt 编写"
date: 2020-12-01T17:16:06+08:00
categories:
- AOSP
keywords:
- vendor
- proprietary-files.txt
tags:
- AOSP 准备工作
---

{% alert info %}
翻译自 Lineage OS 的 [Working with proprietary blobs](https://wiki.lineageos.org/proprietary_blobs.html)，并增加了部分描述
{% endalert %}

<!-- toc -->

## 使用专有 blob

所有设备在其 device tree 中都应包含专有 blob 列表 `proprietary-files.txt`。此文件用于创建 vendor 库，以通过从运行 LineageOS 的最新系统的设备和/或系统转储中提取 blob 来构建设备。尽管你可能会遇到同时具有 `proprietary-files.txt` 、 `proprietary-files-qc.txt` 甚至更多 blob 描述文件的设备，但通常设备仅需要一个 blob 列表。把一个文件写成多个文件不是必要的，你能看到的分写的文件很可能只是历史遗留的。

首先，从 `vendor/lineage/build/templates` 中把 `extract-files.sh` 和 `setup-makefiles.sh` 复制到你的 device tree 中，编辑其中的三个必要字段（设备，供应商和 copyright 年份）。

假设是小米 10 ，代号为 `umi`，在 2020 年初始化 device tree，那么在两个 `.sh` 文件中找到对应字段替换：

```
DEVICE=umi
VENDOR=xiaomi
INTITIAL_COPYRIGHT_YEAR=2020
```

`proprietary-files.txt` 的内容是一个Blob列表，其中可以包含注释（行前声明 `#` ）。每行 blob 声明的形式为：

```
[-]source[:destination][|sha1sum]
```

每个路径声明都是基于 `system` 分区的相对路径。每一行声明代表该 blob 来自 `vendor/lib/libblob.so` ，要安装到 `/system/vendor/lib/libblob.so` 。

所有可能用到的声明语法如下

```
libbasic.so
-libneeded-to-build.so
libsource.so:libdestination.so
-libneeded-source.so:libneeded-destination.so
libstock.so|sha1sum
-libstock-needed.so|sha1sum
-libstock-source.so:libstock-destination.so|sha1sum
```

如果声明以破折号（`-`）开头，则表示 vendor 库将声明一个模块来提供此 blob 。如果在 Android 中构建另一个组件时需要用到这个 blob，则需要这样声明。如果破折号不存在，那么将仅在构建过程中复制 blob。

如果声明中 `source` 和 `destination` 之间包含冒号 (`:`)，则提取程序将检查目标路径是否存在。如果目标路径存在则优先采用，不然就把源文件提取出来保存到目标路径。这么做使得你既可以从原厂系统中来提取 blob ，也可以从已有的、其他参考设备的 Lineage 固件中提取 blob。

`sha1sum` 字段代表的是，我们要提取的 blob 的版本校验码。如果现有 vendor 库中该 blob 的 `sha1sum` 和声明中的一致，则不论你到底新提取的 blob 来自什么地方都不会更新该 blob；否则，提取将正常进行。

{% alert info %}
**提示** 如果你之前已经从其他参考设备中拉取 blob 到 vendor 中，然后又想要从官方镜像中更新部分 blob，`sha1sum` 就十分重要了。如果把任何从其他设备拉取的 blob 或已通过 hex 编辑过（非官方） blob 的 sha1sum 都声明好，那么你始终可以通过工具来更新原厂 blob，而不必担心丢失你已经定制过的 blob。
{% endalert %}

{% alert info %}
提示： 如果 blob 是进行过 hex 编辑的或者是从其他设备中拉出的，那么你需要在 `proprietary-files.txt` 列表中声明 `sha1sum` 。最好花点时间记录一下三方 blob 的来源或描述 hex 编辑的步骤。
{% endalert %}

如果你从已经 odexed 化的 system 分区（也就是生成了 boot.oats）中提取 app（`*.apk`）或 jar 库，并且这个 app / jar 中还不包含 `classes.dex` ，那么在提取时会对其进行 de-odexed。

## 提取 blob 文件

首先你需要一个包含了 system 分区完整内容的文件夹，通常形如 

```
- system
  ...
  - product
  - vendor
  ...
```

{% alert info %}
如何解压出 system 目录见 [此章节](../extract-system)
{% endalert %}

使用以下命令则会将 `proprietary-files.txt` 声明的 blob 拉到前文声明的 `vendor/{VENDOR}/{DEVICE}` 中的 `proprietary` 目录下。

```shell
./setup-makefiles.sh /SOME_PATH/system
./extract-files.sh /SOME_PATH/system
```

<!-- more -->