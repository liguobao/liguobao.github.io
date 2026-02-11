---
title: "Get Computer Hardware Information via Python, Including CPU, GPU, Memory, Motherboard, Hard Disk, OS, etc."
description: "Python, CPU, GPU, Memory, Motherboard, Hard Disk, OS, Hardware Info"
slug: how-to-get-hwinfo-by-python
date: 2024-07-08 08:00:00+0000
image: logo.png
categories:
  - TC
tags:
  - TC
  - Python
---

## Requirements

Previously implemented a hardware info interface in C++, using [hwinfo](https://github.com/lfreist/hwinfo), roughly returns the following hardware info.

```json
{ "cpuModel": "12th Gen Intel(R) Core(TM) i7-12700K", "gpuName": "Microsoft Remote Display Adapter | NVIDIA GeForce RTX 3070", "ramVendor": "A-DATA Technology", "ramSerialNumber": "CC500", "ramTotalSizeBytes": 34359740000, "ramFreeSizeBytes": -1, "mainBoardVendor": "Rev 1.xx", "mainBoardName": "PRIME Z690-P D4", "mainBoardSerialNumber": "xxx", "disks": [ { "model": "ST12000VN0008-2YS101", "serialNumber": " ZRT0DJ", "size": "12000132426240", "freeSize": "-1" }, { "model": "Samsung SSD 980 1TB", "serialNumber": "0025.", "size": "1000202273280", "freeSize": "-1" }, { "model": "WDC WD10EZEX-00BBHA0", "serialNumber": " WD-WATM2266", "size": "1000202273280", "freeSize": "-1" } ], "osOperatingSystem": "19044", "osShortName": "Windows", "osArchitecture": "32 bit", "diskTotalSize": 14000536972800, "logicDiskTotalFreeSpace": 0 }
```

The code is nothing special, basically calling libraries, hwinfo's examples are well written~

But a few days ago found this thing occasionally crashes, error 0000005, should be memory access error, probably null reference triggered.

Long ago fixed a string encoding bug for this library, later the maintainer upgraded several versions, I was too busy to tinker more.

Thought about it, forget it, let's implement one in Python first.

## Ask GPT?

Q:

1. I want to implement computer hardware info reading in Python3, return the above JSON, how to handle.
2. Help make it compatible, if not on Windows, give as much info as possible, for missing info give default values.
3. What are the corresponding library versions? Give me requirements

A:

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
                continue  # Avoid errors from disks without access permission
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

## Optimization

- get_hardware_info function is slow, can optimize? Some info after getting should be cacheable
- I use loguru library, please add some logs to confirm call time
- My logs use stdout, no need to write file; can lru_cache set expiration time?
- get_cpu_info is too slow

2024-07-05 16:24:16.370 | INFO | main:get_cpu_info:54 - get_cpu_info() took 1.1144 seconds

Why is cpu = cpuinfo.get_cpu_info() so slow, are there other implementations?

Final code

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

# Configure loguru to output logs to stdout
logger.remove()
logger.add(lambda msg: print(msg, end=''))

# Custom cache decorator
def timed_lru_cache(seconds: int, maxsize: int = 128):
    def wrapper(func):
        cache = {}
        cache_expiration = {}
        
        @wraps(func)
        def inner(*args):
            now = datetime.now()
            # Clear expired cache
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
                continue  # Avoid errors from disks without access permission
    except Exception as e:
        total_size, total_free_space = -1, -1
    end_time = time.time()
    logger.info(f"get_total_disk_size_and_free_space() took {end_time - start_time:.3f} seconds")
    return total_size, total_free_space

def get_hardware_info():
