---
title: 宇树机器人G1便携开发指北：基于树莓派的环境搭建
description: 本文档介绍了如何在树莓派上进行宇树机器人G1的便携开发，涵盖环境配置、代码获取及基本操作步骤。
slug: unitree-g1-develop-on-raspberry-pi
date: 2026-01-24 08:00:00+0000
image: logo.jpg
categories:
  - linux
tags:
  - unitree
  - raspberry-pi
  - robotics
  - development
---

## 引言

如果要调试 unitree-g1-edu 机器人的话，

肯定是看过他们的官方文档的。

快速开发文档在这里：

[G1_developer/快速开发](https://support.unitree.com/home/zh/G1_developer/quick_development)

环境要求倒没撒，基于Ubuntu 20.04 or 22.04, 

装上依赖库，

基本是有手就能行的操作。

只是，

如果你想去调试一下G1机器人的话，

官方的方案是，大概是这样的。


[演示视频](https://oss-global-cdn.unitree.com/static/0818dcf7a6874b92997354d628adcacd.mp4)。

- 拖一根网线从G1机器人颈部网口接入，连接到你的Ubuntu主机

- 给Ubuntu主机网络设定固定IP, 通过lan口做网络通讯

大概如下：

![](/img/post/unitree-g1-develop-on-raspberry-pi/1.lan.png)

```text

建议新用户使用网线与转接线将用户计算机接入 G1 交换机，

并将与机器人通信的网卡设置在 192.168.123.X 网段下，

推荐使用 192.168.123.99。

```

倒没什么问题，只是拖着一个网线，怎么看都是麻烦。

有什么别的办法么？

思索了一下，直接用“树莓派”做个主控机器连接G1网络，

同时树莓派连接到本地局域网，

那不就可以直接远程是操控机器了？

又研究了一下，G1-EDU网卡旁边还有三个供电口，

分别是12V 、 24V 、 48V，

也足够给树莓派供电了（树莓派是5V，买个电压转换器即可）。

就这个电压，给更多的设备供电也没什么问题。

例如：灵巧手？很是有趣了。

## “树莓派” 硬件环境准备

转压器(EVEPS直流降压器) + 电源线，

![](/img/post/unitree-g1-develop-on-raspberry-pi/2.EVEPS.png)

EVEPS直流降压器

![](/img/post/unitree-g1-develop-on-raspberry-pi/3.to_len.png)

电源线

加起来三十来块搞掂，

淘宝直接买就可以。


3M胶布固定“树莓派”到机器背面，电源线接上去，

网线从脖子后面出来，接到树莓派上面，

最后，套一个衣服到G1上面，

一切都可以藏起来了。

## Python 动作调试

本地先把他们家的unitree_sdk2py装好，

然后就可以试试调试一下G1了~

```python
import time
import sys
from unitree_sdk2py.core.channel import ChannelSubscriber, ChannelFactoryInitialize
from unitree_sdk2py.g1.arm.g1_arm_action_client import G1ArmActionClient
from unitree_sdk2py.g1.arm.g1_arm_action_client import action_map
from dataclasses import dataclass

@dataclass
class TestOption:
    name: str
    id: int

option_list = [
    TestOption(name="release arm", id=0),     
    TestOption(name="shake hand", id=1),    
    TestOption(name="high five", id=2), 
    TestOption(name="hug", id=3), 
    TestOption(name="high wave", id=4),
    TestOption(name="clap", id=5), 
    TestOption(name="face wave", id=6),
    TestOption(name="left kiss", id=7),
    TestOption(name="heart", id=8),
    TestOption(name="right heart", id=9),
    TestOption(name="hands up", id=10),
    TestOption(name="x-ray", id=11),
    TestOption(name="right hand up", id=12),
    TestOption(name="reject", id=13),
    TestOption(name="right kiss", id=14), 
    TestOption(name="two-hand kiss", id=15),  
    
]

class UserInterface:
    def __init__(self):
        self.test_option_ = None

    def convert_to_int(self, input_str):
        try:
            return int(input_str)
        except ValueError:
            return None

    def terminal_handle(self):
        input_str = input("Enter id or name: \n")

        if input_str == "list":
            self.test_option_.name = None
            self.test_option_.id = None
            for option in option_list:
                print(f"{option.name}, id: {option.id}")
            return

        for option in option_list:
            if input_str == option.name or self.convert_to_int(input_str) == option.id:
                self.test_option_.name = option.name
                self.test_option_.id = option.id
                print(f"Test: {self.test_option_.name}, test_id: {self.test_option_.id}")
                return

        print("No matching test option found.")

def shake_hand(network_interface):
    ChannelFactoryInitialize(0, network_interface)
    armAction_client = G1ArmActionClient()  
    armAction_client.SetTimeout(10.0)
    armAction_client.Init()
    time.sleep(0.5)
    armAction_client.ExecuteAction(action_map.get("shake hand"))
    time.sleep(2)
    armAction_client.ExecuteAction(action_map.get("release arm"))
    print("Shaked hand and released arm.")

def release_arm(network_interface):
    ChannelFactoryInitialize(0, network_interface)
    armAction_client = G1ArmActionClient()  
    armAction_client.SetTimeout(10.0)
    armAction_client.Init()
    time.sleep(0.5)
    result = armAction_client.ExecuteAction(action_map.get("release arm"))
    print(f"Released arm with result: {result}")


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print(f"Usage: python3 {sys.argv[0]} networkInterface")
        sys.exit(-1)

    print("WARNING: Please ensure there are no obstacles around the robot while running this example.")
    input("Press Enter to continue...")

    ChannelFactoryInitialize(0, sys.argv[1])

    test_option = TestOption(name=None, id=None) 
    user_interface = UserInterface()
    user_interface.test_option_ = test_option

    armAction_client = G1ArmActionClient()  
    armAction_client.SetTimeout(10.0)
    armAction_client.Init()
    

    # actionList = armAction_client.GetActionList()
    # print("actionList\n",actionList)

    print("Input \"list\" to list all test option ...")
    while True:
        user_interface.terminal_handle()

        print(f"Updated Test Option: Name = {test_option.name}, ID = {test_option.id}")

        if test_option.id == 0:
            armAction_client.ExecuteAction(action_map.get("release arm"))
        elif test_option.id == 1:
            armAction_client.ExecuteAction(action_map.get("shake hand"))
            time.sleep(2)
            armAction_client.ExecuteAction(action_map.get("release arm"))
        elif test_option.id == 2:
            armAction_client.ExecuteAction(action_map.get("high five"))
            time.sleep(2)
            armAction_client.ExecuteAction(action_map.get("release arm"))
        elif test_option.id == 3:
            armAction_client.ExecuteAction(action_map.get("hug"))
            time.sleep(2)
            armAction_client.ExecuteAction(action_map.get("release arm"))
        elif test_option.id == 4:
            armAction_client.ExecuteAction(action_map.get("high wave"))
        elif test_option.id == 5:
            armAction_client.ExecuteAction(action_map.get("clap"))
        elif test_option.id == 6:
            armAction_client.ExecuteAction(action_map.get("face wave"))
        elif test_option.id == 7:
            armAction_client.ExecuteAction(action_map.get("left kiss"))
        elif test_option.id == 8:
            armAction_client.ExecuteAction(action_map.get("heart"))
            time.sleep(2)
            armAction_client.ExecuteAction(action_map.get("release arm"))
        elif test_option.id == 9:
            armAction_client.ExecuteAction(action_map.get("right heart"))
            time.sleep(2)
            armAction_client.ExecuteAction(action_map.get("release arm"))
        elif test_option.id == 10:
            armAction_client.ExecuteAction(action_map.get("hands up"))
            time.sleep(2)
            armAction_client.ExecuteAction(action_map.get("release arm"))
        elif test_option.id == 11:
            armAction_client.ExecuteAction(action_map.get("x-ray"))
            time.sleep(2)
            armAction_client.ExecuteAction(action_map.get("release arm"))
        elif test_option.id == 12:
            armAction_client.ExecuteAction(action_map.get("right hand up"))
            time.sleep(2)
            armAction_client.ExecuteAction(action_map.get("release arm"))
        elif test_option.id == 13:
            armAction_client.ExecuteAction(action_map.get("reject"))
            time.sleep(2)
            armAction_client.ExecuteAction(action_map.get("release arm"))
        elif test_option.id == 14:
            armAction_client.ExecuteAction(action_map.get("right kiss"))
        elif test_option.id == 15:
            armAction_client.ExecuteAction(action_map.get("two-hand kiss"))

        time.sleep(1)
```