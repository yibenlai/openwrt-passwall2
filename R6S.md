# NanoPi R6S

## 参考资料

- [FriendlyElec Wiki](https://wiki.friendlyelec.com/wiki/index.php/NanoPi_R6S)
- [openwrt-passwall2](https://github.com/Openwrt-Passwall/openwrt-passwall2)
- [ImmortalWrt](https://github.com/immortalwrt/immortalwrt)
- [ImmortalWrt Firmware Selector](https://firmware-selector.immortalwrt.org/)

## 设备信息

- Model: FriendlyARM NanoPi R6S
- Platform: rockchip/armv8
- Version: 24.10.5 (r33805-7c4e882aaf6f)

---

## 软路由方案：ImmortalWrt + Passwall2

### 总体架构

```text
光猫 (桥接模式)
    │
    ▼
NanoPi R6S (ImmortalWrt + Passwall2)
├── LAN1 → 主路由/交换机
├── LAN2 → 可选直连设备
└── WAN  → 拨号上网 (PPPoE)
```

---

### 第一步：获取 ImmortalWrt 固件

1. 前往 [firmware-selector.immortalwrt.org](https://firmware-selector.immortalwrt.org/)
2. 搜索 `NanoPi R6S`
3. 下载 `EMMC` 版本（写入内置存储）或 `SD Card` 版本（TF 卡启动）

- Target: `rockchip/armv8`
- Profile: `friendlyarm_nanopi-r6s`
- 镜像文件名类似：`immortalwrt-24.10.x-rockchip-armv8-friendlyarm_nanopi-r6s-squashfs-sysupgrade.img.gz`

---

### 第二步：刷入固件

#### 方案 A：TF 卡启动（推荐测试用）

```bash
# Linux/macOS
gunzip immortalwrt-*.img.gz
dd if=immortalwrt-*.img of=/dev/sdX bs=1M status=progress

# Windows 用 balenaEtcher 或 Win32DiskImager
```

插入 TF 卡后开机，R6S 优先从 TF 卡启动。

#### 方案 B：写入 eMMC（生产用）

TF 卡启动后，通过 dd 将镜像写入 eMMC：

```bash
wget http://your-server/immortalwrt-emmc.img
dd if=immortalwrt-emmc.img of=/dev/mmcblk2 bs=1M status=progress
sync
```

---

### 第三步：初始配置

默认地址：`192.168.1.1`，用户名 `root`，密码为空

#### 接口设置（网络 → 接口）

| 接口 | 类型  | 说明                              |
|------|-------|-----------------------------------|
| WAN  | PPPoE | 填入宽带账号密码（或 DHCP 接光猫）|
| LAN  | 静态  | 默认 192.168.1.1，可改为偏好网段  |

将光猫改为**桥接模式**，由 R6S 负责 PPPoE 拨号。

---

### 第四步：安装 Passwall2

#### 方法 A：通过 opkg

```bash
opkg update
opkg install luci-app-passwall2
```

#### 方法 B：手动安装依赖

```bash
opkg update
opkg install kmod-tun \
             ip-full \
             iptables-nft \
             luci-app-passwall2
```

#### 方法 C：编译时集成（最稳定）

在 `make menuconfig` 中勾选：

```text
LuCI → Applications → luci-app-passwall2
```

---

### 第五步：Passwall2 配置

1. **LuCI → Services → Passwall2**
2. 添加节点（支持 vmess / vless / trojan / shadowsocks / hysteria2 等）
3. 开启**透明代理**

#### 分流策略

| 流量类型     | 处理方式 |
|--------------|----------|
| 大陆 IP/域名 | 直连     |
| GFW 列表域名 | 代理节点 |
| 其余         | 直连     |

#### DNS 防污染

- 国内 DNS：`223.5.5.5`（阿里）或 `119.29.29.29`（腾讯）
- 远程 DNS：通过代理节点解析（如 `8.8.8.8`）

---

### 第六步：验证与优化

```bash
# 检查连通性
ping -c 4 google.com

# 查看代理日志
logread | grep passwall

# 备份配置
sysupgrade -b backup.tar.gz
```

#### 性能说明

R6S 搭载 RK3588S（4x A76 + 4x A55），千兆路由绰绰有余：

- 全速千兆转发：无压力
- 透明代理吞吐：取决于节点带宽，CPU 占用低

---

### 注意事项

| 事项     | 说明                                         |
|----------|----------------------------------------------|
| 固件选择 | 优先用 ImmortalWrt，对国内设备支持更好       |
| 备份配置 | 稳定后用 `sysupgrade -b backup.tar.gz` 备份  |
| 更新固件 | 用 `sysupgrade` 而非重新刷入，保留配置       |

---

## Passwall1 vs Passwall2 分流策略详解

### 核心架构差异

| 层面         | Passwall1              | Passwall2                             |
|--------------|------------------------|---------------------------------------|
| 代理核心     | xray / v2ray           | **sing-box**（主）+ xray              |
| 透明代理层   | iptables + ipset       | iptables/nftables + sing-box 路由引擎 |
| 分流执行位置 | **内核态**（iptables） | **用户态**（sing-box 路由规则）       |
| DNS 方案     | dnsmasq + ipset        | **FakeIP** 或 DNS 分流（独立策略）    |

---

### Passwall1 分流：粗粒度，模式驱动

只有几个预设模式，本质是：**先用 DNS 判断 IP，再用 ipset 匹配 IP 决定走不走代理**

```text
模式选项：
  ① 绕过大陆 IP（GFW 域名走代理，大陆 IP 直连）
  ② 仅代理 GFWList
  ③ 全局代理
  ④ 直连
```

流量判断链路：

```text
域名请求
  → dnsmasq 查 GFW/CN 列表
  → 把域名对应 IP 加入 ipset
  → iptables 匹配 ipset → 转发到代理 or 直连
```

缺点：DNS 必须先解析才能分流，存在 DNS 污染窗口期；无法处理纯 IP 连接的细分。

---

### Passwall2 分流：细粒度，规则引擎驱动

sing-box 内置路由引擎，支持多维度规则叠加：

| 规则维度       | 说明                       |
|----------------|----------------------------|
| domain         | 精确域名                   |
| domain_suffix  | 域名后缀（*.google.com）   |
| domain_keyword | 域名关键词                 |
| domain_regex   | 正则匹配                   |
| geosite        | 预置域名集合（cn 等）      |
| ip_cidr        | IP 段                      |
| geoip          | 预置 IP 集合（cn / private）|
| port           | 端口 / 端口范围            |
| protocol       | 协议嗅探（http / tls / quic）|
| rule_set       | 外部规则集（可订阅更新）   |

规则示例（sing-box 内部逻辑）：

```json
{
  "route": {
    "rules": [
      { "geosite": ["cn"], "outbound": "direct" },
      { "geoip": ["cn", "private"], "outbound": "direct" },
      { "rule_set": ["gfw-list"], "outbound": "proxy" },
      { "outbound": "direct" }
    ]
  }
}
```

---

### DNS 策略差异（最关键）

#### Passwall1：dnsmasq 分流

```text
所有 DNS → dnsmasq
  ├── GFW 域名 → dns2socks（通过代理查）
  └── 其余     → 国内 DNS 直接查

问题：DNS 查询和流量代理是两套系统，容易不同步
```

#### Passwall2：FakeIP 模式（推荐）

```text
所有 DNS → sing-box DNS 模块（FakeIP）
  ├── 返回假 IP（198.18.0.0/15 段）给应用
  ├── sing-box 截获流量时查 FakeIP 表还原真实域名
  └── 用域名直接匹配路由规则 → 不依赖 IP 判断

优点：
  ✓ 彻底规避 DNS 污染（应用拿到假IP，真实解析在代理端）
  ✓ 分流决策基于域名而非 IP，更准确
  ✓ 纯 IP 连接也能通过 sniff 还原域名
```

#### Passwall2：DNS 分流模式（替代 FakeIP）

```text
国内域名（geosite:cn） → 223.5.5.5（直连查）
境外域名             → 8.8.8.8（走代理查）
```

---

### 流量嗅探（Passwall2 独有）

sing-box 对流量做深度嗅探，识别实际目标：

```text
HTTPS → 读取 TLS SNI → 得到真实域名 → 重新匹配路由规则
HTTP  → 读取 Host 头  → 得到真实域名 → 重新匹配路由规则
QUIC  → 嗅探域名      → 重新匹配路由规则
```

实际意义：即使应用直接用 IP 连接（绕过 DNS），Passwall2 也能通过嗅探还原域名并正确分流。Passwall1 对这类流量无能为力。

---

### 节点与出口策略

| 功能         | Passwall1    | Passwall2              |
|--------------|--------------|------------------------|
| 多节点       | 手动选主节点 | 支持节点组             |
| 负载均衡     | 不支持       | 支持（轮询 / 哈希）    |
| 自动选优     | 不支持       | 支持（延迟最低）       |
| 故障转移     | 不支持       | 支持（主备切换）       |
| 按规则选出口 | 有限         | **完整支持**           |

---

### R6S 推荐分流配置

```text
DNS：FakeIP 模式

规则顺序（从上到下）：
  1. geoip:private            → 直连（局域网）
  2. geosite:cn               → 直连
  3. geoip:cn                 → 直连
  4. geosite:geolocation-!cn  → 代理
  5. 兜底                     → 直连（保守）或 代理（激进）
```

这套规则比 Passwall1 的模式选择更精准，且不会因 DNS 污染导致分流失效。
