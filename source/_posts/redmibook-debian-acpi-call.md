---
title: RedmiBook Pro 14 2025 在 Debian Testing 下配置电池充电限制
date: 2026-09-22 17:32:54
tags:
---

本文的内容是关于如何在 Redmibook Pro 14 2025 上的 Debian Testing 系统内配置 acpi-call 包，进行电池充电限制并且不关闭 SecureBoot。

涉及几个重要的参考链接。以及非常感谢 Deepseek v4 Flash 通过 Codex 帮我排查问题。

[Debian 软件源上 acpi-call 包的 README](https://sources.debian.org/src/acpi-call/1.2.2-2.1/README.md/)

[Arch Wiki 上提到的类似机型的配置方法](https://wiki.archlinux.org.cn/title/Xiaomi_RedmiBook_Pro_16_2025)

[配置DKMS签名的 Github 文档，来自 Debian 的 README](https://gist.github.com/s-h-a-d-o-w/53c2215e955c3326c6ec8f812a0d2f27)

###  前置准备

需要安装`dkms`包。使用`apt`安装即可，这个命令疑似只会出现在 /usr/sbin 下，所以必须`sudo`使用，即使是root也是。

需要安装内核版本对应的头文件。

```bash
sudo apt update
sudo apt install dkms linux-headers-$(uname -r)
```

可能需要安装`acpi-call-dkms`包。不清楚是不是需要先装上。应该不装也可以。

如果想在这一步安装`acpi-call-dkms`，可使用下述命令检查。

```bash
sudo dkms status
# 如果不是 installed，用这个命令安装
sudo dkms autoinstall
```

这一步可以尝试一下

```bash
sudo depmod
sudo modprobe acpi_call
```

如果没报错或许可以直接用。报错了可移步下文继续。

### 配置签名

如果开启 Secure Boot，就没法加载没签名的内核模块，这就是上文无法加载模块的原因。大概可以直接关闭 Secure Boot 解决，但是我懒得，怕我另一边的 Windows 出什么奇怪的问题，因此我选择配置签名。

#### 导入 Key 和修改脚本

接下来，需要配置`dkms`签名，[参考这个文档](https://gist.github.com/s-h-a-d-o-w/53c2215e955c3326c6ec8f812a0d2f27)。

这个脚本不能直接用，需要修改一下，参见下文。不过用于导入 Key 是没问题的。

下文是安装了`acpi-call-dkms`之后的解决方法，如果没有安装，只需要按照修复办法里第一条修改一下脚本。

---

## 二、acpi_call 为什么加载不了（`bootp.log`）

**根因：模块被重新压缩成了内核不支持的 xz 格式（CRC64 校验），内核解压阶段就失败了。**

- `bootp.log:948`、`1031`（开机时 systemd-modules-load 触发）、`1138`、`1139`（你手动 `sudo modprobe acpi_call`）都是同一条内核消息：`decompression failed with status 6`

  **（注：这里特别坑人，因为这个时候`modprobe`给我报的是`module not found`之类的东西，没想到是解压的问题）**

- `bootp.log:967-969` 因此 `systemd-modules-load.service` 启动失败

- 该消息来自 `kernel/module/decompress.c`（我在本机 vmlinux 里核对了字符串位置，紧邻 `not an an xz compressed module`），status 6 对应内核 `include/linux/xz.h` 的 `XZ_OPTIONS_ERROR`，即「压缩流头部用了内核不支持的特性」

- 实测格式差异：
  - 你现在的模块 `/lib/modules/7.1.13+deb14-amd64/updates/dkms/acpi_call.ko.xz` → `Check: CRC64`、`--lzma2=dict=8MiB`
  - Debian 自带模块（如 `snd-seq-device.ko.xz`）→ `Check: CRC32`
  - `/usr/sbin/dkms:3158` 的默认值就是 `compress_xz_opts="--check=crc32 --lzma2=dict=1MiB"`
  
- 出处就在你下载的那个 zip 里：`sign-modules` 脚本把模块 `unxz` → 签名 → 用**裸 `xz -f`** 重新压缩，而裸 `xz` 默认用 `--check=crc64`。DKMS 自己压缩时用的是 `--check=crc32`，被这个脚本覆盖掉了，所以 `/var/lib/dkms/.../module/acpi_call.ko.xz` 也是 CRC64。

签名本身现在是对的，我核对过：模块 PKCS#7 里的签发者 CN=`NeptaLiy module signing key`、serial `1d:8c:de:f0:…38:af`，与已注册 MOK（key 2）完全一致，`bootp.log:703` 也显示开机把该证书装进了 keyring。

另外，`boot.log:951/1005/1026`、`boot_last.log:950/1018/1024/1129` 里的 `Loading of module with unavailable key is rejected` 是**另一个、已经修好的问题**：那次启动时模块是用 DKMS 自己的密钥（`/var/lib/dkms/mok.pub`，CN=`DKMS module signing key`，**没有注册进 MOK**）签的，所以被 `kernel/module/signing.c` 拒收。换成 gist 的 MOK 签名后这条就消失了，之后才暴露出压缩格式的问题。

## 修复办法

1. 改 `/root/module-signing/sign-modules`，把重新压缩那行换成和 DKMS/内核兼容的参数：
   - `xz -f $module_basename` → `xz --check=crc32 --lzma2=dict=1MiB -f $module_basename`
2. 重新构建签名：`sudo dkms remove acpi-call/1.2.2 --all && sudo dkms install acpi-call/1.2.2`
3. 校验：`xz -lvv /lib/modules/$(uname -r)/updates/dkms/acpi_call.ko.xz` 应显示 `Check: CRC32` / `Memory needed: 2 MiB`，然后 `sudo modprobe acpi_call && lsmod | grep acpi_call`。

---

总之就是要改那一行。到时候进行签名可以直接用`apt`重装`acpi-call-dkms`包，会自动触发签名脚本，也可以用`dkms`。

这里需要特别注意名字，`acpi-call`和`acpi_call`不同。似乎是要用`acpi-call`的。

执行`one-time-setup`，输入密码即可生成 MOK Key（这一步是用`openssl`），脚本会把它加入待注册的 Key 队列，之后还有一个 MOK 密码，需要在重启导入 Key 的时候用。反正是自己机器，建议直接都设置成同一个。

重启后，会出现一个蓝色的MOK Manager界面，依次选择`Enroll MOK`，`Continue`，`Yes`，输入密码后选择`Reboot`即可。

重启正常进入系统后，使用

```bash
mokutil --list-enrolled | grep -i "Issuer"
```

就可以看到（一般而言）两个Issuer，我这一个是`Debian Secure Boot CA`，一个是`<机器名> module signing key`。

#### 配置签名

此时，安装或重装`acpi-call-dkms`，或者是用上述`sudo dkms remove acpi-call/1.2.2 --all && sudo dkms install acpi-call/1.2.2`命令重装并构建签名，就可激活脚本对这个包进行签名。这一步也需要使用到之前的密码。

再使用

```bash
sudo modprobe acpi_call
```

理论上就没问题了。此时执行`ls /proc/acpi`，应该已经有一个新文件名为`call`。

### 充电限制脚本

参考[Arch Wiki 上提到的类似机型的配置方法](https://wiki.archlinux.org.cn/title/Xiaomi_RedmiBook_Pro_16_2025)即可。如果疑心效果，可以用如下办法验证。

```bash
# 使用 upower
upower -i /org/freedesktop/UPower/devices/battery_BAT0 # state 会显示为 pending-charge
# 简单查看
cat /sys/class/power_supply/ADP1/online # 显示为 1
cat /sys/class/power_supply/BAT0/status # 显示为 Not charging
cat /sys/class/power_supply/BAT0/power_now # 显示为0
```

再把它用`systemd`自动化就可以了。

