# Ruijie RG-X60 vs RG-X60 Pro 硬件差异对比文档

## 一、芯片级硬件对比

| 组件 | RG-X60 (New) | RG-X60 Pro |
|------|-------------|------------|
| **SoC** | MT7986A | MT7986A |
| **内存** | W634GU6QB (Winbond **DDR3** 512MB) | DDR4 512MB |
| **闪存** | W25N01GVZEIG (Winbond SPI NAND 128MB/1Gbit) | SPI NAND 128MB |
| **2.5G WAN PHY** | **Airoha EN8811HN** (PHY ID: 03a2.a411, MDIO地址15) | **Realtek RTL8221B** (MDIO地址7) |
| **千兆交换芯片** | MT7531A | MT7531 |
| **2.4G WiFi** | MT7975N | MT7975N |
| **5G WiFi** | MT7975PN | MT7975PN |

### 关键差异说明

1. **内存类型不同 (最关键)**：X60 使用 DDR3，X60 Pro 使用 DDR4。这意味着 ATF/BL2 配置必须使用 `_DRAM_DDR3=y`，否则无法启动。
2. **2.5G PHY 不同**：X60 使用 Airoha EN8811HN，X60 Pro 使用 Realtek RTL8221B。需要不同的内核驱动。
3. **WiFi 和交换芯片相同**：两者均使用 MT7975N/PN + MT7531(A)。

## 二、GPIO 引脚映射对比

| 功能 | RG-X60 (New) | RG-X60 Pro |
|------|-------------|------------|
| **复位按键** | GPIO 14 | GPIO 9 |
| **Mesh 按键** | GPIO 15 | GPIO 10 |
| **交换芯片复位** | GPIO 5 (HIGH) | GPIO 5 (HIGH) |
| **PHY 复位** | GPIO 6 (LOW) | GPIO 6 (LOW) |

### LED 对比

| LED | RG-X60 (New) | RG-X60 Pro |
|-----|-------------|------------|
| LED 0 | **红色** - GPIO 10 | **白色** - GPIO 22 |
| LED 1 | **绿色** - GPIO 9 | **紫色** - GPIO 11 |
| LED 2 | **蓝色** - GPIO 11 | 无 |
| LED 3 (Mesh) | GPIO 22 | 无 |

X60 有 RGB 三色状态灯 + 独立 Mesh 指示灯（共4个），X60 Pro 只有白色+紫色两个状态灯。

## 三、分区布局对比

两者分区起始地址和大小基本一致：

| 分区 | 偏移 | 大小 | 说明 |
|------|------|------|------|
| BL2 | 0x000000 | 1MB | 只读 |
| u-boot-env | 0x100000 | 512KB | 只读 |
| Factory | 0x180000 | 2MB | 只读，含 WiFi 校准数据 |
| FIP | 0x380000 | 2MB | 含 U-Boot + ATF |
| product_info | 0x580000 | 512KB | 只读，含 MAC 地址 |
| kdump | 0x600000 | 512KB | 只读 |
| ubi | 0x680000 | 见下文 | OpenWrt 系统 |

### UBI 分区大小

| 变体 | UBI 大小 | 说明 |
|------|---------|------|
| **stock** | 0x3F00000 (~63MB) | 保留原厂分区布局，与原厂双系统分区兼容 |
| **非 stock** | 0x6B00000 (~107MB) | 重新分区后使用闪存全部可用空间 |

## 四、Stock 与非 Stock 的区别

### Stock 版本
- **分区布局**：保持与原厂固件一致的分区表（UBI 约 63MB）
- **用途**：首次从原厂固件刷入 OpenWrt 时使用
- **优点**：可以直接通过原厂升级机制刷入，不需要改动分区表
- **缺点**：可用空间较小（~63MB）

### 非 Stock 版本
- **分区布局**：重新规划分区，UBI 扩展到约 107MB
- **用途**：在已经刷入 OpenWrt 并完成分区调整后使用
- **优点**：充分利用 128MB 闪存空间
- **前提**：需要先通过 stock 版本刷入，然后在 OpenWrt 中调整分区后再刷非 stock 版本

### 刷机流程
1. 原厂固件 → 刷入 **stock** 版本（保持原厂分区布局）
2. 进入 OpenWrt → 调整分区/重新分区
3. 刷入**非 stock** 版本（使用完整空间）

## 五、BL-MT798X vs BL-MT798X-DHCPD 对比

### bl-mt798x（基础版）

| 特性 | 说明 |
|------|------|
| X60 支持 | 有（`mt7986_ruijie_rg-x60`） |
| X60 New 支持 | **无** |
| U-Boot 版本 | 20220606, 20230718 |
| ATF 版本 | 20220606, 20240117 |
| 构建变体 | 仅默认 |
| DHCP 服务器 | **无** |
| Web Failsafe | 基础 |
| FIT 支持 | 无 |
| 配置目录 | 仅 configs/ |

### bl-mt798x-dhcpd（增强版，by Yuzhii0718）

| 特性 | 说明 |
|------|------|
| X60 支持 | 有 |
| X60 New 支持 | **有**（含 DDR3 ATF 配置） |
| U-Boot 版本 | 20220606 ~ 20250711 |
| ATF 版本 | 20220606 ~ 20250711 |
| 构建变体 | default / fit / nonmbm |
| DHCP 服务器 | **有** (mtk_dhcpd) |
| Web Failsafe | 增强（自带 DHCP） |
| FIT 支持 | 有 |
| 配置目录 | configs/ + configs-fit/ + configs-nonmbm/ |

### 关键区别

1. **DHCP 服务器**：bl-mt798x-dhcpd 在 U-Boot 中内置了 DHCP 服务器（`mtk_dhcpd.c`），当进入 Web Failsafe 模式时，会自动给连接的电脑分配 IP 地址。使用 bl-mt798x 时，你需要手动配置电脑的静态 IP 才能访问 Failsafe 页面。

2. **X60 New DDR3 支持**：**只有 bl-mt798x-dhcpd 有 x60-new 的 ATF 配置**（`_DRAM_DDR3=y`）。bl-mt798x 没有，因此 **必须使用 bl-mt798x-dhcpd** 来为 X60 编译 U-Boot。

3. **多构建变体**：
   - **default**：标准 NMBM + MTD 环境存储
   - **fit**：FIT 镜像格式 + UBI 环境存储，FIP 和 RF 数据存入 MTD 分区
   - **nonmbm**：不使用 NMBM 坏块管理，使用 skip-bad 方式

### 结论：X60 应使用 bl-mt798x-dhcpd

原因：
- 这是唯一包含 X60 New DDR3 配置的 U-Boot 项目
- 内置 DHCP 服务器让 Failsafe 救砖更方便
- 版本更新，支持更多功能（FIT、nonmbm 等）

## 六、BL2 和 FIP 刷写顺序问题

### 正常刷写顺序
标准流程是：**先刷 BL2（位于闪存起始位置），再刷 FIP**。

BL2 是芯片上电后执行的第一段代码（存储在闪存 0x0 位置），它负责：
1. 初始化 DRAM（DDR3/DDR4）
2. 加载并验证 FIP（含 BL31 + U-Boot）

FIP 包含：
- BL31（ARM Trusted Firmware Runtime）
- BL33（U-Boot）

### "先刷 FIP，再在 FIP 下刷 BL2" 是否可行？

**可以，但有条件**：

- **前提**：当前正在运行的 U-Boot（旧 FIP）能正常工作
- **流程**：
  1. 在当前 U-Boot 中先刷新 FIP 分区（0x380000）
  2. 重启 → 旧 BL2 加载新 FIP → 进入新 U-Boot
  3. 在新 U-Boot 中刷新 BL2 分区（0x0）
  4. 重启 → 新 BL2 + 新 FIP 完整生效

- **风险**：
  - 如果新 FIP 与旧 BL2 不兼容（例如 DRAM 初始化参数变化），第二步可能无法启动
  - BL2 和 FIP 通常是配套编译的，建议尽量一起刷新
  - **最安全的做法**：如果有 UART/串口访问，先刷 BL2 再刷 FIP；如果只能通过 Web Failsafe，先刷 FIP 风险较低（因为 BL2 不变，至少能保证启动到 U-Boot）

- **实际操作建议**：
  - 如果从原厂 U-Boot 升级：先刷 FIP（因为原厂 BL2 能加载新 FIP），确认新 U-Boot 正常后再刷 BL2
  - 如果从已有 OpenWrt U-Boot 升级：BL2 和 FIP 一起刷最安全
  - 刷 BL2 前确保有串口连接作为后备，因为 BL2 损坏 = 完全变砖（只能通过编程器恢复）

## 七、当前代码硬件支持验证

### EN8811HN PHY 驱动
| 项目 | 驱动状态 | 包名 |
|------|---------|------|
| immortalwrt-mt798x (5.4) | 有（通过 patch 添加） | `kmod-phy-air-en8811h` |
| immortalwrt-mt798x-6.6 | 有（完整驱动 + 固件） | `kmod-phy-airoha-en8811h` |

DTS 中 PHY 配置（两个项目均已正确配置）：
- PHY ID: `03a2.a411`
- MDIO 地址: 15 (0xf)
- C45 接口

### MT7531A 千兆交换芯片
两个项目均完整支持 MT7531 驱动，DTS 中配置正确。

### MT7975N/PN WiFi
- 5.4 项目：使用 MTK 闭源驱动（`mt_wifi`）
- 6.6 项目：支持 mt7915e 开源驱动 + MTK 闭源驱动，DTS 中通过 `&wifi` 或 `&wbsys` 节点启用

### W634GU6QB DDR3 内存
- OpenWrt 内核层面不需要特别配置（DTS 中定义 memory 大小即可）
- **ATF/BL2 层面必须使用 DDR3 配置**（bl-mt798x-dhcpd 中已有）

### W25N01GVZEIG SPI NAND
两个项目均支持 Winbond SPI NAND 系列：
- 5.4：通过 backport patch 支持
- 6.6：原生支持 + 额外 patch
- DTS 中 SPI NAND 频率配置为 52MHz（5.4）/ 20MHz（6.6），兼容 W25N01GVZEIG 的最大 104MHz
