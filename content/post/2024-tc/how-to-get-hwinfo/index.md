---
title: 如何获取计算机的硬件信息
description: 硬件信息, hwinfo
slug: how-to-get-hwinfo
date: 2024-07-08 08:00:00+0000
categories:
  - TC
tags:
  - 2024
---

之前在C++实现过一个获取硬件信息的接口，

用的是 [hwinfo](https://github.com/lfreist/hwinfo),

大概返回了下面的硬件信息。

```json
{ "cpuModel": "12th Gen Intel(R) Core(TM) i7-12700K", "gpuName": "Microsoft Remote Display Adapter | NVIDIA GeForce RTX 3070", "ramVendor": "A-DATA Technology", "ramSerialNumber": "CC500", "ramTotalSizeBytes": 34359740000, "ramFreeSizeBytes": -1, "mainBoardVendor": "Rev 1.xx", "mainBoardName": "PRIME Z690-P D4", "mainBoardSerialNumber": "xxx", "disks": [ { "model": "ST12000VN0008-2YS101", "serialNumber": " ZRT0DJ", "size": "12000132426240", "freeSize": "-1" }, { "model": "Samsung SSD 980 1TB", "serialNumber": "0025.", "size": "1000202273280", "freeSize": "-1" }, { "model": "WDC WD10EZEX-00BBHA0", "serialNumber": " WD-WATM2266", "size": "1000202273280", "freeSize": "-1" } ], "osOperatingSystem": "19044", "osShortName": "Windows", "osArchitecture": "32 bit", "diskTotalSize": 14000536972800, "logicDiskTotalFreeSpace": 0 }
```

代码没撒看的，基本都是调库，hwinfo 的样例写得还不错~

不过前几天发现这玩意偶尔会崩掉，

错误是0000005，应该是内存访问错误，

道理来是个空引用触发的。

老早之前还给这个库修个字符串编码的Bug，

在后面老哥应该还升级过几个版本，

我自己太忙了也就没空再折腾了。

想了下，算了，先用Python实现一个吧。


## 问问GPT？

Q：
1、我想在Python3 中实现电脑硬件信息读取，返回上面的JSON，请问如何处理。
2、帮我兼容一下，如果不在Windows平台的话，尽可能给足够的信息回去，没有的信息给默认值即可。
3、对应的库版本是哪些？给我个requirements

A：

```python
import psutil
import cpuinfo
import GPUtil
import platform
import json

def get_cpu_info():
    try:
        cpu = cpuinfo.get_cpu_info()
        return cpu['brand_raw']
    except Exception as e:
        return f"Unknown CPU ({e})"

def get_gpu_info():
    try:
        gpus = GPUtil.getGPUs()
        return " | ".join([gpu.name for gpu in gpus])
    except Exception as e:
        return f"Unknown GPU ({e})"

def get_ram_info():
    try:
        if platform.system() == "Windows":
            import wmi
            c = wmi.WMI()
            for memory in c.Win32_PhysicalMemory():
                ram_vendor = memory.Manufacturer
                ram_serial_number = memory.SerialNumber
                ram_total_size = int(memory.Capacity)
                ram_free_size = psutil.virtual_memory().available
                return ram_vendor, ram_serial_number, ram_total_size, ram_free_size
        else:
            vm = psutil.virtual_memory()
            return "Unknown Vendor", "Unknown Serial", vm.total, vm.available
    except Exception as e:
        return "Unknown Vendor", "Unknown Serial", -1, -1

def get_mainboard_info():
    try:
        if platform.system() == "Windows":
            import wmi
            c = wmi.WMI()
            for board in c.Win32_BaseBoard():
                mainboard_vendor = board.Manufacturer
                mainboard_name = board.Product
                mainboard_serial = board.SerialNumber
                return mainboard_vendor, mainboard_name, mainboard_serial
        else:
            return "Unknown Vendor", "Unknown Board", "Unknown Serial"
    except Exception as e:
        return "Unknown Vendor", "Unknown Board", f"Unknown Serial ({e})"

def get_disk_info():
    try:
        if platform.system() == "Windows":
            import wmi
            c = wmi.WMI()
            disks = []
            for disk in c.Win32_DiskDrive():
                model = disk.Model
                serial_number = disk.SerialNumber.strip()
                size = int(disk.Size)
                disks.append({
                    "model": model,
                    "serialNumber": serial_number,
                    "size": size,
                    "freeSize": -1
                })
            return disks
        else:
            disks = []
            for disk in psutil.disk_partitions():
                usage = psutil.disk_usage(disk.mountpoint)
                disks.append({
                    "model": "Unknown Model",
                    "serialNumber": "Unknown Serial",
                    "size": usage.total,
                    "freeSize": usage.free
                })
            return disks
    except Exception as e:
        return [{"model": "Unknown", "serialNumber": "Unknown", "size": -1, "freeSize": -1}]

def get_os_info():
    try:
        os_version = platform.version()
        os_short_name = platform.system()
        os_arch = platform.architecture()[0]
        return os_version, os_short_name, os_arch
    except Exception as e:
        return "Unknown", "Unknown", "Unknown"

def get_total_disk_size_and_free_space():
    try:
        total_size = 0
        total_free_space = 0
        for partition in psutil.disk_partitions():
            try:
                usage = psutil.disk_usage(partition.mountpoint)
                total_size += usage.total
                total_free_space += usage.free
            except PermissionError:
                continue  # 避免某些磁盘没有权限访问导致的错误
        return total_size, total_free_space
    except Exception as e:
        return -1, -1

def get_hardware_info():
    cpu_model = get_cpu_info()
    gpu_name = get_gpu_info()
    ram_vendor, ram_serial_number, ram_total_size, ram_free_size = get_ram_info()
    mainboard_vendor, mainboard_name, mainboard_serial_number = get_mainboard_info()
    disks = get_disk_info()
    os_version, os_short_name, os_arch = get_os_info()
    total_disk_size, total_free_space = get_total_disk_size_and_free_space()

    return {
        "cpuModel": cpu_model,
        "gpuName": gpu_name,
        "ramVendor": ram_vendor,
        "ramSerialNumber": ram_serial_number,
        "ramTotalSizeBytes": ram_total_size,
        "ramFreeSizeBytes": ram_free_size,
        "mainBoardVendor": mainboard_vendor,
        "mainBoardName": mainboard_name,
        "mainBoardSerialNumber": mainboard_serial_number,
        "disks": disks,
        "osOperatingSystem": os_version,
        "osShortName": os_short_name,
        "osArchitecture": os_arch,
        "diskTotalSize": total_disk_size,
        "logicDiskTotalFreeSpace": total_free_space
    }

if __name__ == "__main__":
    hardware_info = get_hardware_info()
    print(json.dumps(hardware_info, indent=2))
```


## 优化

- get_hardware_info 这个函数执行比较慢，能优化一下吗？ 部分信息获取之后应该是可以缓存起来的
- 我使用的lorugu 库，请帮我加一些日志，确认一下调用耗时
- 我的日志使用标准输出即可，不用写文件；lru_cache 可以设置过期时间吗?
- get_cpu_info太慢了

2024-07-05 16:24:16.370 | INFO | main:get_cpu_info:54 - get_cpu_info() took 1.1144 seconds 

cpu = cpuinfo.get_cpu_info() 为嘛它这么慢，有其他的实现吗？


最终代码

```python 
import psutil
import GPUtil
import platform
import json
import time
import subprocess
from functools import wraps
from loguru import logger
from datetime import datetime, timedelta

# 配置 loguru 将日志输出到标准输出
logger.remove()
logger.add(lambda msg: print(msg, end=''))

# 自定义缓存装饰器
def timed_lru_cache(seconds: int, maxsize: int = 128):
    def wrapper(func):
        cache = {}
        cache_expiration = {}
        
        @wraps(func)
        def inner(*args):
            now = datetime.now()
            # 清除过期缓存
            for key in list(cache_expiration):
                if cache_expiration[key] < now:
                    cache.pop(key, None)
                    cache_expiration.pop(key, None)
                    
            if args in cache:
                return cache[args]
            
            result = func(*args)
            cache[args] = result
            cache_expiration[args] = now + timedelta(seconds=seconds)
            
            if len(cache) > maxsize:
                oldest = min(cache_expiration, key=cache_expiration.get)
                cache.pop(oldest, None)
                cache_expiration.pop(oldest, None)
                
            return result
        
        return inner
    return wrapper

@timed_lru_cache(60)
def get_cpu_info():
    start_time = time.time()
    try:
        if platform.system() == "Windows":
            result = subprocess.run(['wmic', 'cpu', 'get', 'Name'], stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
            cpu_model = result.stdout.split('\n')[1].strip()
        elif platform.system() == "Linux":
            result = subprocess.run(['lscpu'], stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
            for line in result.stdout.split('\n'):
                if 'Model name' in line:
                    cpu_model = line.split(':')[1].strip()
                    break
        elif platform.system() == "Darwin":
            result = subprocess.run(['sysctl', '-n', 'machdep.cpu.brand_string'], stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
            cpu_model = result.stdout.strip()
        else:
            cpu_model = "Unsupported platform"
    except Exception as e:
        cpu_model = f"Unknown CPU ({e})"
    end_time = time.time()
    logger.info(f"get_cpu_info() took {end_time - start_time:.3f} seconds")
    return cpu_model

@timed_lru_cache(60)
def get_gpu_info():
    start_time = time.time()
    try:
        gpus = GPUtil.getGPUs()
        gpu_name = " | ".join([gpu.name for gpu in gpus])
    except Exception as e:
        gpu_name = f"Unknown GPU ({e})"
    end_time = time.time()
    logger.info(f"get_gpu_info() took {end_time - start_time:.3f} seconds")
    return gpu_name

@timed_lru_cache(60)
def get_ram_info():
    start_time = time.time()
    try:
        if platform.system() == "Windows":
            import wmi
            c = wmi.WMI()
            for memory in c.Win32_PhysicalMemory():
                ram_vendor = memory.Manufacturer
                ram_serial_number = memory.SerialNumber
                ram_total_size = int(memory.Capacity)
                ram_free_size = psutil.virtual_memory().available
                break
        else:
            vm = psutil.virtual_memory()
            ram_vendor, ram_serial_number, ram_total_size, ram_free_size = "Unknown Vendor", "Unknown Serial", vm.total, vm.available
    except Exception as e:
        ram_vendor, ram_serial_number, ram_total_size, ram_free_size = "Unknown Vendor", "Unknown Serial", -1, -1
    end_time = time.time()
    logger.info(f"get_ram_info() took {end_time - start_time:.3f} seconds")
    return ram_vendor, ram_serial_number, ram_total_size, ram_free_size

@timed_lru_cache(60)
def get_mainboard_info():
    start_time = time.time()
    try:
        if platform.system() == "Windows":
            import wmi
            c = wmi.WMI()
            for board in c.Win32_BaseBoard():
                mainboard_vendor = board.Manufacturer
                mainboard_name = board.Product
                mainboard_serial = board.SerialNumber
                break
        else:
            mainboard_vendor, mainboard_name, mainboard_serial = "Unknown Vendor", "Unknown Board", "Unknown Serial"
    except Exception as e:
        mainboard_vendor, mainboard_name, mainboard_serial = "Unknown Vendor", "Unknown Board", f"Unknown Serial ({e})"
    end_time = time.time()
    logger.info(f"get_mainboard_info() took {end_time - start_time:.3f} seconds")
    return mainboard_vendor, mainboard_name, mainboard_serial

@timed_lru_cache(60)
def get_disk_info():
    start_time = time.time()
    try:
        if platform.system() == "Windows":
            import wmi
            c = wmi.WMI()
            disks = []
            for disk in c.Win32_DiskDrive():
                model = disk.Model
                serial_number = disk.SerialNumber.strip()
                size = int(disk.Size)
                disks.append({
                    "model": model,
                    "serialNumber": serial_number,
                    "size": size,
                    "freeSize": -1
                })
        else:
            disks = []
            for disk in psutil.disk_partitions():
                usage = psutil.disk_usage(disk.mountpoint)
                disks.append({
                    "model": "Unknown Model",
                    "serialNumber": "Unknown Serial",
                    "size": usage.total,
                    "freeSize": usage.free
                })
    except Exception as e:
        disks = [{"model": "Unknown", "serialNumber": "Unknown", "size": -1, "freeSize": -1}]
    end_time = time.time()
    logger.info(f"get_disk_info() took {end_time - start_time:.3f} seconds")
    return disks

@timed_lru_cache(60)
def get_os_info():
    start_time = time.time()
    try:
        os_version = platform.version()
        os_short_name = platform.system()
        os_arch = platform.architecture()[0]
    except Exception as e:
        os_version, os_short_name, os_arch = "Unknown", "Unknown", "Unknown"
    end_time = time.time()
    logger.info(f"get_os_info() took {end_time - start_time:.3f} seconds")
    return os_version, os_short_name, os_arch

@timed_lru_cache(60)
def get_total_disk_size_and_free_space():
    start_time = time.time()
    try:
        total_size = 0
        total_free_space = 0
        for partition in psutil.disk_partitions():
            try:
                usage = psutil.disk_usage(partition.mountpoint)
                total_size += usage.total
                total_free_space += usage.free
            except PermissionError:
                continue  # 避免某些磁盘没有权限访问导致的错误
    except Exception as e:
        total_size, total_free_space = -1, -1
    end_time = time.time()
    logger.info(f"get_total_disk_size_and_free_space() took {end_time - start_time:.3f} seconds")
    return total_size, total_free_space

def get_hardware_info():
    start_time = time.time()
    cpu_model = get_cpu_info()
    gpu_name = get_gpu_info()
    ram_vendor, ram_serial_number, ram_total_size, ram_free_size = get_ram_info()
    mainboard_vendor, mainboard_name, mainboard_serial_number = get_mainboard_info()
    disks = get_disk_info()
    os_version, os_short_name, os_arch = get_os_info()
    total_disk_size, total_free_space = get_total_disk_size_and_free_space()
    end_time = time.time()
    logger.info(f"get_hardware_info() took {end_time - start_time:.3f} seconds")

    return {
        "cpuModel": cpu_model,
        "gpuName": gpu_name,
        "ramVendor": ram_vendor,
        "ramSerialNumber": ram_serial_number,
        "ramTotalSizeBytes": ram_total_size,
        "ramFreeSizeBytes": ram_free_size,
        "mainBoardVendor": mainboard_vendor,
        "mainBoardName": mainboard_name,
        "mainBoardSerialNumber": mainboard_serial_number,
        "disks": disks,
        "osOperatingSystem": os_version,
        "osShortName": os_short_name,
        "osArchitecture": os_arch,
        "diskTotalSize": total_disk_size,
        "logicDiskTotalFreeSpace": total_free_space
    }

if __name__ == "__main__":
    hardware_info = get_hardware_info()
    print(json.dumps(hardware_info, indent=2))
```

## requirements.txt

```txt
psutil==5.9.5
gputil==1.4.0
WMI==1.5.1; sys_platform == 'win32'
pywin32==306; sys_platform == 'win32'
loguru==0.5.3
```

## 总结

- 我一句代码都没写。jpg
- 确实GPT过于强大。jpg
- GPT确实需要调教。jpg