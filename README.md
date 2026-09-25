# TL-WDR5600 v2 OpenWrt 云编译工程

针对 **TP-Link TL-WDR5600 v2**（QCA9561 + QCA9887，8MB Flash / 64MB RAM）的 GitHub Actions 自动编译。
补丁来自 Gitee `afeng11/openwrt-tl-wdr5600-v2`（OpenWrt 19.07.4 / ath79 目标）。

---

## 一、怎么用（只需 4 步）

1. 注册/登录 GitHub，新建一个**公开（Public）仓库**（公开仓库云编译免费、不限时长）。
2. 把本文件夹里的全部文件按原目录结构上传到仓库根目录：
   ```
   .github/workflows/build.yml
   patches/0001-add-tl-wdr5600-v2-support.patch
   patches/0002-ath79-drop-redundant-chosen-bootargs-for-tl-wdr5600-.patch
   README.md
   ```
   （网页上传时直接把 patches 文件夹和 .github 文件夹拖进去即可；注意 `.github` 文件夹要在根目录。）
3. 仓库页面点 **Actions** 标签 → 左边选 **Build OpenWrt for TL-WDR5600 v2** → 点 **Run workflow**。
4. 等编译跑完（约 1.5~2.5 小时），在该次运行页面底部 **Artifacts** 里下载 `openwrt-wdr5600-v2-sysupgrade`，解压后得到：
   ```
   openwrt-19.07.4-ath79-generic-tplink_tl-wdr5600-v2-squashfs-sysupgrade.bin
   ```

> 首次若提示 Actions 未启用，到仓库 Settings → Actions → General 里允许即可。

---

## 二、刷写方法（必须用编程器）

这台机器原厂有两个 U-Boot：`factory_boot`（0x0 起）会校验签名、串口阶段无法输入；进 Linux 又要 root 密码（未知），所以**只能拆机用编程器刷**。

准备：编程器读出的**原厂整片备份**（务必先留一份）。

1. 打开原厂备份镜像；把 `0x00000–0x1D800`、`0x40000–0x800000` 区段填 `FF`（不填也行，作者强迫症）。
2. 把原厂镜像里 `0x30000–0x40000`（即 `normal_boot`，第二个 U-Boot，不校验签名）**复制到 `0x0–0x10000`**。
3. 把下载到的 **sysupgrade.bin** 的全部内容**写到 `0x40000` 偏移处**。
4. 校验无误后用编程器整写回 Flash。

完成后开机：
- 网段 `192.168.1.x`，OpenWrt 后台 `http://192.168.1.1`，首次用 root 登录、**无密码**，进去后自己设密码。
- 无线默认开启，2.4G/5G 都能用；MAC 由固件从 `factory_info`/`art` 自动读取。

---

## 三、关键硬件/分区信息（已在补丁中固化）

| 项目 | 值 |
|---|---|
| SoC / 5G | QCA9561 / QCA9887（ath10k-ct） |
| Flash / RAM | 8MB（25Q64） / 64MB DDR2 |
| Reset 键 | GPIO 1（低电平有效） |
| 状态灯 | GPIO 21 |
| 串口 | TTL，TX→RX/RX→TX/GND，VCC 不接；115200 |

原厂分区：
```
0x00000–0x01D800  factory_boot
0x01D800–0x01E000  factory_info
0x01E000–0x020000  art        （2G 校准在开头 1088B；5G 校准在 +0x1000，2116B）
0x020000–0x030000  config
0x030000–0x040000  normal_boot （第二个 U-Boot）
0x040000–0x800000  firmware    （kernel + rootfs）
```

---

## 四、可选：想加软件

编辑 `.github/workflows/build.yml` 里 `cat > .config` 那段，按下面格式加一行再提交即可：
```
CONFIG_PACKAGE_luci-app-xxx=y
```
例如加上网攻略常用的：
```
CONFIG_PACKAGE_luci-app-ttyd=y
CONFIG_PACKAGE_luci-app-opkg=y
```
注意只有 8MB Flash，别塞太多。
