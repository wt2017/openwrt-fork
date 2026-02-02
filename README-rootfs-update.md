# OpenWrt RootFS 更新指南

## 概述

当需要更新 rootfs 分区时，OpenWrt 提供了多种方法。根据设备类型和存储介质的不同，更新 rootfs 的方法也有所不同。本指南将详细说明如何将新的 rootfs 覆盖原来的 rootfs。

## 1. 系统升级 (sysupgrade) - 推荐方法

### 1.1 使用 sysupgrade 命令

对于大多数 OpenWrt 设备，推荐使用 `sysupgrade` 命令进行系统升级，它会自动处理 rootfs 的更新。

```bash
# 基本用法
sysupgrade /path/to/firmware.bin

# 保留配置文件
sysupgrade -n /path/to/firmware.bin

# 交互模式
sysupgrade -i /path/to/firmware.bin

# 测试模式（不实际刷写）
sysupgrade -T /path/to/firmware.bin
```

### 1.2 sysupgrade 工作流程

1. **验证固件**：检查固件文件的完整性和兼容性
2. **备份配置**：创建当前配置的备份（默认保存在 `/tmp/sysupgrade.tgz`）
3. **准备升级**：将固件复制到临时位置
4. **执行升级**：调用平台特定的升级函数或使用默认升级方法
5. **重启系统**：升级完成后自动重启

### 1.3 关键文件位置

- **sysupgrade 脚本**: `/sbin/sysupgrade`
- **升级库文件**: `/lib/upgrade/`
- **配置文件备份**: `/etc/sysupgrade.conf`

## 2. SD卡设备的 rootfs 更新

对于 Raspberry Pi 等使用 SD 卡的设备，rootfs 更新有特殊处理。

### 2.1 SD卡镜像结构

SD卡镜像通常包含两个分区：
1. **boot分区** (FAT32): 包含内核、设备树和启动文件
2. **rootfs分区** (ext4/squashfs): 包含根文件系统

### 2.2 生成 SD卡镜像

OpenWrt 使用以下流程生成 SD卡镜像：

```bash
# 从 Makefile 中可以看到镜像构建链
IMAGE/factory.img.gz := boot-common | sdcard-img | gzip
```

关键脚本：`target/linux/bcm27xx/image/gen_rpi_sdcard_img.sh`

### 2.3 手动更新 rootfs 分区

如果需要手动更新 rootfs 分区，可以按照以下步骤：

#### 方法一：使用 dd 命令直接写入

```bash
# 1. 解压固件镜像
gunzip openwrt-bcm27xx-bcm2712-rpi-5-ext4-factory.img.gz

# 2. 查看分区信息
fdisk -l openwrt-bcm27xx-bcm2712-rpi-5-ext4-factory.img

# 3. 提取 rootfs 分区
# 计算 rootfs 分区的偏移量（单位：字节）
# 假设 rootfs 从第 12582912 扇区开始，每个扇区 512 字节
offset=$((12582912 * 512))

# 4. 将 rootfs 写入 SD卡
sudo dd if=openwrt-bcm27xx-bcm2712-rpi-5-ext4-factory.img of=/dev/sdX2 bs=1M skip=$((offset/1048576)) conv=fsync
```

#### 方法二：使用 losetup 挂载后复制

```bash
# 1. 创建 loop 设备
sudo losetup -fP openwrt-bcm27xx-bcm2712-rpi-5-ext4-factory.img

# 2. 查看创建的 loop 设备
losetup -a

# 3. 挂载 rootfs 分区
sudo mount /dev/loop0p2 /mnt

# 4. 清空目标 rootfs 分区并复制文件
sudo rm -rf /target/rootfs/*
sudo cp -a /mnt/* /target/rootfs/

# 5. 卸载并清理
sudo umount /mnt
sudo losetup -d /dev/loop0
```

## 3. 嵌入式设备的 rootfs 更新

对于使用 NAND/SPI Flash 的嵌入式设备，通常使用 mtd 工具。

### 3.1 使用 mtd 工具

```bash
# 查看 mtd 分区
cat /proc/mtd

# 备份当前 rootfs
dd if=/dev/mtdblockX of=/tmp/rootfs_backup.bin

# 写入新的 rootfs
mtd write /path/to/new_rootfs.bin rootfs

# 或者使用 sysupgrade 的底层函数
default_do_upgrade() {
    get_image "$1" | mtd write - "${PART_NAME:-image}"
}
```

### 3.2 双分区系统

许多 OpenWrt 设备使用双分区系统（A/B 分区）来实现无缝升级：

```bash
# 查看当前活动分区
fw_printenv

# 切换分区
fw_setenv bootpart 2
```

## 4. 运行时 rootfs 更新策略

### 4.1 覆盖挂载 (overlay)

OpenWrt 使用 overlayfs 来允许运行时修改只读的 rootfs：

```
/overlay (读写层)
  └── /rom (只读层，原始 rootfs)
```

### 4.2 更新 overlay

要更新 overlay 中的文件：

```bash
# 1. 创建新的 overlay 目录
mkdir -p /new_overlay

# 2. 复制当前配置
cp -a /overlay/* /new_overlay/

# 3. 应用更新
# （根据具体更新需求修改 /new_overlay 中的文件）

# 4. 切换 overlay
mount -o move /overlay /old_overlay
mount -t overlay overlay -o lowerdir=/rom,upperdir=/new_overlay /overlay
```

## 5. 构建系统中的 rootfs 处理

### 5.1 rootfs 生成流程

在 OpenWrt 构建系统中，rootfs 的生成涉及以下步骤：

1. **准备目标目录**：`target-dir-%` 规则
2. **生成文件系统镜像**：`Image/mkfs/` 函数族
3. **创建完整镜像**：设备特定的构建链

### 5.2 关键 Makefile 函数

```makefile
# ext4 文件系统生成
define Image/mkfs/ext4
    $(STAGING_DIR_HOST)/bin/make_ext4fs -L rootfs \
        -l $(ROOTFS_PARTSIZE) -b $(CONFIG_TARGET_EXT4_BLOCKSIZE) \
        $@ $(call mkfs_target_dir,$(1))/
endef

# squashfs 文件系统生成  
define Image/mkfs/squashfs
    $(STAGING_DIR_HOST)/bin/mksquashfs4 $(call mkfs_target_dir,$(1)) $@ \
        -nopad -noappend -root-owned \
        -comp $(SQUASHFSCOMP) $(SQUASHFSOPT)
endef
```

## 6. 故障排除

### 6.1 常见问题

1. **空间不足**：确保新的 rootfs 不超过分区大小
2. **文件系统损坏**：更新后运行 `fsck` 检查
3. **启动失败**：保留旧版本备份，支持回滚

### 6.2 调试技巧

```bash
# 启用详细输出
sysupgrade -v /path/to/firmware.bin

# 查看升级日志
logread | grep upgrade

# 检查分区布局
fdisk -l /dev/sdX
parted /dev/sdX print
```

## 7. 安全注意事项

1. **始终备份**：更新前备份当前配置和重要数据
2. **验证固件**：确保固件来源可信且完整
3. **电源稳定**：更新过程中保持电源稳定
4. **测试环境**：先在测试设备上验证更新流程

## 8. 自动化脚本示例

### 8.1 自动更新脚本

```bash
#!/bin/bash
# auto_update_rootfs.sh

set -e

FIRMWARE_URL="https://downloads.openwrt.org/releases/xx.xx.xx/target/device/firmware.bin"
BACKUP_DIR="/tmp/backup_$(date +%Y%m%d_%H%M%S)"

# 创建备份
mkdir -p "$BACKUP_DIR"
sysupgrade -b "$BACKUP_DIR/config.tar.gz"

# 下载固件
wget -O /tmp/firmware.bin "$FIRMWARE_URL"

# 验证固件
if /usr/libexec/validate_firmware_image /tmp/firmware.bin; then
    # 执行升级
    sysupgrade -n /tmp/firmware.bin
else
    echo "固件验证失败"
    exit 1
fi
```

## 总结

更新 OpenWrt 的 rootfs 有多种方法，选择哪种方法取决于：

1. **设备类型**：SD卡设备、NAND设备、eMMC设备等
2. **使用场景**：生产环境、开发测试、批量部署等
3. **风险承受**：是否需要保留配置、是否支持回滚等

对于大多数用户，推荐使用 `sysupgrade` 命令，它提供了最完整和最安全的升级流程。对于特殊需求或开发调试，可以手动操作分区和文件系统。