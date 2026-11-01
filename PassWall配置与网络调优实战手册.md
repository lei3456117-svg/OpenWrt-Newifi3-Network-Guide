# PassWall 企业级配置与网络深度调优实战手册 (Newifi 3 D2 终极稳定版)

> **设备与运行环境**：
> - **硬件型号**：联想 Newifi 3 (D2)（联发科 MT7621AT 双核四线程，512MB RAM，32MB Flash）
> - **系统固件**：ImmortalWrt 23.05.4（Linux 内核 5.15，firewall4 + nftables 架构）
> - **核心插件**：PassWall (`luci-app-passwall` + `luci-i18n-passwall-zh-cn`)
> - **核心引擎**：Trojan-Plus / Xray + ChinaDNS-NG
> - **关联网络**：铁威马 NAS (`192.168.1.198` / `100.87.114.20`) + Tailscale P2P 异地组网
> - **日期**：2026-10-31

---

## 一、 架构迭代背景：为何弃用 v2rayA 切换至 PassWall？

在此前的使用中，Newifi 3 曾运行 OpenWrt 25.12.5 + v2rayA。但随着网络设备与服务增多，遭遇了多重底层瓶颈：

| 维度 | v2rayA 架构痛点 | ImmortalWrt 23.05.4 + PassWall 方案 |
| :--- | :--- | :--- |
| **系统稳定性** | 25.12.5 快照版在 MT7621 MIPS 架构下 Go 符号表偶发越界，报错 `panic before malloc heap initialized` | 官方成熟稳定版（23.05.4），内核 5.15 驱动成熟，零内存越界崩溃 |
| **内存与负载** | v2rayA 常驻内存高，启停时无法彻底回收，内存余量常低于 50MB | 纯 C / 紧凑 Go 架构，系统空闲内存高达 **320MB+**，CPU 负载长期仅 **0.2~0.3** |
| **节点协议兼容** | 捆绑的新版 Xray 删除了 `allowInsecure: true`，导致伪装 SNI 的 Trojan 节点证书握手失败 | 原生支持 `Trojan-Plus` 与 `Xray` 双引擎，100% 兼容各类中转机场节点 |
| **透明代理与分流** | 独立于系统防火墙，容易截胡局域网内部通信，配置复杂且易与终端客户端冲突 | 基于 nftables / iptables 底层链式分流，支持端口级、IP 级无损旁路放行 |

---

## 二、 PassWall 安装与核心组件部署

### 1. 软件源更新与安装
在全新刷入 ImmortalWrt 23.05.4 后，登录后台网页（`http://192.168.1.1`）执行：
1. 进入 **系统 -> 软件包**，点击 **更新列表...**；
2. 搜索并安装核心包：
   * `luci-app-passwall`（主程序控制面板）
   * `luci-i18n-passwall-zh-cn`（中文语言包）
3. 刷新页面后，顶部导航栏即出现 **服务 -> PassWall** 菜单。

### 2. 节点导入与密码丢失排障（关键踩坑）
* **踩坑现象**：导入订阅后选择美国节点保存，PassWall 无法启动，日志提示鉴权失败。
* **原因分析**：部分机场订阅转换脚本在导入特定 Trojan 节点时，密码字段（UUID）在前端提交时变空。
* **修复方法（SSH 底层一键写入）**：
  ```bash
  # 1. 查找节点 ID
  uci show passwall | grep -iE '美国|US'
  
  # 2. 补全对应节点的密码字段并应用
  uci set passwall.<节点ID>.password='你的节点密码或UUID'
  uci commit passwall
  /etc/init.d/passwall restart
  ```

---

## 三、 整网打通：NAS 与 Tailscale 打洞深度调优

为了保证局域网千兆高速传输，同时打通异地办公的 Tailscale P2P 直连，必须解决透明代理对打洞握手包的拦截。

### 1. 根因剖析：代理插件为什么会导致 Tailscale 变卡？
* **故障现象**：在异地终端运行 `tailscale ping 100.87.114.20` 显示 `via DERP(hkg) in 1500ms`（绕道香港中继），连接极慢。
* **物理机制**：PassWall 默认接管所有 UDP 流量（`1:65535`）。Tailscale 发送的 STUN 打洞探测包（端口 3478）和通信包（端口 41641）被路由器代理抓包并转发至海外服务器，导致两端无法完成 NAT 穿透握手，被迫降级为延迟高达上千毫秒的公共中继节点。

### 2. 核心调优三部曲

#### ① PassWall 放行 Tailscale 通信端口
在 PassWall 转发设置中将 STUN 与 Tailscale 端口加入非代理列表：
```bash
uci set passwall.@global_forwarding[0].udp_no_redir_ports='3478,41641'
uci commit passwall
/etc/init.d/passwall restart
```
> **生效验证**：在路由器终端执行 `nft list table inet passwall | grep 41641`，输出包含 `udp dport { 3478, 41641 } counter return`，证明打洞数据直接放行，不再进代理。

#### ② 防火墙配置静态端口转发 (DNAT)
外部打洞连接需要能穿透路由器直接抵达 NAS：
```bash
uci add firewall redirect
uci set firewall.@redirect[-1].name='tailscale'
uci set firewall.@redirect[-1].src='wan'
uci set firewall.@redirect[-1].proto='udp'
uci set firewall.@redirect[-1].src_dport='41641'
uci set firewall.@redirect[-1].dest_ip='192.168.1.198'
uci set firewall.@redirect[-1].dest_port='41641'
uci set firewall.@redirect[-1].target='DNAT'
uci commit firewall
/etc/init.d/firewall restart
```

#### ③ 启用 Full Cone NAT（全锥型 NAT）
ImmortalWrt 23.05.4 内置 Full Cone NAT 加速。在防火墙配置中确保：
```bash
uci set firewall.@defaults[0].fullcone='1'
uci commit firewall
```
> **状态验证**：在 Mac 或内网机器运行 `tailscale netcheck`，报告显示 `MappingVariesByDestIP: false`，代表路由器已工作在全锥型独立映射模式，为打洞提供最高成功率。

---

## 四、 铁威马 NAS 局域网与远程两套 IP 访问法则

| 场景 | 访问入口 | 网络协议 | 延迟 | 实际速率 | 适用业务 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **公司办公室内** | `smb://192.168.1.198` | 局域网物理直连 | **0.4ms ~ 0.8ms** | **110MB/s ~ 280MB/s**（千兆跑满） | 大文件剪辑、备份、工程协同 |
| **居家 / 异地出差** | `smb://100.87.114.20` | Tailscale P2P 穿透 | **19ms ~ 35ms** | 跑满办公室上行带宽 | 远程资料调取、代码同步 |

> ⚠️ **避坑警示**：人在办公室时，切勿通过 `100.87.114.20` 挂载硬盘！否则会触发路由器 NAT 回环（Hairpinning），导致本可在内网传输的流量绕道外网，不仅增加路由器负荷，还会导致速度骤降。

---

## 五、 Wi-Fi 5G 芯片硬件防崩溃加固

Newifi 3 的 5G 无线芯片为联发科 `MT76x2E`。如果使用默认无线参数，会引发致命硬件崩溃：

### 1. 硬件崩溃日志还原
当 5G 工作在低频 36 信道且开启了弱信号剔除时，路由器内核日志（`dmesg`）会不断报错：
```text
[103539.761045] ieee80211 phy1: Hardware restart was requested
```
这会导致 Wi-Fi 信号短暂消失、设备频繁掉线断流。

### 2. 终极加固方案（已实测验证）
```bash
# 1. 锁死中国国家代码
uci set wireless.radio0.country='CN'
uci set wireless.radio1.country='CN'

# 2. 5G 切换到高频纯净信道 149 (5.745 GHz)
uci set wireless.radio1.channel='149'

# 3. 关闭弱信号丢包踢除机制（避免移动设备跳频掉线）
uci set wireless.default_radio0.disassoc_low_ack='0'
uci set wireless.default_radio1.disassoc_low_ack='0'

# 4. 提交配置并重载 Wi-Fi 驱动
uci commit wireless
wifi reload
```

---

## 六、 域名分流与 AI 业务规则设计

### 1. 节点分配原则
* **TCP 主节点**：优先选用 **`[IPLC·US] 美国`** 专线。
  * **优势**：物理延迟稳定（~140ms），出口 IP 纯净，100% 兼容 OpenAI / ChatGPT、Google Gemini / AI Studio、Anthropic / Claude、Cursor (含 WorkOS 鉴权及 x.ai 探针)。
* **国内直连白名单**：
  * PassWall 的 `chn_list='direct'` 与 `passwall_chn` IP 集合生效中，百度、阿里、腾讯全系国内服务内核级 0 毫秒放行，不走代理。

### 2. DNS 防污染体系
* **DNS 转发模式**：`ChinaDNS-NG` + `Dnsmasq` 结合。
* **国内解析**：通过系统默认的上游 DNS（如运营商或 `223.5.5.5`）解析国内域名。
* **海外解析**：经由远程安全 DoH / TCP DNS（`1.1.1.1`）解析，从源头杜绝 GFW 污染与劫持，保障海外开发工具秒开。

---

## 七、 常用维护与排障速查表

| 操作需求 | 终端执行命令 |
| :--- | :--- |
| **重启 PassWall** | `/etc/init.d/passwall restart` |
| **查看 PassWall 运行日志** | `logread \| grep -i passwall \| tail -n 30` |
| **查看系统剩余内存与负载** | `uptime; free -m` |
| **测试外网连接状态 (Google)** | `curl -I -s --connect-timeout 4 -x socks5h://127.0.0.1:1070 https://www.google.com \| head -n 3` |
| **测试国内直连速度 (百度)** | `curl -I -s --connect-timeout 4 https://www.baidu.com \| head -n 3` |
| **测试 NAS 内网端口** | `nc -zv -w 2 192.168.1.198 445` |
| **测试 Tailscale 远程端口** | `nc -zv -w 2 100.87.114.20 445` |
| **查看 5G 无线工作状态** | `uci show wireless.radio1` |

---

## 八、 总结与最佳实践

1. **固件选型**：对 MT7621 老牌路由器，`ImmortalWrt 23.05.4 + PassWall` 是当前综合资源开销（320MB+ 内存空闲）与功能扩展的最佳组合，彻底告别 25.x 快照版的未知崩溃。
2. **端口旁路**：任何带有透明代理的路由器，**必须对 Tailscale (41641) 与 STUN (3478) 显式放行**，否则 P2P 打洞必死。
3. **硬件维稳**：MT7621 5G 芯片切记锁定 **149 信道**，避免驱动看门狗复位。
