# Debian for iceland

为 **一加平板 3 Pro（OnePlus Pad 3 Pro，OPD2513）** 与 **一加平板 4（OnePlus Pad 4，OPD2514）** 构建 Debian 根文件系统与内核的 GitHub Actions 工程。设备代号 **iceland**，SoC 为 **SM8850（kaanapali）**。

思路参考 [debian-sheng-kxt](https://github.com/rrrRex1336/debian-sheng-kxt)：在 GitHub Actions 里从零 debootstrap 一个 Debian arm64 根文件系统，编译主线内核，自动下载设备固件，最后产出可直接 `fastboot flash` 的镜像。

> **状态：早期开发中，尚未在真机上验证启动。** 第一版把构建链跑通，boot 参数和分区细节需要真机反馈后调整。

## 产物

一次构建会产出：

| 文件 | 说明 |
| --- | --- |
| `rootfs.img` | ext4 根文件系统，按 `PARTLABEL` 挂载根分区 |
| `boot.img` | Android boot header v2，含内核 `Image.gz`、DTB、Debian initramfs |
| `esp.img` | FAT32 EFI 分区，含内核、DTB、initramfs、systemd-boot 与 `extlinux.conf` |
| `linux-iceland.deb` | 内核与模块的 Debian 包 |
| `firmware-iceland.deb` | 从固件仓库自动打包的固件 |

`boot.img` 与 `esp.img` 二选一，由 `boot_method` 参数控制。

## 使用方法

1. Fork 本仓库（Actions 需要在你的仓库里运行）。
2. 打开 **Actions → Build Debian → Run workflow**，按需选择参数。
3. 构建完成后在 run 页面底部下载 artifacts。
4. 按下面的刷机步骤写入设备。

如果不想公开构建产物，把仓库设为 private 即可（arm64 runner 对 private 仓库会计费）。

### 参数

| 参数 | 说明 | 默认 |
| --- | --- | --- |
| `distro_suite` | Debian 版本，`trixie`（13）/ `forky`（14） | `trixie` |
| `desktop` | 桌面环境：`KDE Plasma` / `GNOME` / `server` | `KDE Plasma` |
| `plasma_mobile` | 用 `plasma-mobile` 代替 `plasma-desktop` | `false` |
| `autologin` | 自动登录 | `true` |
| `username` | 用户名 | `user` |
| `hostname` | 主机名 | `iceland` |
| `language` | 系统语言 | `zh_CN.UTF-8` |
| `boot_mode` | 根分区：`dual (linux)` / `single (userdata)` / `custom` | `dual (linux)` |
| `custom_partition` | `boot_mode=custom` 时的分区名 | 空 |
| `boot_method` | `android_boot` / `uefi_esp` / `none` | `android_boot` |
| `quiet_boot` | Plymouth 静默启动 | `true` |
| `kernel_repo` | 内核仓库 | `map220v/sm8850-mainline` |
| `kernel_branch` | 内核分支 | `iceland-7.3` |
| `kernel_config` | 内核配置（仓库内路径） | `arch/arm64/configs/sm8850.config` |
| `kernel_fragment` | 追加的配置片段 | `config/debian-iceland.fragment` |
| `dtb` | 打包的 DTB | `kaanapali-oneplus-iceland.dtb` |
| `firmware_repo` | 固件仓库（构建时自动下载） | `map220v/iceland-firmware` |
| `firmware_branch` | 固件分支 | `master` |
| `image_size` | rootfs 镜像大小 | `12G` |

密码通过仓库 secret `ROOTFS_PASSWORD` 设置；不设置时回退为 `password`（构建日志里会警告）。

## 内核

- 源码：[map220v/sm8850-mainline](https://github.com/map220v/sm8850-mainline)，分支 `iceland-7.3`（Linux 7.3）。
- 仓库里的 `sm8850.config` 偏向 Android：**没有** ext4、initramfs、cgroups、tmpfs，直接拿来引导 Debian 是起不来的。
- 因此构建时会先合并 [config/debian-iceland.fragment](config/debian-iceland.fragment)，打开 Debian/systemd 需要的选项（ext4、initramfs + zstd、devtmpfs、cgroups、namespaces、VFAT 等），再 `olddefconfig`。
- 设备树为 `arch/arm64/boot/dts/qcom/kaanapali-oneplus-iceland.dts`（当前 `model = "OnePlus Pad 4"`，Pad 3 Pro 共用同一套 DTS）。

## 固件

构建时 `git clone` [map220v/iceland-firmware](https://github.com/map220v/iceland-firmware)，整个目录打包为 `firmware-iceland.deb` 安装到 `/usr/lib/firmware`。换固件源只改 `firmware_repo` / `firmware_branch` 即可。

## 刷机

> 以下操作会改动分区，可能变砖。先备份，并确认自己了解每一步。EDL 对 SM8850 基本不可用，fastboot 是最后的底线。

### 双系统（保留 Android）

在 TWRP 里用 `parted` 缩小 `userdata`，在末尾新建分区并命名 `linux`（概念与 sheng 的方案一致）。然后：

```bash
fastboot erase dtbo_b
fastboot flash boot_b boot.img
fastboot flash linux rootfs.img
fastboot reboot
```

### 单系统（清空 Android）

`boot_mode` 选 `single (userdata)`，把 rootfs 刷进 `userdata`：

```bash
fastboot erase dtbo_b
fastboot flash boot_b boot.img
fastboot flash userdata rootfs.img
fastboot reboot
```

## 启动方式与双系统切换

| 方式 | 说明 |
| --- | --- |
| `android_boot` | ABL 直接引导 Linux 内核（boot header v2，DTB 放在 header 里）。**没有启动菜单**，切回 Android 要刷回原厂 boot 或换 slot。 |
| `uefi_esp` | 产出 `esp.img`，给 **U-Boot / EDK2** 用。U-Boot 的启动菜单可以在 Android 与 Linux 之间切换。需要自己编译并烧录 U-Boot 到 `boot_b`。 |

关于平板自带的 Qualcomm UEFI（`uefi_a/b` 分区）：它只负责启动 ABL，不会直接引导 Linux。想在开机时选择 Android / Linux，现实做法是 U-Boot（参考 [mzakamin-hub/iceland-mainline](https://github.com/mzakamin-hub/iceland-mainline) 的安装方式）。`esp.img` 里已经放好 systemd-boot、`loader/entries` 和 `extlinux/extlinux.conf`，U-Boot 起来后可以直接读。

## 已知限制

- 尚未在 OPD2513 / OPD2514 真机验证。
- 内核 fragment 只覆盖 Debian 启动必需项，音频、传感器、摄像头等设备包还没有做。
- `uefi_esp` 依赖你自己准备 U-Boot；本仓库不构建 U-Boot。
- 单系统模式会清空 Android 数据。

## 致谢

- [map220v](https://github.com/map220v) — sm8850 主线内核、iceland 固件、sheng 的一系列工作
- [debian-sheng-kxt](https://github.com/rrrRex1336/debian-sheng-kxt) — 本仓库的构建流程参考
- [iceland-mainline](https://github.com/mzakamin-hub/iceland-mainline) — iceland 的 U-Boot / 启动链资料
