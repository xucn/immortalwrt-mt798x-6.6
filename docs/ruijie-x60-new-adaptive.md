# 锐捷 X60 New (Ruijie RG-X60 New) OpenWRT/ImmortalWRT 适配文档

## 硬件规格

- **SoC**: MediaTek MT7986A (Filogic 830)
- **无线**: MT7986A 集成 Wi-Fi 6 (2.4G + 5G)
- **内存**: 512MB DDR
- **闪存**: SPI NAND
- **网络**:
  - LAN: 4x 1Gbps (通过 MT7531 交换机)
  - WAN: 1x 2.5Gbps (通过 Airoha EN8811H PHY)

## 问题描述

### 首次适配问题

首次适配时，WAN口 (eth1) 没有任何流量，EN8811H PHY 驱动已加载但 MAC 无法挂载。

**错误日志**:
```
[   23.730182] mtk_soc_eth eth1: validation of rgmii with support 00,00000000,00008000,000062e8 failed: -EINVAL
[   23.760185] mtk_soc_eth eth1: mtk_open: could not attach PHY: -22
```

### 原因分析

1. **错误的 phy-mode**: 设备树中使用了 `phy-mode = "rgmii"`，但 RGMII 接口最高只支持 1Gbps
2. **EN8811H 是 2.5G PHY**: 需要使用 2500base-x 模式才能正常工作

## 解决方案

### 1. 设备树配置 (关键修改)

**文件**: `target/linux/mediatek/dts/mt7986a-ruijie-rg-x60-new.dts`

#### gmac1 配置 (WAN口)

```dts
gmac1: mac@1 {
    compatible = "mediatek,eth-mac";
    reg = <1>;
    phy-mode = "2500base-x";

    fixed-link {
        speed = <2500>;
        full-duplex;
        pause;
    };
};
```

#### EN8811H PHY 配置

```dts
phy15: phy@f {
    compatible = "ethernet-phy-id03a2.a411";
    reg = <15>;
    reset-gpios = <&pio 6 GPIO_ACTIVE_LOW>;
    reset-assert-us = <10000>;
    reset-deassert-us = <20000>;
    phy-mode = "2500base-x";
    full-duplex;
    pause;
};
```

**关键点**:
- `phy-mode` 必须使用 `2500base-x`，不能使用 `rgmii`
- 使用 `fixed-link` 固定 2.5Gbps 速率
- PHY 节点也需要配置 `phy-mode = "2500base-x"`

### 2. EN8811H 驱动修改

**文件**: `target/linux/mediatek/files-6.6/drivers/net/phy/air_en8811h_main.c`

修改 `en8811h_get_features` 函数：

```c
static int en8811h_get_features(struct phy_device *phydev)
{
    // ... (省略部分代码)

    linkmode_zero(phydev->supported);
    /* EN8811H supports 100M/1G/2.5G speed. */
    linkmode_set_bit(ETHTOOL_LINK_MODE_Autoneg_BIT, phydev->supported);
    linkmode_set_bit(ETHTOOL_LINK_MODE_TP_BIT, phydev->supported);
    linkmode_set_bit(ETHTOOL_LINK_MODE_MII_BIT, phydev->supported);
    linkmode_set_bit(ETHTOOL_LINK_MODE_100baseT_Full_BIT, phydev->supported);
    linkmode_set_bit(ETHTOOL_LINK_MODE_1000baseT_Full_BIT, phydev->supported);
    linkmode_set_bit(ETHTOOL_LINK_MODE_2500baseT_Full_BIT, phydev->supported);
    linkmode_set_bit(ETHTOOL_LINK_MODE_Pause_BIT, phydev->supported);
    linkmode_set_bit(ETHTOOL_LINK_MODE_Asym_Pause_BIT, phydev->supported);

    linkmode_copy(phydev->advertising, phydev->supported);

    return 0;
}
```

**修改说明**:
- 清除原有 capabilities 并重新设置
- 添加 `2500baseT_Full` 支持
- 确保自动协商和暂停帧正确配置

### 3. 内核配置

确保以下配置已启用:

```
CONFIG_AIR_EN8811H_PHY=y
```

### 4. 其他修改

- **defconfig**: 添加 `CONFIG_TARGET_DEVICE_mediatek_filogic_DEVICE_ruijie_rg-x60-new=y`
- **netdevices.mk**: 修正驱动模块名 `air_en8811` (原为 `air_en8811h`)

## 参考设备

| 设备 | PHY | 配置方式 | 状态 |
|------|-----|----------|------|
| Ruijie EW-6000GX Pro | EN8811H | 2500base-x + fixed-link | ✅ 正常 |
| BPI-R3 Mini | EN8811H | 2500base-x + fixed-link | ✅ 正常 |
| Ruijie X60 Pro | Realtek | 2500base-x + phy-handle | ✅ 正常 |

## 关于 2.5G WAN 与 1G 上级设备

当前配置使用 `fixed-link` 固定 2.5Gbps 速率。如果上级路由器/光猫是千兆 (1G) 设备：

### 方案 A: 固定 2.5G (当前方案)
- 适用于上级设备有 2.5G 口
- 稳定可靠

### 方案 B: 使用 phy-handle (phy-mode 仍为 2500base-x)
- 适用于上级设备支持自动协商
- 可能支持 1G/2.5G 自适应

## 编译与刷机

### 编译命令

```bash
# 拷贝修改后的文件到编译目录
cp target/linux/mediatek/dts/mt7986a-ruijie-rg-x60-new.dts ~/immortalwrt/target/linux/mediatek/dts/

# 编译
cd ~/immortalwrt
make -j$(nproc)
```

### 固件位置

编译完成后在:
```
~/immortalwrt/bin/targets/mediatek/filogic/
```

### 刷机

通过 UBoot 或厂商固件升级界面上传固件。

## 验证

### 检查驱动加载

```bash
dmesg | grep -i air
```

预期输出:
```
[    1.170430] Airoha EN8811H mdio-bus:0f: PHY = 3a2 - a411
[    3.613900] Airoha EN8811H mdio-bus:0f: EN8811H initialize OK! (v1.2.5)
```

### 检查网络接口

```bash
ifconfig eth1
```

预期: eth1 显示 2.5Gbps 全双工速率

### 测试连通性

```bash
ping -I eth1 8.8.8.8
```

## 常见问题

### Q: 驱动加载成功但仍无流量?

A: 检查:
1. 上级设备是否为 2.5G 口
2. 网线是否为超五类或六类
3. 尝试重新插拔网线

### Q: 如何查看 PHY 状态?

A:
```bash
ethtool eth1
```

## 更新日志

- **2026-03-14**: 首次适配，解决 EN8811H WAN 口无流量问题
