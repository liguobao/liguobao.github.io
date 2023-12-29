---
layout: post
title: 一些有用的脚本
category: sctripts
date: 2016-10-04
tags:
- sctripts
---
# 一些有用的脚本

## 获取电池使用情况报告(battery-report)-电池历史记录

来源：[获取电池使用情况报告(battery-report)-电池历史记录](http://answers.microsoft.com/zh-hans/windows/wiki/windows_10-performance/%E8%8E%B7%E5%8F%96%E7%94%B5%E6%B1%A0%E4%BD%BF/2d025ee0-8529-45a0-a5ae-46aa5e67f8e8)

1. 点击任务栏搜索框，搜索：POWERSHELL
2. 鼠标右键点击搜索结果中的“Windows Powershell”，点击“以管理员身份运行”

```csharp
$HTML=[System.Environment]::GetFolderPath('Desktop')+"\"+(Get-Date -Format 'yyyy-MM-dd')+"-电池记录.html";POWERCFG /BATTERYREPORT /OUTPUT "$HTML";$TF=Get-Content "$HTML";$TF| %{$_.Replace("Battery report","电池报告")}| %{$_.Replace("COMPUTER NAME","计算机名")}| %{$_.Replace("SYSTEM PRODUCT NAME","计算机型号")}| %{$_.Replace("OS BUILD","操作系统内部版本")}| %{$_.Replace("PLATFORM ROLE","平台角色")}| %{$_.Replace("CONNECTED STANDBY","InstantGo（连接待机）")}| %{$_.Replace("REPORT TIME","报告时间")}| %{$_.Replace("Installed batteries","已安装的电池")}| %{$_.Replace("Information about each currently installed battery","查看当前已安装电池的信息")}| %{$_.Replace("NAME","名称")}| %{$_.Replace("MANUFACTURER","制造商")}| %{$_.Replace("SERIAL NUMBER","序列号")}| %{$_.Replace("CHEMISTRY","化学成分")}| %{$_.Replace("AT DESIGN CAPACITY","设计容量时")}| %{$_.Replace("DESIGN CAPACITY","设计容量")}| %{$_.Replace("FULL CHARGE CAPACITY","完全充电容量")}| %{$_.Replace("CYCLE COUNT","循环计数")}| %{$_.Replace("Recent usage","最近使用情况")}| %{$_.Replace("Power states over the last 3 days","过去72小时内的电源状态")}| %{$_.Replace("START TIME","开始时间")}| %{$_.Replace("STATE","状态")}| %{$_.Replace("SOURCE","电源")}| %{$_.Replace("CAPACITY REMAINING","剩余容量")}| %{$_.Replace("Battery usage","电池使用情况")}| %{$_.Replace("Battery drains over the last 3 days","过去72小时内的电池消耗")}| %{$_.Replace("DURATION","使用时间")}| %{$_.Replace("ENERGY DRAINED","消耗的能量")}| %{$_.Replace("Usage history","使用历史记录")}| %{$_.Replace("History of system usage on AC and battery","有关交流电源和电池的使用记录")}| %{$_.Replace("BATTERY DURATION","电池使用时间")}| %{$_.Replace("AC DURATION ","交流电源使用时间")}| %{$_.Replace("PERIOD","周期")}| %{$_.Replace("ACTIVE","活动")}| %{$_.Replace("Battery capacity history","电池设计容量历史记录")}| %{$_.Replace("Charge capacity history of the system's batteries","电池充电能力历史记录")}| %{$_.Replace("Battery life estimates based on observed drains","以观察到的消耗情况预计电池寿命")}| %{$_.Replace("Battery life estimates","预计电池寿命")}| %{$_.Replace("AT FULL CHARGE","完全充电时")}| %{$_.Replace("Current estimate of battery life based on all observed drains since OS install","以操作系统安装后所有观察到的消耗记录为基础预计的当前电池寿命")}| %{$_.Replace("Since OS install","从操作系统安装后")}| %{$_.Replace("Supported","支持")}| %{$_.Replace("Not supported","不支持")}| %{$_.Replace("BATTERY","电池")}| %{$_.Replace("Suspended","已暂停")}| %{$_.Replace("Active","活动")}| %{$_.Replace("Unspecified","未知")}| %{$_.Replace("Mobile","移动")}| %{$_.Replace("Desktop","桌面")}| %{$_.Replace("Workstation","工作站")}| %{$_.Replace("Report generated","生成当前报告")}| %{$_.Replace("Battery","电池")}| %{$_.Replace("AC","交流电源")}|Out-File "$HTML";pause
```