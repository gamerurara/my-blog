---
title: "OpenClash 订阅更新故障排查与本地订阅转换器搭建"
description: "记录 OpenWrt 上 OpenClash 订阅更新持续失败的排查过程：从 DNS 逐层定位到订阅转换器无法解析 vless reality 节点，最终在本机搭建轻量转换器彻底解决，并实现了规则自动更新与开机自启。"
keywords: "OpenClash,OpenWrt,订阅转换器,subconverter,vless,reality,AdGuard,广告拦截,Clash"
date: 2026-08-04T15:00:00+08:00
lastmod: 2026-08-04T15:00:00+08:00
categories:
  - 技术
tags:
  - OpenClash
  - OpenWrt
  - 网络
  - 代理
  - Linux
---

<!--more-->

最近家里的网络又出问题了：YouTube 和 Google 全部打不开，OpenClash 日志里全是 `i/o timeout`。之前遇到这种情况手动切一下"增强-TUN"模式能撑一两天，但这次怎么切都不行。排查下来发现根本不是模式的问题，而是一个"转换器"在背后捣乱。记录一下完整过程。

## 环境

- 软路由：OpenWrt（x86_64，OpenClash 运行中）
- 机场订阅：某机场（后文用 `<订阅地址>` 代替）
- 订阅转换器：`api.dler.io`（OpenClash 默认）
- 客户端：Clash（mihomo 内核）

## 现象

- 本机 `curl https://www.google.com` 超时（exit 28）
- OpenClash 日志反复出现：

```
dial 🐟 漏网之鱼 mihomo --> 8.8.8.8:443 error: <旧节点IP>:6688 connect error: dial tcp <旧节点IP>:6688: i/o timeout
```

- 运行中的 `config.yaml` 只有一个节点，且该节点**端口已死**（TCP 直测不通，Clash API 显示 `alive: false`）

> 疑问：为什么手动切换"增强-TUN"模式偶尔能恢复？因为切换模式 = 重启 OpenClash = 重新测速，死节点偶尔能短暂握手成功。治标不治本。

## 排查过程：api.dler.io 到底怎么了

### 第一步：DNS 解析

在 OpenWrt 上分别用本机 DNS 和公共 DNS 解析转换器域名：

```bash
nslookup api.dler.io          # → 104.21.69.27（Cloudflare，正常）
nslookup api.dler.io 8.8.8.8  # → 172.67.203.47（正常）
```

DNS 没问题，不是解析失败。

### 第二步：HTTP 请求

```bash
curl -sv "https://api.dler.io/sub?target=clash&url=<订阅地址>"
```

结果：

```
HTTP/2 400
No nodes were found!
```

**转换器根本没挂**，它明确拒绝了请求：拉取订阅后"找不到任何节点"。但直接访问订阅地址明明能拿到节点啊？

### 第三步：找到第一个元凶 —— `noss=1` 参数

对比测试：

| 请求 | 结果 |
|---|---|
| 转换器 + 订阅地址带 `noss=1` | ❌ 400 `No nodes were found!` |
| 转换器 + 订阅地址不带 `noss=1` | ✅ 200，正常返回配置 |

`noss=1` 是"排除 SS 节点"的参数，但转换器在应用该参数时把节点全过滤了，返回空。**把 OpenClash 订阅地址里的 `&noss=1` 去掉**，订阅更新恢复。

### 第四步：第二个元凶 —— 转换器不认识 vless reality

去掉 `noss=1` 后订阅更新"成功"了，但打开 config.yaml 一看：节点全是 **ss 类型**，而订阅源里明明有 4 个 vless reality 节点。

进一步测试发现：

| 输入 | dler.io 输出 |
|---|---|
| 订阅源（2 ss + 4 vless） | 只有 2 个 ss |
| 纯 vless 订阅（`noss=1` 过滤后） | `No nodes were found!` |

**结论：api.dler.io 这个转换器实例无法解析 vless reality 节点**（reality 是较新的协议，很多公共转换器实例版本陈旧不支持）。而我又不想要 ss 节点，于是陷入死结：

- 不带 `noss=1` → 只有 ss（不想要）
- 带 `noss=1` → 纯 vless → 转换器报"无节点"

### 第五步：顺带发现 HTTP/2 响应截断

排查中还发现 dler.io 对带模板的请求会返回 **Content-Length 与实际不符**的响应：

```
HTTP/1.1 200 OK
Content-Length: 529455   ← 声称 529KB
实际只收到 130802 字节  ← 差 398653 字节
```

curl 因此报 `(18) end of response with N bytes missing`。给 OpenClash 的下载脚本加了 `--http1.1` 也只是换了个报错，治不了根。

## 解决方案：本地搭建轻量订阅转换器

既然 dler.io 靠不住，订阅源本身又是好的（返回标准 vless:// 链接），那就在本机写一个**轻量转换器**：拉取订阅 → 解析 vless 链接 → 生成 Clash YAML。

### 架构

```
本机 :8080 转换器 (Python, systemd 自启)
  ├─ 拉取机场订阅 (UA: clash.meta)
  ├─ base64 解码 → 解析 vless:// 链接
  ├─ 自动排除 ss 节点
  ├─ 生成 proxy-groups + rules
  └─ OpenClash convert_address 指向本机
```

### 核心逻辑

1. **解析 vless 链接**：`vless://uuid@server:port?flow=...&security=reality&pbk=...&sid=...` → Clash proxy 字段
2. **排除 ss**：只保留 `type: vless` 的节点
3. **规则**：从 AdGuard 官方 DNS 过滤列表（adguardteam.github.io）生成广告拦截规则，叠加本地静态规则（直连/微软/苹果等分组）

### 关键配置修改

```bash
# OpenWrt 上把转换器指向本机
uci set openclash.@config_subscribe[0].convert_address="http://<本机IP>:8080/sub"
uci commit openclash
/usr/share/openclash/openclash.sh config
/etc/init.d/openclash restart
```

### 开机自启（systemd 用户服务）

```ini
# ~/.config/systemd/user/subconverter-local.service
[Unit]
Description=Local Clash Subscription Converter
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /home/<user>/.hermes/scripts/subconverter_local.py 8080
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable --now subconverter-local.service
```

> 注意：用户服务要开机自启（无需登录），需开启 linger：`loginctl enable-linger <user>`。另外转换器需要响应 **HEAD 请求**，OpenClash 更新时会先发 HEAD 拿 ETag，不实现的话会 501。

### 规则自动更新

AdGuard 官方列表每天都会更新，写了个定时任务每天凌晨 4 点拉取最新规则并重启转换器：

```bash
# 每天 4:00 cron：拉取 → 重建规则文件 → systemctl --user restart subconverter-local.service
# 网络不可用时静默跳过，保留旧规则，不影响服务
```

## 最终效果

| 项目 | 修复前 | 修复后 |
|---|---|---|
| 节点 | 1 个死节点 | 4 个 vless reality 节点 |
| ss 节点 | 不需要但强制存在 | 自动排除，0 个 |
| 广告拦截规则 | 依赖 dler.io 模板 | **16 万+ 条**，直连 AdGuard 官方源 |
| 订阅更新 | 反复失败 | 正常 |
| 网络 | YouTube/Google 全断 | 全通 |

## 经验总结

1. **先看响应体再下结论**：`HTTP 400` 不等于"服务挂了"，读 body 才能看到 `No nodes were found!` 这种真实原因。
2. **公共转换器靠不住**：版本陈旧、不支持新协议、响应还会截断。自建轻量转换器其实很简单——订阅源返回的链接格式是标准的，解析成本很低。
3. **`noss=1` 这类参数要谨慎**：它在不同转换器上的行为不一致，可能把节点全过滤掉。
4. **重启 ≠ 修复**：切换"增强-TUN"模式能临时恢复是因为触发了重启，掩盖了真正的问题（节点失效 + 订阅更新失败）。

---

*文中所有订阅地址、节点 IP、密码等信息均已脱敏。*
