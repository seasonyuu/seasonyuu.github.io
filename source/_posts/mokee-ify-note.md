---
title: "小米10Pro适配魔趣笔记"
date: 2020-12-01T17:16:06+08:00
categories:
- AOSP
keywords:
- Mokee
tags:
- AOSP 编译
---

> [魔趣 ROM](https://download.mokeedev.com/) 基于 Android 开源项目 (AOSP) 二次开发的三方定制 ROM。
> 
> 选择魔趣作为主要的学习方向，是因为魔趣本地化定制的部分、以及中国开发者更加活跃，魔趣在国内的开发氛围更为友善。

首先，为自己定下一些小目标

* [x] 指纹
* [x] NFC
* [x] 声音
* [x] 蓝牙
* [ ] 微信指纹支付
* [ ] 支付宝指纹支付
* [x] 帧率切换
* [ ] Others

<!-- toc -->

我的 CMI 开源代码

[device tree](https://github.com/seasonyuu/device_xiaomi_cmi)

[vendor](https://github.com/seasonyuu/vendor_xiaomi_cmi)

## 20201212 小记

基本确定了第一版搞定了。小白入场基本都是靠前人肩膀了。

小米 10 系列机型有很多个大佬在维护( [markakash](https://github.com/markakash), [Manuel](https://github.com/dataoutputstream), [VladV1V](https://github.com/VladV1V) )，我维护的版本基本基于 markakash 的 PE ten 分支和 Manuel 的 lineage 17.1 分支。

前人的肩膀还是比较坚实的，当我编译出包时 NFC、蓝牙这些东西就已经好了。

这里说几个自己学到的东西。

### 底包

由于目前小米 865 系列主要还是通过预编译内核形式来编译三方 ROM ，因此强调底包还是比较重要的。

通过 [XINGRZ](https://github.com/xingrz) 大佬维护的 [Web 解包工具](https://boot.mokeedev.com/) 可以方便地将 `boot.img` 中的 `kernel.img` 和 `dtb.img` 提取出来，替换进 device 的 prebuilt 目录中，`dtbo.img` 的话则在线刷包的解压包中可以找到。

此外还有 vendor 的部分也需要更新，主要还是通过解压 system, product, vendor 三个镜像来处理，具体操作可以通过 [这篇博客](../extract-system) 中的介绍学习。

### Sepolicy

由于使用了预编译内核，last_kmsg 中的配置似乎是炸的，所以我们需要把 AOSP 中关于这部分的定义都先去掉。否则会出现开机看不到开机动画直接重启到 Recovery 的现象。

参考 [这条提交](https://github.com/dataoutputstream/lineage_system_sepolicy/commit/5d2c2db26798ca067c213b13b7d063a33eaceeff) Pick 一下即可。

### 信号

当不更新 vendor 时信号是好的，但是一旦更新了 vendor 信号就炸了，vendor 也未知了。

有一个 `qti-telephony-common.jar` 的包是负责基带通信的。但是如果更新成 MIUI 自带的 jar 是不行的。应该是 MIUI 进行了定制，所以这里我沿用了 markakash vendor 仓库中的 jar，并在 `proprietary-files.txt` 中增加 sha1sum 避免下次同步时此文件被覆盖。

> 另外还有一个情况，是之前我用 umi 的 device 修改来跑 cmi 的时候遇到的。如果 device 没有声明对的话，也有可能导致基带未知。

### 声音破音

这个主要出现在 markakash 维护的版本中，而 Manuel 的版本并没有这个问题。通过一系列搜索和验证，主要的根由是从 MIUI vendor 中拉出的 soundfx 库应该是小米自己修改过的版本，使用 Manual 的版本（虽然不知道哪来的）就不会出现这个问题了，与此同时，我们也同步修改 `proprietary-files.txt` 进行标记。

### 指纹支付

微信和支付宝指纹支付其实我是同步开始调研的，但是目前只有支付宝的在之前编译 RR 时是通了的，到魔趣这边时还没调通。

两家方案的关键字分别是 `IFAA` 和 `Soter` ，直接 Google 相关关键字其实就可以搜到很多参考资料了。

[这个提交](https://review.lineageos.org/c/LineageOS/android_device_xiaomi_sm6150-common/+/274232/6) 中可以看到相关的一些实现，主要就是定义了 `org.ifaa.android.manager` 的模块来调用支付宝和微信的指纹支付。

对于小米而言，另外还涉及了 `Mlipay` 这个服务，小米对两家指纹支付方案进行了封装。我们需要在 `propriety-files.txt` 中定义好 `Mlipay` 和 `Soter` 相关文件的引用。

```
# Mlipay
vendor/bin/mlipayd@1.1
vendor/etc/init/vendor.xiaomi.hardware.mlipay@1.1-service.rc
vendor/lib64/libmlipay.so
vendor/lib64/libmlipay@1.1.so
vendor/lib64/vendor.xiaomi.hardware.mlipay@1.0.so
vendor/lib64/vendor.xiaomi.hardware.mlipay@1.1.so
vendor/lib64/vendor.xiaomi.hardware.mtdservice@1.0.so
vendor/lib64/vendor.xiaomi.hardware.mtdservice@1.1.so
vendor/lib64/vendor.xiaomi.hardware.mtdservice@1.2.so

# Soter
-vendor/app/SoterService/SoterService.apk
vendor/bin/hw/vendor.qti.hardware.soter@1.0-service
vendor/etc/init/vendor.qti.hardware.soter@1.0-service.rc
vendor/lib64/hw/vendor.qti.hardware.soter@1.0-impl.so
vendor/lib64/vendor.qti.hardware.soter@1.0.so
```

然后就开始最关键的地方，配置 `sepolicy`。

```
// TO BE CONTINUE
```

### 屏幕刷新率切换

AOSP 直接就是支持配置刷新率的，但是设置界面的话就需要我们自己撸了。如果不想撸界面的话，可以直接配置一个默认的刷新率。

在 framework 的 overlay 中，配置

```xml
<!-- 文件位于 framework/base/res/res/values/config.xml -->

<!-- The default peak refresh rate for a given device. Change this value if you want to allow
    for higher refresh rates to be automatically used out of the box -->
<integer name="config_defaultPeakRefreshRate">90</integer>
```