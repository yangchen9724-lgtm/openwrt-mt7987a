# ImmortalWrt 云编译 · Hiveton H5000M（MT7987A / Filogic 860）5G CPE 固件

> 面向 **Hiveton H5000M / Airpi H5000M** 5G CPE：**MT7987A 四核 A53 + WiFi7(BE5040/MT7992) + 2×2.5G 网口 + eMMC 8GB + DDR4 1GB + M.2 B-Key 5G 模组 + PWM 风扇**
> 基于 **ImmortalWrt master**（设备官方支持 `hiveton_h5000m`），使用 GitHub 免费云服务器编译，不占用本机资源。
> 已集成：拼拼WiFi 收费WiFi、巴龙哥 QModem 5G 界面、H5000M 风扇/网络模式、**PassWall2 科学上网（访问国外网络）**、硬件加速、**插件扩展性基础（在线装包 + dnsmasq-full + tun/tproxy）**。

---

## 一、这套文件是什么

| 文件 | 作用 |
|---|---|
| `.github/workflows/build-openwrt.yml` | GitHub Actions 编译工作流（自动拉源码、配置、编译、上传固件） |
| `config.seed` | 编译配置：目标设备(H5000M) + WiFi7(MT7992) + 5G 模组(鼎桥/移远) + **5G 模组 WEB 界面(巴龙哥 QModem)** + **H5000M 风扇温控/网络模式切换** + **PassWall2 科学上网** + LuCI 界面 + **拼拼WiFi(Wiwiz) 全套插件** |
| `README.md` | 本说明 |

**已核实的编译依据**（截至 2026-09）：

- **Hiveton H5000M 已被 ImmortalWrt 官方支持**：`master` 分支 `target/linux/mediatek/image/filogic.mk` 内置设备 `hiveton_h5000m`（厂商 Hiveton/Airpi，DTS `mt7987a-hiveton-h5000m`），设备定义自动带上 `kmod-mt7996e` + `kmod-mt7992-23-firmware`（WiFi7 驱动/固件）、`mt7987-2p5g-phy-firmware`（2.5G PHY 固件）、`kmod-hwmon-pwmfan`（风扇）、`f2fsck/mkf2fs`（eMMC）等；
- 硬件已按 DTS 核实：WiFi 为 **MT7992（PCIe0 挂载，eeprom 存于 eMMC factory 分区）**；双 2.5G 网口 = eth0 外置 **RTL8221B PHY** + eth1 **内置 2.5G PHY**；eMMC 8GB 启动（root=PARTLABEL=rootfs）；USB3（`ssusb`）接 5G 模组；PWM 主动散热风扇；
- **设备原厂即预装 ImmortalWrt fork**（官方出货固件），因此**刷机=系统内直接 sysupgrade**，无需更换 U-Boot、无需写 eMMC 整卡镜像；固件产物为 `hiveton_h5000m-squashfs-sysupgrade.bin`（sysupgrade-tar 格式）；
- **社区 CI 已在跑**（lianxia233/OpenWRT-CI-H5000M，每日 05:00 CST 用 ImmortalWrt master 自动编译并发布），方案成熟可复现；
- 拼拼WiFi（Wiwiz）开源插件仓库 **WiFiPortal 已适配 25.12 体系**；ImmortalWrt master 的 legacy iptables 包名与 25.12 一致（`iptables-zz-legacy`），组件 `wifidog-wiwiz`（Portal 认证）、`eqos`+`luci-app-eqos`（限速）、`dcc2-wiwiz-nossl`（远程管理）、`autokick-wiwiz`（到期自动断开）全部可编译；
- 5G 模组管理界面 **巴龙哥 QModem**（GitHub `FUjr/QModem`）：以官方 feeds 方式集成（`src-git qmodem`），主界面 `luci-app-qmodem-next` 为纯 JS 现代化界面（LuCI 菜单 **Modem → QModem**），支持 模组信息/信号强度/小区/拨号设置/锁频段锁小区/AT 命令/短信收发与转发/模组重启；官方支持列表明确包含 **鼎桥 MT5700M-CN**（海思平台，USB ECM/NCM）与 **移远 RM520N 全系 / RM500 全系**（高通平台，USB QMI/MBIM/NCM/RNDIS）；配套 `qmodem-smsd`（短信服务）与 `sms-forwarder-next`（短信转发）由依赖自动带入；
- **H5000M 专属插件（FAN789）**：`luci-app-h5000m-fancontrol`（PWM 风扇多档温控，读 CPU+模组双路温度）、`luci-app-h5000m-netmode`（仅5G/仅有线/负载均衡+故障转移一键切换，配 mwan3）；
- **科学上网（PassWall2，访问国外网络）**：官方双 feed 集成（`passwall_packages` + `passwall2`，Openwrt-Passwall 组织维护），支持 SS/SSR/VMess/VLESS/Trojan/Hysteria2/NaiveProxy/Tuic 全协议 + 机场订阅自动更新 + tproxy 透明代理分流（GFW 名单/国内直连/自定义规则）；核心 xray-core / sing-box 随依赖自动编入；官方 SDK CI 每日构建验证，ImmortalWrt 兼容性有保障；
- **插件扩展性基础**：`dnsmasq-full`（完整 DNS，替代默认 dnsmasq，支持 ipset/nftset，科学上网与去广告类插件必需）、`kmod-tun` + `kmod-nft-tproxy`（透明代理内核模块）、保留 LuCI 在线装包能力（系统 → 软件包）、rootfs 分区 2GB（装插件空间充足）——日后想加 OpenClash、AdGuard Home、DDNS 等任何插件，直接在 LuCI 在线安装即可，无需重新编译；
- **网络硬件加速已核实**：ImmortalWrt master（内核 6.18）`CONFIG_NET_MEDIATEK_SOC=y`、`CONFIG_NET_MEDIATEK_SOC_WED=y`（WED 默认启用），PPE 随 mtk_eth 内置；**默认关闭"硬件流量分载"开关**（科学上网 tproxy 分流与拼拼WiFi Portal 认证需要流量经过 netfilter，硬件分载会绕过导致分流失效），需要极限转发时可在 LuCI 防火墙页面手动开启。

---

## 二、使用步骤（约 5 分钟操作 + 1.5~3 小时编译）

### 第 1 步：注册/登录 GitHub
打开 https://github.com 注册账号（或直接登录）。免费。

### 第 2 步：新建仓库
1. 右上角 **+** → **New repository**
2. Repository name 填：`openwrt-mt7987a`（可自定义）
3. 选择 **Public**（免费 Runner 必需）或 Private（需自行启用 Actions 计费）
4. 不要勾选 "Add a README"，直接 **Create repository**

### 第 3 步：上传本目录全部文件
1. 进入新建的仓库页面 → **Add file** → **Upload files**
2. 把本目录下的 **两个文件**（`config.seed`、`README.md`）和 **`.github` 文件夹**（里面是 `workflows/build-openwrt.yml`）一起拖进上传区
   - 注意：`.github` 是隐藏文件夹，Windows 下直接拖整个文件夹即可，GitHub 会自动保留目录结构
3. **Commit changes**

### 第 4 步：触发编译
1. 仓库页面顶部进入 **Actions** 标签
2. 左侧点击 **"编译 ImmortalWrt (H5000M)"**
3. 右侧 **Run workflow** → 绿色 **Run workflow** 按钮
4. 等待编译（约 1.5~3 小时），期间可以随时点进该任务看实时日志

### 第 5 步：下载固件
1. 编译成功后，进入该次运行页面
2. 底部 **Artifacts** 区域 → 下载 `immortalwrt-h5000m-master`（zip 包）
3. 解压后即得固件（GitHub 国内下载一般没问题，慢的话可用加速工具或再触发一次）

---

## 三、固件产物说明（解压 zip 后）

以 `hiveton_h5000m` 前缀开头的文件：

| 文件 | 用途 |
|---|---|
| `immortalwrt-mediatek-filogic-hiveton_h5000m-squashfs-sysupgrade.bin` | **升级固件（主要用这个）**：原厂系统内 sysupgrade 刷入，或 U-Boot tftp 刷写 |
| `...-initramfs-kernel.bin` | 内存版内核（重启即失），救砖/试机用 |
| `...-preloader.bin` / `...-bl31-uboot.fip` | 引导组件（**一般不需要刷**，仅 U-Boot 损坏救砖时用） |

> 另附 `build-config.txt`（本次实际生效配置）和所有文件的 sha256 校验值。
> 注意：本设备是 eMMC 启动 + sysupgrade-tar 产物，**不包含** rfb 方案里的 `sdcard.img.gz` 整卡镜像。

---

## 四、刷机方法

### 方案 A：系统内直接升级（推荐，原厂固件就是 ImmortalWrt fork）
1. 登录路由器 LuCI（原厂默认 `192.168.10.1` 或按说明书，root 密码见出厂标签）
2. 菜单 **系统 → 备份/升级 → 刷写新的固件**
3. 选择 `hiveton_h5000m-squashfs-sysupgrade.bin`，勾选"不保留配置"（首次刷写建议全新），点击刷写
4. 等待自动重启（约 3~5 分钟），完成后进入全新 ImmortalWrt 系统

### 方案 B：U-Boot tftp 刷写（系统进不去时用）
1. 通过 TTL 串口进入 U-Boot（波特率 115200，开机按住 reset 或中断启动）
2. 执行：
   ```
   setenv serverip 192.168.1.2
   setenv ipaddr 192.168.1.1
   tftpboot 0x46000000 immortalwrt-mediatek-filogic-hiveton_h5000m-squashfs-sysupgrade.bin
   sysupgrade ${fileaddr}
   ```
3. 重启后 `192.168.10.1`（或 192.168.1.1）登录 LuCI

> ⚠️ **与 rfb 公版方案的关键区别**：H5000M 的 WiFi 节点写死在设备树（PCIe0 → MT7992），**不需要**更换 OpenWrt/ImmortalWrt U-Boot 注入 overlay——原厂 U-Boot 直接 sysupgrade 即可，刷 U-Boot 反而有变砖风险，非必要不要动 preloader/fip。

---

## 五、本固件已包含的硬件支持

- **WiFi7（MT7992）驱动与固件**：设备定义自动带上 `kmod-mt7996e`（驱动，官方 mt76）+ `kmod-mt7992-23-firmware`（MT7992 固件，注意是 ImmortalWrt 新版包名），LuCI 里自动识别为两个 radio（2.4G + 5G）
  - ✅ 无需更换 U-Boot（DTS 写死 PCIe0 → mt7992，eeprom 读 eMMC factory 分区）
- **5G 模组**（M.2 B-Key，USB3.2）：**鼎桥 MT5700-CN、移远 RM520N 系列、移远 RM500 系列** 全覆盖
  - 内核驱动：QMI（qmi_wwan）、MBIM（cdc_mbim）、NCM（cdc_ncm）、RNDIS、CDC-ECM、串口 Modem（option/wwan/qualcomm）+ usb-modeswitch 模式切换
  - 拨号协议：LuCI 界面可选 QMI Cellular / MBIM / NCM / 3G（网络 → 接口 → 添加新接口）
  - **外挂 WEB 界面（已集成，巴龙哥 QModem）**：LuCI 菜单 **Modem → QModem**，可查看模组信息/信号强度/小区/IMSI、拨号设置、锁频段锁小区、AT 命令、短信收发与转发、模组重启
- **H5000M 专属（FAN789）**：
  - **风扇温控**：LuCI 菜单 系统 → H5000M 风扇控制（或对应菜单），读 CPU + 模组双路温度，多档 PWM 调速，5G 高负载自动加强散热
  - **网络模式切换**：LuCI 菜单 网络 → H5000M 网络模式，一键切换 仅5G / 仅有线宽带 / 负载均衡+故障转移（配 mwan3，主链路故障毫秒级切换）
- **2×2.5G 网口**：eth0 = 外置 RTL8221B（2.5G PHY），eth1 = 内置 2.5G PHY；`mt7987-2p5g-phy-firmware` 已随设备自动带上
- **USB3.0**：U 盘/存储扩展（设备自带 automount）
- **网络硬件加速（MT7987 三层加速，能力已内置）**：
  1. **WED**（无线以太网卸载引擎）：内核默认启用，WiFi↔有线流量卸载到硬件 DMA，无线转发不掉速（与科学上网无冲突，保持开启）；
  2. **PPE**（包处理引擎硬件流卸载）：mtk_eth 内置，由防火墙"硬件流量分载"触发，NAT/桥接流线速转发；
  3. **软件流表卸载**（nft flowtable / `kmod-nft-offload`）：兜底加速，兼容所有接口（含 5G 模组 wwan0）；
  - 为兼容科学上网（PassWall2 tproxy 分流）与拼拼WiFi Portal 认证，**"硬件流量分载"开关默认关闭**（需要时可到 LuCI 防火墙页面手动开启，见第八章）
- **拼拼WiFi（Wiwiz）收费WiFi全套**：Portal 网页认证、EQOS 限速、DCC2 远程管理、到期自动断开、免认证网络

### 巴龙哥 QModem 快速上手（刷机后）

1. **首次打开**：LuCI → **Modem → QModem**，页面会自动扫描 USB 总线上的模组（扫描间隔默认 30 秒，无需手动配置）；
2. **模组识别差异**：
   - 鼎桥 **MT5700-CN** → QModem 按海思(Hisilicon)平台识别，USB 网卡模式走 **ECM/NCM**，界面可看到模组信息/信号/基站；
   - 移远 **RM520N / RM500 系列** → 按高通(Qualcomm)平台识别，USB 走 **QMI**（默认）或 MBIM/NCM，支持锁频段/锁小区/AT 下发；
3. **拨号上网**：先在 LuCI → 网络 → 接口 添加 QMI Cellular 接口（或直接在 QModem 的 Dial 页设置 APN/鉴权后一键拨号）；
4. **短信**：QModem 自带短信收发界面；需要短信转发（如发到微信/Telegram）时，在 LuCI → Modem → QModem → SMS Forward 配置 Server酱/Telegram/Webhook；
5. **许可证提示**：QModem 核心为 MPL 2.0 并附"**禁止商业使用**"条款——自用无碍，若日后商用本固件请先联系作者 FUjr。

---

## 六、科学上网（PassWall2）使用说明

固件已集成 **PassWall2**（Openwrt-Passwall 组织维护，社区最活跃的 OpenWrt/ImmortalWrt 科学上网插件），LuCI 菜单 **服务 → PassWall2**。支持 SS / SSR / VMess / VLESS / Trojan / Hysteria2 / NaiveProxy / Tuic 全协议与机场订阅。

### 1. 快速开始（以机场订阅为例）
1. LuCI → **服务 → PassWall2** → **节点列表 → 订阅节点**；
2. 点击"添加"，把机场的**订阅链接**粘贴进去，设置更新间隔（如 12 小时），保存并应用；
3. 等待订阅更新完成后，在**节点列表**里点选一个节点（可测速挑选）；
4. 回到 **运行状态** 页，把"启用"开关打开，选 **TCP 节点 / UDP 节点**，模式建议 **透明代理（tproxy）**；
5. 浏览器访问 https://www.google.com 验证是否可访问国外网络。

### 2. 分流与规则（默认即可用）
- PassWall2 自带 **GFW 名单分流**：被墙域名走代理、国内域名直连、其余按规则（默认"GFW 模式"即可）；
- 需自定义（如某域名强制走代理/直连）→ **分流规则 → 自定义规则** 添加；
- DNS：PassWall2 接管 DNS 分流（依赖本固件的 dnsmasq-full），无需手动配置。

### 3. 多节点/故障转移
**节点列表 → 自动切换**：添加多节点后可设置测速间隔与切换策略，节点挂掉自动切换，适合 5G 网络波动场景。

### 4. 与硬件加速的关系（重要）
PassWall2 使用 tproxy 透明代理，要求流量经过 netfilter 用户规则。**本固件已默认关闭"硬件流量分载"**（防火墙页面该开关为未勾选），两者可正常共存；软件分载（flow_offloading）为 ImmortalWrt 默认值，如发现个别节点连接异常，到 LuCI → 网络 → 防火墙 取消勾选"软件流量分载"再试。
> 反向提示：如果你**不用**科学上网/Portal 认证，且想要极限内网转发性能，可以在防火墙页面手动勾选"硬件流量分载"（PPE 加速），此时 PassWall2 需要保持关闭。

### 5. 常见协议与端口
SS/VMess/Trojan/VLESS 等均可直接在"添加节点"里选类型填写，或粘贴 `ss://`、`vmess://`、`vless://`、`trojan://` 分享链接自动解析。

---

## 七、插件扩展性说明（想加什么插件都可以）

本固件在编译时专门留好了"扩展底子"，**绝大多数插件无需重新编译、直接在路由器上在线安装**：

| 扩展基础 | 说明 |
|---|---|
| **在线装包** | LuCI → **系统 → 软件包**，可直接搜索/安装 ImmortalWrt 官方源里的数千个软件包（ImmortalWrt 软件源默认已配置） |
| **dnsmasq-full** | 已用完整版替代默认 dnsmasq（支持 ipset/nftset/通配符），AdGuard Home、去广告、科学上网类插件的硬性前置已满足 |
| **tun/tproxy 内核模块** | `kmod-tun`、`kmod-nft-tproxy` 已编入，OpenClash、Nikki、Sing-Box 等代理内核可直接运行 |
| **rootfs 空间** | eMMC rootfs 分区 2GB（squashfs 只读 + 可写 overlay），装几十个插件毫无压力 |
| **USB 存储** | USB3 + 自动挂载已内置，插件/日志可放 U 盘 |

**常用扩展示例（在线安装即可）**：
- **OpenClash**（Clash 内核图形化，与 PassWall2 二选一使用）：LuCI 软件包搜索 `luci-app-openclash` 安装；
- **AdGuard Home**（DNS 广告过滤）：`luci-app-adguardhome`；
- **DDNS**（动态域名）：`luci-app-ddns`；
- **异地组网**：`luci-app-tailscale` / `luci-app-easytier`；
- 规则提醒：**科学上网插件（PassWall2 / OpenClash / Nikki）同时只启用一个**，避免端口与防火墙规则冲突。

---

## 八、网络硬件加速说明（本固件已内置能力）

### 1. 编译期做了什么
- 内核层：`CONFIG_NET_MEDIATEK_SOC=y` + `CONFIG_NET_MEDIATEK_SOC_WED=y`（WED 引擎，ImmortalWrt 对 MT7987 默认启用，无需干预）；
- 软件包层：`kmod-nft-offload`（nft flowtable 卸载）编译进固件；
- 运行层（uci-defaults）：**默认关闭"硬件流量分载"**（`flow_offloading_hw=0`）——这是为科学上网（PassWall2 tproxy 分流）与拼拼WiFi Portal 认证让路：硬件分载会绕过 netfilter 用户规则，导致代理分流失效、认证页不弹出；软件分载保留 ImmortalWrt 默认值。

### 2. 如何验证加速生效
- LuCI → 网络 → 防火墙 → 常规设置：需要硬件加速时，勾选"启用硬件流量分载"（默认关闭）并应用；
- SSH 执行 `cat /sys/kernel/debug/mtk_ppe/entries 2>/dev/null | head` 能看到 PPE 表中的转发条目（有输出即硬件分载在干活）；
- 最直接：千兆测速（iperf3 / speedtest），2.5G 内网互传应稳定在 2.3Gbps 以上、NAT 转发不掉线。

### 3. 何时开启/关闭（权衡说明）
- **默认状态（推荐）**：硬件分载关闭、软件分载默认——科学上网 + Portal 认证 + 日常上网全部正常工作；
- **需要极限转发性能时**（如纯内网 NAS 大流量互传、不用科学上网和 Portal）：防火墙页面手动勾选"启用硬件流量分载"（PPE 生效），同时可保留软件分载；
- **开启科学上网后个别节点异常**：到防火墙页面取消勾选"软件流量分载"再试；
- 注意：硬件分载与**多拨（mwan3）、SQM/QoS 限速、部分流量统计**不兼容，开启后如异常请关闭。

---

## 九、WiFi 驱动说明（网速与稳定性关键）

### 1. 本固件已集成什么
| 组件 | 包名 | 作用 |
|---|---|---|
| WiFi7 驱动 | `kmod-mt7996e` | ImmortalWrt 官方 mt76 驱动，支持 MT7992 芯片（设备定义自动带上） |
| MT7992 固件 | `kmod-mt7992-23-firmware` | 芯片运行固件（ImmortalWrt 新版包名，自动带上） |
| 2.5G PHY 固件 | `mt7987-2p5g-phy-firmware` | 双 2.5G 网口 PHY 固件（自动带上） |
| 硬件加速联动 | WED（内核内置） | 无线↔有线流量走 DMA 卸载，这是"WiFi 网速上得去"的关键 |

### 2. 与 rfb 公版方案的区别（重要）
H5000M 的 WiFi 节点**写死在设备树**（PCIe0 挂 `mt7992`，eeprom 读 eMMC factory 分区），**不需要**像 rfb 那样靠 U-Boot 注入 DTS overlay。因此：
- **直接 sysupgrade 即可**，原厂 U-Boot 无需更换；
- 刷完验证：SSH 执行 `dmesg | grep -i mt7996` 应看到驱动加载和固件版本；`iw dev` 能看到 `phy0/phy1` 两个无线。

### 3. 性能与稳定性建议（刷机后）
1. LuCI → 网络 → 无线 → 编辑，把 **国家/地区设为 `CN`**（信道与发射功率合规）；
2. 5G 频段选 **160MHz 带宽**（BE 设备才能跑满 5Gbps 级速率；BE5040 双频合计速率按此发挥）；
3. WED 无线 DMA 卸载默认生效（不依赖防火墙分载开关）；若开启科学上网或 Portal 认证，保持防火墙"硬件流量分载"关闭（本固件默认已关），软件分载如遇分流异常也请关闭；纯内网大流量场景可手动开启硬件分载；
4. mt76 驱动已内置 AQL/队列管理，一般无需手动调 QoS；若多设备并发延迟高，可再开 SQM（注意 SQM 与硬件分载互斥，二选一）；
5. 5G 高速负载下注意设备散热，风扇温控插件会按温度自动调速，保持默认阈值即可。

---

## 十、拼拼WiFi（Wiwiz）插件配置与使用

固件已内置拼拼WiFi 开源插件（编译自官方 WiFiPortal 仓库，2026-02 版适配 25.12 体系）。刷机后按以下步骤启用：

### 1. 开通平台账号并创建场所
1. 注册/登录拼拼WiFi平台：http://www.wiwiz.com/pinpinwifi
2. 创建一个"场所"，记下页面上显示的 **Hotspot ID**

### 2. 在路由器上绑定 Hotspot ID（启用收费认证）
1. 登录路由器 LuCI（默认 `192.168.10.1` 或按上面刷机说明）
2. 菜单 **Wiwiz → Portal**，填写 Hotspot ID
3. 勾选"**启用**"，建议同时勾选"**启用 DHCP Captive Portal 通告**"
4. 保存并应用 → **重启一次路由器**
5. 此时访客连接 WiFi 就会弹出认证/付费页面，收益计入拼拼WiFi 平台

### 3. 启用远程管理（DCC2，可选）
1. 打开 DCC2 后台：http://cp.wiwiz.com/ppwf/dcc2（建议 PC 访问）
2. 右上角账号 → "显示 Token"，复制 Token
3. LuCI 菜单 **Wiwiz → DCC2**，填入 Token 并勾选"启用"、保存应用

### 4. 其他组件
- **EQOS 限速**：LuCI → 网络 → EQOS，按 IP 分配上行/下行带宽（收费用户限速常用）
- **Autokick**（到期自动断开）：LuCI → Wiwiz → Autokick 或对应菜单，启用后付费到期用户会被自动断开无线连接
- **免认证网络**：固件已内置 NoPortal 配置，可指定一个网络/接口不经过 Portal 认证（供管理员使用）

### 5. 注意事项（官方说明）
- 外网必须接 **WAN 口**，内网接 LAN 口；LAN 网段不要与上级路由冲突（默认 192.168.10.1，冲突时改为其他网段）
- 认证页面的 HTTPS 证书为自签名，浏览器会提示警告，属正常现象
- 固件默认管理员 `root`、密码为空，**上线前务必设置密码**
- 若认证页不弹出，优先检查：WAN 接线、Hotspot ID 是否绑定、iptables legacy 组件是否正常（本固件已内置 `iptables-zz-legacy`，ImmortalWrt 与 25.12 同包名）

---

## 十一、常见问题

**Q1：编译失败怎么办？**
进 Actions 日志看红色报错。常见原因与处理：
- 网络抖动拉包失败 → 直接重新 Run workflow 即可（有缓存）；
- `MISSING xxx` 校验失败 → 说明某个包名在当前分支不存在，检查 `config.seed` 对应行（改配置后重新触发）。

**Q2：WiFi 没有出现 / 5G 模组不识别？**
- H5000M 的 WiFi 由 DTS 写死（PCIe0 → MT7992），直接 sysupgrade 后应能看到两个 radio；若无线不出现，SSH 执行 `dmesg | grep -i mt7996` 看驱动是否加载、`ls /sys/bus/pci/devices/` 看 PCIe 设备是否枚举（多数是模组挡板/天线接触问题）；
- 5G 模组（鼎桥 MT5700-CN / 移远 RM520N / RM500）需确认插紧且已装 SIM 卡，进 LuCI 看 `内核日志`（状态 → 系统日志）中 USB 设备是否枚举成功（`lsusb` / `dmesg`）；
- 模组被识别为串口 + 网卡是正常的（QMI 模组通常有 `/dev/ttyUSB2` + `wwan0`）；若只出现串口没有网卡，多半是模组处于 AT/存储模式，检查 usb-modeswitch 是否触发（`lsusb` 看 VID/PID 是否变化）；
- 界面查看：LuCI → **Modem → QModem**（巴龙哥 QModem：信号/拨号/锁频/AT/短信）；AT 调试也可在界面内"Debug"页直接下发 AT 命令。

**Q3：5G 高负载发热/风扇不转？**
H5000M 自带 PWM 风扇。固件已集成 `luci-app-h5000m-fancontrol`（依赖 `kmod-hwmon-pwmfan`，设备定义自动带上）。到 LuCI 风扇控制页确认启用并按需调整温度阈值；若风扇完全无反应，检查风扇排线是否插紧。

**Q4：想加其他插件（如 DDNS、下载工具、去广告）？**
两种方式：
1. **在线安装（首选，无需重新编译）**：LuCI → **系统 → 软件包**，搜索包名直接安装（ImmortalWrt 官方源数千软件包可用，本固件已备好 dnsmasq-full/tun/tproxy 等扩展基础）；
2. **编译进固件**：在 `config.seed` 末尾追加对应的 `CONFIG_PACKAGE_xxx=y`，重新上传触发编译。参考：ImmortalWrt 官方包源、lianxia233/OpenWRT-CI-H5000M 的插件清单。
3. 注意：科学上网插件（PassWall2 / OpenClash / Nikki）**同时只启用一个**。

**Q5：科学上网（PassWall2）连不上/个别节点不通？**
1. 先确认 **服务 → PassWall2** 已启用且节点选中（节点列表里可先"测速"）；
2. 确认防火墙页面"硬件流量分载"未勾选（本固件默认已关，若你手动开过请关掉）；
3. 软件分载（flow_offloading）如开着仍异常，取消勾选"软件流量分载"再试；
4. 换 TCP 模式/关闭 UDP 节点、或更换分流规则（"GFW 模式"→"全局模式"）排查；
5. 5G 网络波动时启用**自动切换**（节点列表 → 自动切换）提高稳定性。

**Q6：为什么用 ImmortalWrt 而不是 OpenWrt？**
1. H5000M 是 ImmortalWrt **官方支持设备**（`hiveton_h5000m`），出厂固件就是 ImmortalWrt fork，直接升级最稳；
2. ImmortalWrt 保留更多国产模组/插件生态（含 QModem 适配），本地化与稳定性更好；
3. OpenWrt 25.12 对 H5000M 的支持仍在收尾（DTS 近期才合入），而 ImmortalWrt master 社区 CI 已每日验证。

**Q7：为什么不用国内镜像？**
编译发生在 GitHub 服务器上（海外机房），直连官方源速度很快，无需镜像；只有你下载成品固件是在国内网络，通常也足够快。

---

## 十二、技术备注（供后续排查）

- ImmortalWrt 分支：**master**（内核 6.18 系，跟随上游主线；社区 CI 每日验证 H5000M 可编译）
- 目标配置符号：`CONFIG_TARGET_DEVICE_mediatek_filogic_DEVICE_hiveton_h5000m=y`（设备定义见 `target/linux/mediatek/image/filogic.mk`，DTS 见 `target/linux/mediatek/dts/mt7987a-hiveton-h5000m.dts`）
- 镜像产物：`IMAGE/sysupgrade.bin := sysupgrade-tar | append-metadata`（仅 sysupgrade.bin；配 `CONFIG_TARGET_ROOTFS_INITRAMFS=y` 额外产出 initramfs-kernel.bin 救砖件）
- 硬件加速符号：`CONFIG_NET_MEDIATEK_SOC=y`、`CONFIG_NET_MEDIATEK_SOC_WED=y`（ImmortalWrt 默认）；PPE 由 mtk_eth 内置，无需额外内核选项
- 网口拓扑：eth0 = gmac0 2500base-x 外接 **RTL8221B**（phy@1）；eth1 = gmac1 internal 内置 2.5G PHY（phy@15）；2.5G PHY 固件 `mt7987-2p5g-phy-firmware` 设备自动带上
- 拼拼WiFi 插件源码仓库：GitHub `wiwizcom/WiFiPortal`（国内备用镜像：gitee `wiwiz/WiFiPortal`）。若 Actions 中 GitHub 克隆失败，把 workflow 里对应行换成 gitee 地址即可
- 5G 界面源码：**QModem**（github.com/FUjr/QModem，feed 名 `qmodem`）；许可证：核心 MPL 2.0 并附"禁止商业使用"条款（自用无碍，商用需联系作者），UI 包 GPLv3；若需移远 QMAP 多路加速（RM520N 突破单路速率上限），可在 qmodem 包菜单启用 `qmodem_USE_TOM_CUSTOMIZED_QUECTEL_CM`（编入 quectel-CM-5G-M 拨号工具 + 厂商 QMI 驱动），本固件默认使用官方 qmi_wwan（单路，日常足够）
- 科学上网源码：**PassWall2**（github.com/Openwrt-Passwall/openwrt-passwall2，feed 名 `passwall2`，依赖包 feed `passwall_packages`：xray-core / sing-box / hysteria / naiveproxy / shadowsocks 全系 / chinadns-ng 等）；许可证 GPL-2.0；核心 xray-core/sing-box 为 Go 交叉编译（首次编译会多耗时约 30~60 分钟，属正常）；如需 OpenClash 可在线安装（与 PassWall2 二选一）
- H5000M 专属插件（FAN789）：`luci-app-h5000m-fancontrol`（v2.1.0，依赖 kmod-hwmon-pwmfan）、`luci-app-h5000m-netmode`（v1.3.1，配 mwan3）；另有 `luci-app-mt5700m`（MT5700M 专用界面，与本固件 QModem 功能重叠，未集成，需要可自行加）
- 参考仓库：lianxia233/OpenWRT-CI-H5000M（鼎桥 H5000M ImmortalWrt 定制固件 CI，每日自动编译，含 README 与完整配置可对照）
