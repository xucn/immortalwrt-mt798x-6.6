# ImmortalWrt MT798x 6.6 编译指南（WSL 环境）

适用于 Ruijie RG-X60 路由器，基于 immortalwrt-mt798x-6.6 项目（ImmortalWrt 24.10 / Kernel 6.6）。

---

## 一、WSL 环境准备

### 1.1 创建编译用户

OpenWrt/ImmortalWrt **禁止使用 root 用户编译**，WSL 默认是 root，需要先创建普通用户。

```bash
# 创建用户，设置家目录和 shell
useradd -m -s /bin/bash builder
passwd builder

# 赋予 sudo 权限（安装依赖时需要）
usermod -aG sudo builder
```

**（可选）修改 WSL 默认登录用户：**

在 Windows PowerShell 中执行：
```powershell
# 将 Ubuntu 替换为你的实际发行版名称
ubuntu config --default-user builder
```

这样以后打开 WSL 就直接是 builder 用户，不需要每次 `su`。

### 1.2 切换到编译用户

```bash
su - builder
```

后续所有操作均在 builder 用户下执行。

### 1.3 WSL 特别注意事项

- **路径不能包含空格或中文**
- **必须移除 PATH 中的 Windows 路径**，否则编译会出错：
  ```bash
  # 在 builder 用户的 ~/.bashrc 末尾添加
  echo 'export PATH=$(echo "$PATH" | tr ":" "\n" | grep -v /mnt/ | tr "\n" ":")' >> ~/.bashrc
  source ~/.bashrc
  ```
  或者修改 WSL 配置彻底禁用 Windows PATH 注入：
  ```bash
  # 用 root 执行
  sudo tee /etc/wsl.conf <<EOF
  [interop]
  appendWindowsPath = false
  EOF
  ```
  修改后需要在 PowerShell 中执行 `wsl --shutdown` 重启 WSL 生效。

- **文件系统要求区分大小写**，WSL2 的 ext4 文件系统默认满足

## 二、安装编译依赖

```bash
sudo apt update -y
sudo apt full-upgrade -y

# 方法一：一键脚本（推荐）
sudo bash -c 'bash <(curl -s https://build-scripts.immortalwrt.org/init_build_environment.sh)'

# 方法二：手动安装
sudo apt install -y ack antlr3 asciidoc autoconf automake autopoint binutils bison build-essential \
  bzip2 ccache clang cmake cpio curl device-tree-compiler ecj fastjar flex gawk gettext gcc-multilib \
  g++-multilib git gnutls-dev gperf haveged help2man intltool lib32gcc-s1 libc6-dev-i386 libelf-dev \
  libglib2.0-dev libgmp3-dev libltdl-dev libmpc-dev libmpfr-dev libncurses-dev libpython3-dev \
  libreadline-dev libssl-dev libtool libyaml-dev libz-dev lld llvm lrzsz mkisofs msmtp nano \
  ninja-build p7zip p7zip-full patch pkgconf python3 python3-pip python3-ply python3-docutils \
  python3-pyelftools qemu-utils re2c rsync scons squashfs-tools subversion swig texinfo uglifyjs \
  upx-ucl unzip vim wget xmlto xxd zlib1g-dev zstd
```

## 三、获取源码

如果尚未下载源码：
```bash
git clone -b openwrt-24.10-6.6 --single-branch --filter=blob:none \
  https://github.com/padavanonly/immortalwrt-mt798x-24.10 immortalwrt-mt798x-6.6
cd immortalwrt-mt798x-6.6
```

如果源码已在 `/e/mt798x/immortalwrt-mt798x-6.6`，需确保 builder 用户有权限：
```bash
# 用 root 执行
sudo chown -R builder:builder /e/mt798x/immortalwrt-mt798x-6.6

# 切回 builder
su - builder
cd /e/mt798x/immortalwrt-mt798x-6.6
```

## 四、更新 Feeds

```bash
./scripts/feeds update -a
./scripts/feeds install -a
```

feeds 是 ImmortalWrt 的软件包索引，包含 LuCI、路由协议包、常用工具等。此步骤需要联网下载。

## 五、配置

### 5.1 使用预设配置

X60 属于 MT7986 平台，使用 ax6000 配置：

```bash
cp -f defconfig/mt7986-ax6000.config .config
```

此配置已包含 `ruijie_rg-x60-new` 设备支持。

### 5.2 自定义配置（可选）

```bash
make menuconfig
```

在菜单中可以：
- **Target System** → MediaTek Ralink ARM（已选）
- **Subtarget** → Filogic 8x0 (MT798x)（已选）
- **Target Profile** → 可查看/选择具体设备
- **LuCI** → 选择需要的 Web 界面插件
- **Network** → 选择需要的网络工具
- **Kernel modules** → 检查驱动模块

退出时选择 Save 保存配置。

### 5.3 可用的 defconfig 说明

| 配置文件 | 适用平台 |
|---------|---------|
| `mt7986-ax6000.config` | MT7986 设备（X60、X60 Pro、红米 AX6000 等） |
| `mt7981-ax3000.config` | MT7981 设备（AX3000 级别） |
| `mt7986-ax4200-bpir3_mini.config` | BPI-R3 Mini 开发板 |

## 六、编译

### 6.1 首次编译

```bash
# 先下载所有源码（可选，避免编译中途下载失败）
make download -j$(nproc)

# 首次编译建议单线程，方便定位错误
make -j1 V=s 2>&1 | tee build.log
```

- `-j1`：单线程编译
- `V=s`：详细输出，显示每条编译命令
- `tee build.log`：同时输出到屏幕和日志文件

### 6.2 后续编译

```bash
# 多线程编译，N = CPU 核心数
make -j$(nproc)
```

### 6.3 编译耗时参考

| 阶段 | 首次编译 | 增量编译 |
|------|---------|---------|
| 下载源码 | 10-30 分钟（取决于网速） | 跳过 |
| 编译工具链 | 30-60 分钟 | 跳过 |
| 编译内核+软件包 | 30-90 分钟 | 5-15 分钟 |
| **总计** | **1-3 小时** | **5-15 分钟** |

### 6.4 编译失败排查

```bash
# 查看错误日志
grep -i error build.log | tail -20

# 清理后重试
make clean          # 清理编译产物，保留工具链
make -j1 V=s        # 单线程重新编译

# 如果工具链也有问题
make dirclean       # 清理一切，包括工具链（需要完全重编）
```

## 七、编译产物

固件输出目录：

```
bin/targets/mediatek/filogic/
```

### 7.1 X60 相关固件文件

| 文件名包含 | 用途 |
|-----------|------|
| `ruijie_rg-x60-new-squashfs-sysupgrade.bin` | 从已有 OpenWrt 升级 |
| `ruijie_rg-x60-new-squashfs-factory.bin` | 从 U-Boot 刷入 |

### 7.2 查看编译产物

```bash
ls -lh bin/targets/mediatek/filogic/*x60*
```

## 八、刷写固件

### 8.1 从已有 OpenWrt 升级

```bash
# 上传固件到路由器
scp bin/targets/mediatek/filogic/*x60*sysupgrade.bin root@192.168.1.1:/tmp/

# SSH 登录路由器执行升级
ssh root@192.168.1.1
sysupgrade /tmp/*sysupgrade.bin
```

### 8.2 从 U-Boot Web Failsafe 刷入

1. 电脑网线连接路由器 LAN 口
2. 路由器断电，按住 Reset 按键后上电
3. 持续按住约 5-10 秒，等待进入 Failsafe 模式
4. 如果使用 bl-mt798x-dhcpd 的 U-Boot，电脑会自动获取 IP
5. 浏览器访问 U-Boot Web 界面（通常 192.168.1.1）
6. 上传 factory.bin 刷入

### 8.3 从 U-Boot 命令行刷入（串口）

```
# 电脑开启 TFTP 服务，将 factory.bin 放入 TFTP 根目录
setenv ipaddr 192.168.1.1
setenv serverip 192.168.1.2
tftpboot 0x46000000 factory.bin
ubi write 0x46000000 firmware ${filesize}
reset
```

## 九、常见问题

### Q: 编译报错 `do not compile as root`
A: 切换到普通用户，参见第一节。

### Q: 编译报错找不到命令，提示 Windows 路径相关
A: WSL 中 Windows PATH 干扰了编译，参见 1.3 节移除 Windows PATH。

### Q: feeds update 下载失败
A: 网络问题，考虑配置代理：
```bash
export https_proxy=http://127.0.0.1:7890
export http_proxy=http://127.0.0.1:7890
```

### Q: make download 部分源码下载失败
A: 重复执行 `make download -j1 V=s`，或手动下载放入 `dl/` 目录。

### Q: 编译后找不到 X60 固件
A: 确认 `.config` 中包含 X60 设备：
```bash
grep -i x60 .config
```
应看到 `CONFIG_TARGET_DEVICE_mediatek_filogic_DEVICE_ruijie_rg-x60-new=y`。

### Q: X60 和 X60 New 选哪个？
A: OpenWrt 固件层面两者硬件定义完全相同，选 `ruijie_rg-x60-new` 即可，DDR3/DDR4 旧版新版通用。DDR 类型只影响 BL2/ATF 编译，不影响 OpenWrt 固件。
