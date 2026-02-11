---
title: "Control Your Mi Home Devices with One Line of Code"
description: "Mi Home, Smart Home, Automation, HomeAssistant"
slug: one-line-code-control-mi-iot-diy
date: 2024-06-27 08:00:00+0000
image: mi-iot-diy.png
categories:
  - Zhihu
tags:
  - zhihu
  - 2024
---

## Preface

Around last year or the year before, after switching phone to iOS, no Xiaoai classmate anymore, using Mi Home App to control Mi Home devices was really painful. So at that time, I tinkered with HomeAssistant solution, and wrote such an article.

[Zhihu Column: Home Assistant Smart Home System Setup Guide (Mi Home + Siri)](https://zhuanlan.zhihu.com/p/444212384)

This solution with HomePod speaker basically met my needs. Later, home basically had various devices, HomePod was basically the whole house central control.

Home Assistant running on home host 24/7, basically no issues, so didn't tinker for a long time.

## Another "New Requirement"

At the beginning of the year, a big shot contacted me, asking if based on current Mi Home or other hardware devices, can do apartment smart home solution.

Based on the above experience, I gave some suggestions, but didn't research deeply.

At that time, let a certain 20-year teacher tinker with DIY control Mi Home devices solution, this friend's progress not ideal, finally no results.

Just at that time knew there is a [github.com/rytilahti/python-miio](https://github.com/rytilahti/python-miio) good tool.

- Python library & console tool for controlling Xiaomi smart appliances
- A Python library and console tool for controlling Xiaomi smart home devices

Interesting, Mark it.

## Recent Practice

Recently, again because of some strange requirements, need to "seriously" tinker with Mi Home device control solution.

Earliest when doing solution verification, even used Appium to simulate operating Mi Home App, but lag, slow, unstable...

To these two weeks entering formal practice stage, decided to use python-miio, although Hack, but definitely more stable than Appium.

So, let's start.

## Getting Started with python-miio

First, install python-miio locally

```bash
# Remember, do not directly install pip install python-miio
# Default Release version is 0.5.12, 2022 version
pip install python-miio==0.6.0.dev0

# Or directly install latest version from Github

pip install git+https://github.com/rytilahti/python-miio.git
```

After installing, local command line should have miiocli command line tool.

```bash
➜  ~ miiocli --help
Usage: miiocli [OPTIONS] COMMAND [ARGS]...

Options:
  -d, --debug
  -o, --output [default|json|json_pretty]
  --version                       Show the version and exit.
  --help                          Show this message and exit.
```

Normally, we should be able to control our Mi Home devices through miiocli.

First, we need to find our device's IP address and Token.

All subsequent operations are based on these two pieces of information.

### Get Device IP Address and Token

```sh
➜  ~ miiocli cloud
Username: your Xiaomi account
Password: your password
== Xiaomi Smart Camera C400 (Device online ) ==
    Model: chuangmi.camera.039a01
    Token: XXX
    IP: 192.168.31.170 (mac: 94:F8:27:05:5B:AA)
    DID: 1029838640
    Locale: cn
== Xiaomi Smart Camera (Device online ) ==
    Model: chuangmi.camera.029a02
    Token: XXX
    IP: 192.168.31.233 (mac: 60:7E:A4:C0:E6:26)
    DID: 525571884
    Locale: cn
== Smart Plug-Ubuntu (Device online ) ==
    Model: chuangmi.plug.212a01
    Token: XXX
    IP: 192.168.31.141 (mac: 58:B6:23:EB:11:EF)
    DID: 433609370
    Locale: cn
```

Here basically can see devices under your account, then we can control these devices through miiocli commands.

## Control Devices

```sh
# This is a camera device
➜  ~ miiocli device --ip 192.168.31.170 --token xxxx info
Running command info
Model: chuangmi.camera.039a01
Hardware version: Linux
Firmware version: 5.1.6_0420
Supported using: GenericMiot
Command: miiocli genericmiot --ip 192.168.31.170 --token xxxx
Supported by genericmiot: True
```

Here need to pay attention to Supported by genericmiot, if have this, basically can control this device through miiocli genericmiot.

```txt
# Supported by genericmiot: True
Note that the command field which gives you the direct command to use for controlling the device.

If the device is supported by the genericmiot integration as stated in the output,

you can also use miiocli genericmiot for controlling it.
```

![miiocli screenshot](./img/miiocli.png)

The following documentation excerpted and translated from official docs.

### Control Modern (MIoT) Devices

Most modern (MIoT) devices will automatically be supported by genericmiot integration.

Internally, it uses ("miot spec") files to understand supported features like sensors, settings, and actions.

- <https://home.miot-spec.com/spec/xiaomi.controller.86v1>
- [Mi Home Product Library (Unofficial)](https://home.miot-spec.com/)

This device model specific file will be downloaded (and cached locally) when you first use genericmiot integration.

All supported device features can be controlled using common commands: status (show device status), set (change settings), actions list available actions, and call execute actions.

### Device Status

Executing status will show current device status, and acceptable values for settings (marked with access RW):

```shell
miiocli genericmiot --ip 127.0.0.1 --token 00000000000000000000000000000000 status

Service Light (light)
        Switch Status (light:on, access: RW): False (<class 'bool'>, )
        Brightness (light:brightness, access: RW): 60 % (<class 'int'>, min: 1, max: 100, step: 1)
        Power Off Delay Time (light:off-delay-time, access: RW): 1:47:00 (<class 'int'>, min: 0, max: 120, step: 1)
```

### Change Settings

To change settings, you need to provide the setting name (e.g., in the above example light:brightness):

```shell
 miiocli genericmiot --ip 127.0.0.1 --token 00000000000000000000000000000000 set light:brightness 0

 [{'did': 'light:brightness', 'siid': 2, 'piid': 3, 'code': 0}]
```

### Use Actions

Most devices will also provide actions:

```sh
miiocli genericmiot --ip 127.0.0.1 --token 00000000000000000000000000000000 actions

Light (light)
        light:toggle            Toggle
        light:brightness-down   Brightness Down
        light:brightness-up     Brightness Up
```

These can be executed using the call command:

```sh
miiocli genericmiot --ip 127.0.0.1 --token 00000000000000000000000000000000 call light:toggle

{'code': 0, 'out': []}
```

Use miiocli genericmiot --help for more available commands.

Detailed docs here:

- <https://github.com/rytilahti/python-miio?tab=readme-ov-file#controlling-modern-miot-devices>

## Control Your Mi Home Devices with One Line of Code

Based on the above operations, this one line of code came out.

```sh
# Get executable commands
miiocli genericmiot --ip 192.168.31.170 --token xxxx actions

## Execute action
miiocli genericmiot --ip 192.168.31.170 --token xxxx call camera:record-start

## Change property
miiocli genericmiot --ip 192.168.31.170 --token xxxx set camera:record-mode 1
```

Of course if you need to write some code yourself, it's like this.

```python
from miio import Device, DeviceFactory, DeviceStatus
from loguru import logger

class MiIotDevice():
    def __init__(self, ip, token) -> None:
        self.device: Device = DeviceFactory.create(ip, token)
        self.device_info = self.get_info()
        logger.info(f"Device: {ip}-{token} created")

    def load_status(self):
        if self.device is None:
            return None
        d_status: DeviceStatus = self.device.status()
        return d_status.data

    def get_info(self):
        return {
            "ip": self.device.ip,
            "model": self.device.model,
            "device_id": "nx_"+str(self.device.device_id),
            "mi_device_id": self.device.device_id
        }

    def call_action(self, name, args):
        if self.device is None:
            raise Exception("Device not ready")
        call_result = self.device.call_action(name, args)
        logger.info(
            f"call_action: {name}-{args} result: {call_result}")
        return call_result

    def set_properties(self, properties):
        if self.device is None:
            raise Exception("Device not ready")
        call_result = self.device.send("set_properties", properties)
        logger.info(
            f"set_properties: {properties} finish, result: {call_result}")
        return call_result
```

Basically, as above.

## Summary

python-miio is a good tool for controlling Mi Home devices

- Through miiocli can conveniently control devices
- Through miiocli genericmiot can control modern devices
- Through miiocli genericmiot actions can view device supported actions
- Through miiocli genericmiot status can view device status
- Through miiocli genericmiot set can change device properties
- Through miiocli genericmiot call can execute device actions
- Through miiocli genericmiot --help can view more commands

At the same time can write code to tinker with specific devices, making some automation things also simple.

Wish everyone have fun~

## References

- [python-miio](https://github.com/rytilahti/python-miio)
- [miot-spec Mi Home Device Properties Query (Unofficial)](https://home.miot-spec.com/)
