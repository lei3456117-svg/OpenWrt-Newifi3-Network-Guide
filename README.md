# Newifi 3 (D2) OpenWrt 25.12.5 企业网络搭建与分流全手册

> 💡 **网页交互版**：本项目提供富文本排版页面，可直接访问 [在线 GitHub Pages 交互手册](https://lei3456117-svg.github.io/OpenWrt-Newifi3-Network-Guide/) 或本地查看 [`index.html`](index.html)。

---

## 📖 项目简介

本教程记录了一次从**路由器误删核心组件瘫痪**到**重刷官方正式版、U盘 Extroot 扩容、DNS 深度防污染配置、v2rayA MIPS 运行时排障、再到企业级网络双链路物理分流**的完整实操全流程。

* **硬件设备**：联想 Newifi 3 D2（MT7621AT 双核四线程，512MB RAM，32MB 机身 Flash，外接 32GB USB 3.0 闪存盘）
* **固件系统**：官方纯净版 OpenWrt 25.12.5（Linux 6.12.94 内核，现代 apk 包管理器）
* **网络环境**：公司企业有线网（交换机物理绑定 MAC + 静态 IP 策略）+ 桌面千兆非网管交换机

---

## 🎯 核心解决痛点

1. **变砖急救**：误卸载 `dnsmasq` 导致路由器无法分发 IP 后的 Breed 官方纯净底包恢复流程。
2. **闪存瓶颈**：机身剩余 23MB 闪存无法安装现代科学上网插件，通过 U 盘进行 **Extroot 根目录扩容至 3.5GB**。
3. **DNS 深度防污染**：`dnsmasq-full` 与精简版的本质区别，以及为何防污染协议能让**便宜订阅秒开 Google Anti-gravity / Gemini / Codex**。
4. **双重代理套娃踩坑**：为什么硬路由规则无法规避客户端冲突，以及最终采用**桌面交换机物理双链路分流**的最佳实践。
5. **v2rayA 进程崩溃与 MIPS Go 运行时排障**：`v2raya-2.2.7.5-r1` 导致 2017 端口拒绝连接与 `panic before malloc heap initialized` 死循环崩溃的定位与平替升级 SOP（升至 `r2`），以及客户端同时开启 Antify 导致握手超时的排错。

---

## 一、 四大血泪避坑复盘（必读经验）

### 1. 切勿在未保障依赖时直接卸载 dnsmasq（死锁断网变砖）
OpenWrt 的软件包管理器（apk）高度依赖网络域名解析。如果直接执行 `apk del dnsmasq`，系统的 DHCP 和 DNS 服务会瞬间全部终止，导致**路由器立刻失去网络连接能力**，进而无法再从镜像源下载 `dnsmasq-full`，形成致命的逻辑死锁。
* **正解**：先确保本地有离线包或网络平替链路，再执行平替安装。

### 2. 32MB 机身闪存绝不能直接装 v2rayA 插件（爆盘故障）
Newifi 3 机身 Flash 虽标称 32MB，但底层系统固件已占用大部分，可用的 Overlay 剩余空间仅 **23MB 左右**。而 `v2rayA` + `xray-core` + `geoip/geosite` 规则库完全解压后高达 **70MB+**，强行在机身 Flash 安装会瞬间爆盘死机。
* **正解**：必须插入 USB 闪存盘，格式化为 ext4，并通过 Extroot 机制将根目录 `/overlay` 扩容到 3.5GB 以上。

### 3. Mac 切勿插在路由器下游加规则搞“直连”（双重代理套娃冲突）
v2rayA 的透明代理通过 Linux 内核最顶层的 `mangle / tproxy` 链对 53 端口与外部流量执行强行拦截。即使在分流规则中加入直连规则，Mac 本地运行的 Clash / Antify 与路由器的 TProxy 仍在同一条线路上争抢 DNS 劫持权与流量标记，导致数据包被二次封装，产生严重的**“双重代理套娃”**，带来巨大网络延迟与高达 100% 的丢包率。
* **正解**：Mac 与路由器物理平级接入交换机，互不经过，干净利落。若使用移动端测试 WiFi，务必关闭手机端自带的 Antify / 客户端代理。

### 4. v2rayA 2017 端口拒绝连接与 MIPS Go 运行时崩溃死循环（Crash Loop）
使用过程中若发现无线设备突然无法翻墙，浏览器打开 `http://192.168.1.1:2017` 显示“连接被拒绝”，但路由器后台正常。
* **根因**：OpenWrt 官方软件源中的 `v2raya-2.2.7.5-r1` 存在针对 MIPS 架构 Go 编译运行时的符号表缺陷（`moduledataverify1` 越界），导致启动时抛出 `fatal error: index out of range / runtime: panic before malloc heap initialized`，进入 Procd 崩溃死循环被系统保护性杀死。
* **修复**：SSH 登录路由器，更新软件源并平替升级至 `r2` 修复版：
  ```bash
  apk update && apk add -u v2raya
  /usr/bin/v2raya --version  # 验证输出 2.2.7.5 无 panic
  /etc/init.d/v2raya restart # 重启服务并恢复 2017 端口
  ```

---

## 二、 完整实操流程（SOP 五步法）

### 步骤 1：Breed 引导模式与固件重刷
1. 路由器断开电源，用卡针按住机身背部 **RESET** 孔不放，插上电源。
2. 观察前面板指示灯双闪后松开复位键（成功进入 Breed 控制台模式）。
3. Mac 用网线直连路由器 LAN 口，设置 Mac 有线网卡静态 IP：`192.168.1.2` / 掩码 `255.255.255.0` / 网关 `192.168.1.1`。
4. 浏览器访问 `http://192.168.1.1` 进入 Breed Web 控制台。
5. 在“固件更新”中上传官方纯净底包：`openwrt-25.12.5-ramips-mt7621-d-team_newifi-d2-squashfs-sysupgrade.bin`。
6. 刷入完成后，将 Mac 有线网卡改回“自动 (DHCP)”。

### 步骤 2：公司有线环境适配（MAC 隔离与静态绑定）
公司网络部署了交换机 MAC 绑定与静态安全策略，必须保证路由器与电脑参数完全隔离：

| 设备 | 物理 MAC 地址 | IPv4 地址 | 子网掩码 / 网关 | 说明 |
|---|---|---|---|---|
| **Mac 电脑** | `6C:1F:F7:CD:18:52` | `172.16.122.57` | 255.255.255.0 / 172.16.122.254 | 公司 DHCP 自动分配 |
| **路由器 WAN 口** | `20:76:93:4A:E3:08` | `172.16.122.55` | 255.255.255.0 / 172.16.122.254 | 出厂硬件原生物理 MAC 静态绑定 |

路由器 WAN 口配置参数 (`/etc/config/network`)：
```uci
config interface 'wan'
    option device 'wan'
    option proto 'static'
    option ipaddr '172.16.122.55'
    option netmask '255.255.255.0'
    option gateway '172.16.122.254'
    option dns '221.6.4.66 114.114.114.114'
    option macaddr '20:76:93:4a:e3:08'
```

### 步骤 3：USB 闪存挂载与 3.5GB 根空间扩容 (Extroot)
插入 4GB 及以上 U 盘至路由器 USB 接口，执行以下终端命令：
```bash
# 1. 安装磁盘驱动与格式化工具
apk add kmod-usb-storage kmod-fs-ext4 block-mount e2fsprogs fdisk

# 2. 将 U 盘第一分区格式化为 ext4 文件系统
mkfs.ext4 -F /dev/sda1

# 3. 复制现有系统配置到 U 盘
mkdir -p /mnt/sda1
mount /dev/sda1 /mnt/sda1
cp -a -f /overlay/. /mnt/sda1/
umount /mnt/sda1

# 4. 配置 fstab 将 /dev/sda1 挂载为根 overlay 并重启
uci set fstab.overlay='mount'
uci set fstab.overlay.device='/dev/sda1'
uci set fstab.overlay.target='/overlay'
uci set fstab.overlay.enabled='1'
uci commit fstab
/etc/init.d/fstab enable
reboot
```

### 步骤 4：全套功能组件与插件安装
重启后根目录可用空间扩容至 3.2GB 以上，依次安装插件：
```bash
# 1. 更新包索引
apk update

# 2. 平替安装完整版 dnsmasq 并开启 IPv6 过滤
apk del dnsmasq
apk add dnsmasq-full
uci set dhcp.@dnsmasq[0].filter_aaaa='1'
uci commit dhcp
/etc/init.d/dnsmasq restart

# 3. 安装 v2rayA 与 xray 转发核心 (使用修复版 r2)
apk add v2raya xray-core luci-app-v2raya luci-i18n-v2raya-zh-cn

# 4. 安装大陆分流规则库
apk add v2ray-geoip v2ray-geosite

# 5. 安装 Web 网页终端与全套中文包
apk add ttyd luci-app-ttyd luci-i18n-ttyd-zh-cn
apk add luci-i18n-base-zh-cn luci-i18n-firewall-zh-cn luci-i18n-package-manager-zh-cn luci-i18n-attendedsysupgrade-zh-cn

# 6. 启用并启动服务
/etc/init.d/v2raya enable && /etc/init.d/v2raya start
/etc/init.d/ttyd enable && /etc/init.d/ttyd start
```

### 步骤 5：v2rayA 界面配置规范
1. 打开浏览器登录 v2rayA 面板：`http://192.168.1.1:2017`（首次进入设置账号密码）。
2. 点击右上角 **设置**（齿轮图标）：
   * **透明代理/系统代理**：选择 `启用: 大陆白名单模式`。
   * **透明代理实现方式**：选择 `tproxy`。
   * **开启 IP 转发 / 开启端口分享**：两个选项均**开启**（端口分享将 Socks5: 20170 和 HTTP: 20171 开放给局域网设备按需使用）。
   * **防止 DNS 污染**：选择 `使用已配置的DNS协议防止DNS污染`（避免软解 DoH 阻塞 CPU）。
3. 底部 **自定义规则 RoutingA**：填入以下精简分流规则（含 NAS 直连与防 15s 看门狗超时优化）：
```text
default: proxy
ip(192.168.1.198) -> direct
domain(openai.com)->OpenAi
domain(chatgpt.com)->OpenAi
domain(auth0.openai.com)->OpenAi
domain(gemini.google.com)->OpenAi
domain(generativelanguage.googleapis.com)->OpenAi
domain(aistudio.google.com)->OpenAi
domain(alkalimakersuite-pa.clients6.google.com)->OpenAi
domain(domain:mail.qq.com)->direct
domain(geosite:cn)->direct
ip(geoip:private, geoip:cn)->direct
```
> 💡 **防超时关键说明**：切勿在规则中添加 `geosite:geolocation-!cn`。MT7621 解析该 10.8MB 规则库需 17.3 秒，会触发 v2rayA 15 秒核心启动看门狗超时崩溃。上述 12 行精简规则依靠 `default: proxy` 兜底，启动仅需 2.48 秒！
4. **多出口与启动**：
   * 在左上角新增出口：`OpenAi`，绑定订阅中的 **美国节点**；
   * 默认 `PROXY` 出口绑定日常低延迟亚太节点；
   * 点击左上角【启动】生效。

---

## 三、 最终桌面物理网络拓扑图

```text
                    ┌────────────────────────────────────────┐
                    │      公司网络总线 (172.16.122.x)       │
                    └───────────────────┬────────────────────┘
                                        │ (接入桌面网口)
                                        ▼
                    ┌────────────────────────────────────────┐
                    │      桌面千兆非网管物理交换机          │
                    └───────────┬────────────────┬───────────┘
                                │                │
                                │                ▼
                                │     ┌──────────────────────────────────────────┐
                                │     │        Newifi 3 路由器 (WAN 口)          │
                                │     │  - 物理硬件 MAC: 20:76:93:4A:E3:08       │
                                │     │  - 静态 IPv4: 172.16.122.55              │
                                │     │  - U 盘扩容 3.5GB (Extroot)              │
                                │     │  - 核心服务: dnsmasq-full + v2rayA       │
                                │     │  - 独立网页终端: TTYD (端口网页直接登录) │
                                │     └────────────────────┬─────────────────────┘
                                │                          │
                                │                          ▼ (发射无线 WiFi)
                                │             ┌─────────────────────────┐
                                │             │   移动端手机 / 平板设备 │
                                │             │   - 自动走大陆白名单    │
                                │             │   - 便宜节点无感省流    │
                                │             └─────────────────────────┘
                                ▼
                    ┌────────────────────────────────────────┐
                    │            Mac 工作电脑 (有线网卡)     │
                    │  - 网卡物理 MAC: 6C:1F:F7:CD:18:52     │
                    │  - DHCP 分配 IP: 172.16.122.57         │
                    │  - 网络状态: 纯净直连公司千兆骨干网    │
                    │  - 代理方案: 本地 Clash + Antify       │
                    │  - 独享极速: YouTube 4K 秒开，         │
                    │    Codex / Gemini / AI Studio 极速响应 │
                    └────────────────────────────────────────┘
```

---

## 四、 节点更换、Xray 26 内核暴雷与官方原生 v2ray-core 离线包 (2026-10-28 增补)

### 1. 深度故障排错与解决手册
- 📄 **Markdown 完整手册**：[`v2rayA节点更换与Xray内核兼容故障排查实战手册.md`](v2rayA节点更换与Xray内核兼容故障排查实战手册.md)
- 🌐 **独立排版 HTML 手册**：[`v2rayA节点更换与Xray内核兼容故障排查实战手册.html`](v2rayA节点更换与Xray内核兼容故障排查实战手册.html)
- 📦 **免外网一键独立离线安装包**：`v2raya_v2ray-core_完美独立离线安装包_Newifi3_D2.tar.gz` (27 MB)

### 2. Xray 26+ 激进删除 `allowInsecure` 踩坑与终极平替
* **现象**：换用部分优质国内中转 Trojan 节点时，节点显示“已连接”，但服务一启动就崩溃（`exit status 23`），或连上后打不开任何外网网页（握手被直接 Reset）。
* **根因**：中转 Trojan 节点普遍采用伪装 SNI（如 `baidu.com`），真实证书为第三方域名，强依赖跳过证书校验（`allowInsecure: true`）。OpenWrt 官方默认绑定的 Xray 26.3.27 强行废除了该参数，将其作为非法语法阻断。
* **终极解法**：安装 OpenWrt 官方软件源提供的原生 **`v2ray-core (5.53.0)`** 并通过 `uci set v2raya.config.v2ray_bin='/usr/bin/v2ray'` 绑定。官方原生 v2ray-core 完整支持该机制，秒开 YouTube（HTTP/2 200）。

### 3. 最新全功能双出口 RoutingA 规则（含 Claude & Cursor）
```text
default: proxy
ip(100.64.0.0/10) -> direct
port(3478, 41641) -> direct
ip(192.168.1.198) -> direct
domain(openai.com) -> OpenAi
domain(chatgpt.com) -> OpenAi
domain(auth0.openai.com) -> OpenAi
domain(gemini.google.com) -> OpenAi
domain(generativelanguage.googleapis.com) -> OpenAi
domain(aistudio.google.com) -> OpenAi
domain(alkalimakersuite-pa.clients6.google.com) -> OpenAi
domain(anthropic.com) -> OpenAi
domain(claude.ai) -> OpenAi
domain(claude.com) -> OpenAi
domain(claude.usercontent.com) -> OpenAi
domain(cursor.com) -> OpenAi
domain(cursor.sh) -> OpenAi
domain(cursorapi.com) -> OpenAi
domain(x.ai) -> OpenAi
domain(grok.com) -> OpenAi
domain(domain:mail.qq.com) -> direct
domain(geosite:cn) -> direct
ip(geoip:private, geoip:cn) -> direct
```

