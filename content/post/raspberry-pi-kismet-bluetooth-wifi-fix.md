---
title: "树莓派 Kali 折腾记：修复 Kismet 蓝牙扫描与 Wi-Fi 热点"
description: "记录树莓派 3B 上 Kali Linux 运行 Kismet 时蓝牙扫描失效和 Wi-Fi 热点无法连接的排查与修复过程。"
keywords: "树莓派,Kali,Kismet,蓝牙,Wi-Fi热点,hostapd,hciuart,systemd"
date: 2026-07-12T17:10:00+08:00
lastmod: 2026-07-12T17:10:00+08:00
categories:
  - 技术
tags:
  - 树莓派
  - Kali
  - Kismet
  - 蓝牙
  - Wi-Fi
  - Linux
---

<!--more-->

最近发现树莓派 3B 上的 Kismet 蓝牙扫描又双叒不工作了，同时 Wi-Fi 热点手机也连不上。排查一圈发现是两个独立问题，记录一下修复过程。

## 环境

- 硬件：树莓派 3B
- 系统：Kali Linux ARM
- Wi-Fi 热点：USB 网卡 RTL8192EU（wlan1），SSID `ssss`，2.4G Channel 6
- 蓝牙：板载 BCM43430（hci0），通过 hciuart 服务管理
- Kismet：双数据源（wlan0 Wi-Fi + hci0 蓝牙）

---

## 问题一：Kismet 蓝牙扫描失效

### 现象

Kismet 正常运行，Wi-Fi 数据源正常捕获设备，但蓝牙数据源完全没有任何设备被发现。系统服务状态看起来一切正常：

```bash
$ systemctl status kismet bluetooth hciuart
# 全部 active (running)

$ hciconfig hci0
hci0: Type: Primary Bus: UART
    UP RUNNING          # ← 看起来是 UP 的
    Link mode: PERIPHERAL ACCEPT  # ← 问题在这里！
```

### 排查

用 `hcitool scan` 测试，不加 sudo 时扫描没有任何结果，但加上 sudo 后却能看到多个蓝牙设备：

```bash
$ sudo hcitool scan
Scanning ...
    40:EF:4C:15:**:**  EDIFIER R1700BT
    5C:C5:63:82:**:**  尹生的盒子
    CC:DA:20:ED:**:**  Mijia Scale S400
    44:27:F3:39:**:**  Redmi Watch 2
```

这说明蓝牙硬件和驱动都正常，问题出在接口模式上。

### 根因

树莓派板载 BCM43430 蓝牙在 `hciuart` 初始化后默认处于 **PERIPHERAL ACCEPT** 模式（外设模式，等待被连接），而不是主动扫描模式。Kismet 的 `kismet_cap_linux_bluetooth` 进程不会自动切换到扫描模式，所以一直没有任何蓝牙数据。

### 修复

需要确保在 Kismet 启动之前，hci0 处于 **PSCAN + ISCAN**（Page Scan + Inquiry Scan）模式。创建一个 systemd oneshot 服务来解决：

```ini
# /etc/systemd/system/bt-piscan.service
[Unit]
Description=Enable Bluetooth PSCAN/ISCAN for Kismet
Before=kismet.service
After=hciuart.service bluetooth.service
Requires=hciuart.service bluetooth.service

[Service]
Type=oneshot
ExecStart=/usr/bin/hciconfig hci0 piscan
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

启用并启动：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now bt-piscan.service
```

这样 `bt-piscan` 会在 hciuart 初始化蓝牙后、Kismet 启动前执行 `hciconfig hci0 piscan`，确保蓝牙处于扫描模式。

重启验证，Kismet 日志中蓝牙设备恢复正常：

```
Detected new Bluetooth device CC:DA:20:ED:**:** (Mijia Scale S400)
Detected new Bluetooth device 44:DF:65:B5:**:** (DL02V1)
Detected new Bluetooth device 44:27:F3:39:**:** (Redmi Watch 2)
...
```

---

## 问题二：Wi-Fi 热点手机无法连接

### 现象

手机能看到 SSID `ssss`，但连接时提示「无法接入网络」。树莓派端 hostapd 和 dnsmasq 日志完全没有任何客户端连接记录。

### 排查

检查 hostapd 配置文件：

```ini
# /etc/hostapd/hostapd.conf（修复前）
interface=wlan1
driver=nl80211
ssid=ssss
hw_mode=g          # ← 只有 802.11g
channel=6
wmm_enabled=0       # ← WMM 关闭
wpa=2
wpa_passphrase=******
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

问题很明显：配置只启用了 **802.11g**（最高 54Mbps），且 **WMM（Wi-Fi Multimedia）被关闭**。现代手机（尤其是 iPhone）对纯 802.11g 且无 WMM 的热点兼容性很差，要么不显示，要么连接失败。

### 修复

开启 **802.11n** 和 **WMM**：

```ini
# /etc/hostapd/hostapd.conf（修复后）
interface=wlan1
driver=nl80211
ssid=ssss
hw_mode=g
channel=6
ieee80211n=1        # ← 新增：开启 802.11n
wmm_enabled=1        # ← 修复：开启 WMM
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=*****
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

重启 hostapd 后，iPhone 秒连：

```
wlan1: STA 8a:07:7e:5b:**:** IEEE 802.11: authenticated
wlan1: STA 8a:07:7e:5b:**:** IEEE 802.11: associated (aid 1)
WPA: pairwise key handshake completed (RSN)
DHCPACK(wlan1) 192.168.10.88 iPhone
```

连接后 TX 速率达到 **104 Mbit/s**（802.11n MCS 13），比之前纯 802.11g 带宽翻倍。

---

## 总结

| 问题 | 根因 | 修复方式 |
|------|------|----------|
| Kismet 蓝牙不扫描 | hci0 处于 PERIPHERAL 模式，未开启扫描 | 创建 `bt-piscan.service` 在 Kismet 启动前执行 `hciconfig hci0 piscan` |
| 手机连不上热点 | hostapd 仅有 802.11g 且 WMM 关闭 | 开启 `ieee80211n=1` + `wmm_enabled=1` |

两个修复都已通过重启验证，配置持久化，不会再复发。
