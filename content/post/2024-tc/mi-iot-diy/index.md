---
title: 一行代码控制你的米家设备
description: 米家,智能家居,自动化,HomeAssistant
slug: one-line-code-control-mi-iot-diy
date: 2024-06-27 08:00:00+0000
image: mi-iot-diy.png
categories:
  - Zhihu
tags:
  - zhihu
  - 2024
---

## 前言

大概在去年还是前年的时候，因为手机换成了iOS，没有小爱同学了之后，

只能使用米家App来控制米家设备实在蛋疼，

于是那阵子就折腾过 HomeAssistant 方案， 于是写过这样的文章。

[知乎专栏：Home Assistant 智能家居系统搭建指南（米家+Siri）](https://zhuanlan.zhihu.com/p/444212384)

这个方案配合HomePod音响，基本满足了我的需求了。

后面家里基本是就是各种设备都有一点，HomePod基本就是全屋中控的存在了。

Home Assistant 在家里的主机24小时跑着，基本也没出什么岔子，也就很久继续折腾了。

## 又一个“新需求”

年初有个大佬找过我，咨询能不能基于现在的米家或者其他硬件设备，自己做公寓的智能家居方案。

基于上面的一些经验，我也就给了一些建议，但是也没深入去研究。

那阵子让某二十年老师去折腾了一下DIY控制米家设备的方案，

这朋友的进度不那么理想，最后也没出什么成果。

只是那阵子知道了有一个 

[github.com/rytilahti/python-miio](https://github.com/rytilahti/python-miio) 的好工具。

- Python library & console tool for controlling Xiaomi smart appliances
- 一个用于控制小米智能家居设备的Python库和控制台工具

很有趣，Mark 一下。

## 最近的实践

最近嘛，又因为一些奇奇怪怪的需求，需要“正儿八经”来折腾米家设备控制方案了。

最早做方案验证的时候，甚至是用Appium来模拟操作米家App

但是嘛，卡顿，慢，不稳定...

到了这两周开始进入正式的实践阶段，决定还是用 python-miio 来搞了，

虽然Hack，但是肯定比Appium来的稳定。

So，让我们开始吧。

## python-miio 起手

首先，我们先在本地安装 python-miio

```bash
# 切记，请不要直接安装 pip install python-miio 
# 默认的Release版本是0.5.12，22年的版本了
pip install python-miio==0.6.0.dev0

# 或者直接从Github上安装最新的版本

pip install git+https://github.com/rytilahti/python-miio.git

```

这个安装好了之后，本地命令行环境应该有了 miiocli 这个命令行工具了。

```bash
➜  ~ miiocli --help
Usage: miiocli [OPTIONS] COMMAND [ARGS]...

Options:
  -d, --debug
  -o, --output [default|json|json_pretty]
  --version                       Show the version and exit.
  --help                          Show this message and exit.
```

正常情况下，我们应该可以通过 miiocli 来控制我们的米家设备了。

首先，我们需要找到我们的设备的IP地址和Token。

后面的所有操作，都是基于这两个信息来进行的。

### 获取设备IP地址和Token

```sh
➜  ~ miiocli cloud
Username: 你的小米账号
Password: 你的密码
== Xiaomi Smart Camera C400 (Device online ) ==
	Model: chuangmi.camera.039a01
	Token: XXX
	IP: 192.168.31.170 (mac: 94:F8:27:05:5B:AA)
	DID: 1029838640
	Locale: cn
== Xiaomi智能摄像机 (Device online ) ==
	Model: chuangmi.camera.029a02
	Token: XXX
	IP: 192.168.31.233 (mac: 60:7E:A4:C0:E6:26)
	DID: 525571884
	Locale: cn
== 智能插座-Ubuntu (Device online ) ==
	Model: chuangmi.plug.212a01
	Token: XXX
	IP: 192.168.31.141 (mac: 58:B6:23:EB:11:EF)
	DID: 433609370
	Locale: cn
```

这里基本能看到自己账号下的设备了，然后我们可以通过 miiocli 命令来控制这些设备了。

## 控制设备

```sh
# 这是一个摄像头设备
➜  ~ miiocli device --ip 192.168.31.170 --token xxxx info
Running command info
Model: chuangmi.camera.039a01
Hardware version: Linux
Firmware version: 5.1.6_0420
Supported using: GenericMiot
Command: miiocli genericmiot --ip 192.168.31.170 --token xxxx
Supported by genericmiot: True

```

这里需要比较关注的东西是 Supported by genericmiot ,

有这个的话，基本就可以通过 miiocli genericmiot 来控制这个设备了。


```txt
# Supported by genericmiot: True
Note that the command field which gives you the direct command to use for controlling the device. 

If the device is supported by the genericmiot integration as stated in the output, 

you can also use miiocli genericmiot for controlling it.

```

以下文档从官方文档中摘录翻译。

### 控制现代（MIoT）设备

大多数现代(MIoT)设备都会自动受到 genericmiot 集成的支持。

在内部，它使用("miot spec")文件来了解支持的功能，如传感器、设置和操作。


- [https://home.miot-spec.com/spec/xiaomi.controller.86v1](https://home.miot-spec.com/spec/xiaomi.controller.86v1)

- [米家产品库（非官方）](https://home.miot-spec.com/)

此设备型号特定文件将在您首次使用 genericmiot 集成时下载（并在本地缓存）。

所有支持设备的功能都可以使用常见命令来控制：

status （显示设备状态）、 set （更改设置）、 actions 列出可用操作

和 call 执行操作。

### 设备状态

执行 status 将显示当前设备状态，以及设置的可接受值（标有访问 RW ）：

```shell
miiocli genericmiot --ip 127.0.0.1 --token 00000000000000000000000000000000 status

Service Light (light)
        Switch Status (light:on, access: RW): False (<class 'bool'>, )
        Brightness (light:brightness, access: RW): 60 % (<class 'int'>, min: 1, max: 100, step: 1)
        Power Off Delay Time (light:off-delay-time, access: RW): 1:47:00 (<class 'int'>, min: 0, max: 120, step: 1)
```

### 更改设置

要更改设置，您需要提供设置的名称（例如，在上面的示例中 light:brightness ）：

```shell
 miiocli genericmiot --ip 127.0.0.1 --token 00000000000000000000000000000000 set light:brightness 0

 [{'did': 'light:brightness', 'siid': 2, 'piid': 3, 'code': 0}]

```

 使用动作
大多数设备还将提供操作：

```sh
miiocli genericmiot --ip 127.0.0.1 --token 00000000000000000000000000000000 actions

Light (light)
        light:toggle            Toggle
        light:brightness-down   Brightness Down
        light:brightness-up     Brightness Up
```

这些可以使用 call 命令执行：

```sh
miiocli genericmiot --ip 127.0.0.1 --token 00000000000000000000000000000000 call light:toggle

{'code': 0, 'out': []}
```

使用 miiocli genericmiot --help 获取更多可用命令。

详细的文档在这里：

- [](https://github.com/rytilahti/python-miio?tab=readme-ov-file#controlling-modern-miot-devices)

## 一行代码控制你的米家设备
