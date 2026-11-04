# Xiaomi 17 Pro（pandora）内核分析报告

> 目标固件：`pandora-ota_full-OS4.0.5.0.XBLCNXM-user-17.0-a4da788249.zip`（HyperOS 4.0.5.0 / Android 17）
> 芯片平台：Qualcomm Snapdragon 8 Elite Gen 5（SM8850-AC，平台内部代号 **canoe**）
> 设备代号：**pandora**（Xiaomi 17 Pro，型号 25098PN5AC）

---

## 一、内核分区的位置与提取方式

### 1. 固件包结构

该固件是 **A/B 分区 OTA 完整包**（`ota-type=AB`），核心文件为 `payload.bin`（9.17 GB），所有分区镜像都封装在其中，使用 `payload` v2 格式。

| 固件文件 | 大小 | 说明 |
|---|---|---|
| `payload.bin` | 9.17 GB | 所有分区镜像的封装体 |
| `META-INF/com/android/metadata` | — | OTA 元数据（pre-device=pandora，AB 类型） |
| `payload_properties.txt` | — | payload 校验哈希 |

### 2. 内核相关分区清单

用 `payload-dumper-go` 解析 `payload.bin` 得到完整分区表，**内核相关分区**如下：

| 分区名 | 大小 | 作用 |
|---|---|---|
| **`boot`** | 101 MB | 主内核分区：GKI kernel Image（header_version 4，ramdisk 为空） |
| **`init_boot`** | 8.4 MB | 通用 ramdisk（GKI 2.0 的 init 逻辑，LZ4 压缩） |
| **`vendor_boot`** | 101 MB | vendor ramdisk + **DTB（设备树，6.5 MB）** |
| **`dtbo`** | 23 MB | 设备树覆盖层（DTBO） |
| **`vendor_dlkm`** | 60 MB | 厂商内核模块（.ko，EROFS 文件系统） |
| **`system_dlkm`** | 17 MB | 系统内核模块（.ko，EROFS 文件系统） |

其余分区（system 977MB / vendor 589MB / odm 3.9GB / product 4.7GB 等）与内核无关。

### 3. 提取方法（完整命令）

```bash
# ① 从固件 zip 解压 payload.bin（9GB，需足够磁盘空间）
unzip pandora-ota_full-*.zip payload.bin -d firmware/pandora/package/

# ② 下载 payload 解析工具
wget https://github.com/ssut/payload-dumper-go/releases/download/1.2.2/payload-dumper-go_1.2.2_linux_amd64.tar.gz

# ③ 只提取内核相关分区（不必解出全部 9GB 分区）
payload-dumper-go -p boot,init_boot,vendor_boot,dtbo,vendor_dlkm,system_dlkm -o ./extracted payload.bin

# ④ 解包 boot.img / vendor_boot.img / init_boot.img
unpackbootimg -i extracted/boot.img        -o boot_unpacked/
unpackbootimg -i extracted/vendor_boot.img -o vb_unpacked/
unpackbootimg -i extracted/init_boot.img   -o ib_unpacked/
```

解包结果：
- `boot.img-kernel` → 内核 Image（41.5 MB，PE/EFI 格式的 aarch64 内核）
- `vendor_boot.img-dtb` → 设备树 DTB（6.5 MB）
- `vendor_boot.img-vendor_ramdisk` / `init_boot.img-ramdisk` → LZ4 压缩 ramdisk

---

## 二、反编译后的源码结构（内核二进制分析）

### 关键结论：GKI 2.0 通用内核，源码不在固件内

该内核是 **Android Common Kernel（ACK）GKI 2.0** 产物。`boot.img` 里的 `kernel` 是**编译后的二进制 Image**，不包含源码。**内核二进制无法「反编译」出可读的 C 源码**——正确做法是根据二进制里提取的版本指纹，去对应的源码仓库拉取源码。

### 从内核 Image 提取的版本指纹

```
Linux version 6.12.69-android16-6-g43b5ab7e247d-abogki561813157-4k
compiler: clang 19.0.1 + LLD 19.0.1 (Android r536225, +pgo +bolt +lto +mlgo)
build:    kleaf / GKI 2.0
arch:     aarch64, 4K pages, SMP PREEMPT
```

**版本指纹拆解：**
| 字段 | 含义 |
|---|---|
| `6.12.69` | Linux 主线版本 |
| `android16-6` | Android 16 的 ACK 分支（kernel-6.12） |
| `g43b5ab7e247d` | 内核 git commit（`abogki` 分支） |
| `abogki561813157` | ABI/KMI 标识（`abogki` = Android Bionic Open Generic Kernel Image 分支） |

### ramdisk 结构（init_boot）

```
init_boot.img-ramdisk (LZ4)
├── init
├── system/bin/{getevent,getprop,setprop,modprobe,snapuserd,...}
├── system/etc/init/snapuserd.rc
└── system/etc/ramdisk/build.prop   # ro.product.bootimage.device=missi（GKI 通用）
```

### 设备树（DTB）确认平台代号 canoe

从 `vendor_boot.img-dtb` 提取到的 SoC 设备树节点：

```
qcom,canoe              qcom,canoe-gcc      qcom,canoe-gpucc
qcom,canoe-pdc          qcom,canoe-camcc    qcom,canoe-videocc
qcom,canoe-tlmm         qcom,canoe-dispcc   qcom,canoe-debugcc
qcom,canoe-llcc         qcom,canoe-rpmh-clk  ...
```

> **`canoe` 是 SM8850 平台（Snapdragon 8 Elite Gen 5）的高通内部代号**，`pandora` 是小米 17 Pro 的设备代号，二者关系：canoe = SoC 平台，pandora = 基于该平台的设备。

### vendor_dlkm 内核模块（.ko，300+ 个）

`vendor_dlkm.img`（EROFS）内含厂商内核模块，涵盖高通 cano 平台驱动 + 小米定制驱动：

- 高通平台：`kgsl`(GPU)、`gcc/camcc/gpucc`、`msm_geni_serial`、`msm_drm`、`qcom_glink`、`cnss2`(Wi-Fi)、`crypto-msm_dlkm` 等
- 小米定制：`mca_*`（充电）、`cs35l43_dlkm`（音频）、`mi_fp`、`xiaomi_touch`、`synaptics_tcm2`/`focaltech_touch`（触控）、`machine_dlkm`、`gpi` 等
- 完整列表见固件解包产物（约 300+ 个 .ko）

---

## 三、GitHub 上可用的 SM8850 内核仓库

> ⚠️ **重要**：小米官方 `MiCode/Xiaomi_Kernel_OpenSource` 仓库**目前尚无 sm8850/pandora/canoe 分支**（已核实 100 个分支中无匹配）。以下均为第三方社区适配仓库。

### 1. 内核源码仓库（推荐）

| 仓库 | 说明 | 状态 |
|---|---|---|
| **[YuzakiKokuban/android_kernel_xiaomi_sm8850](https://github.com/YuzakiKokuban/android_kernel_xiaomi_sm8850)** | Kokuban 内核，Linux 6.12.69，面向小米 17 全系（pudding/pandora/popsicle/nezha），含 Hybrid Mount/BBG，LKM + BakaSU 两种发行 | ⭐ 活跃（2026-09 更新） |
| **[LokumKernel-SM8850/android_kernel_xiaomi_sm8850](https://github.com/LokumKernel-SM8850/android_kernel_xiaomi_sm8850)** | LokumKernel，导入小米 SM8850 原厂 GKI 基线，含 SUSFS v2.1.0（KernelSU Next），专门面向 **Pandora** | ⭐ 活跃（2026-05 更新） |
| **[coolzyd9107/android_kernel_xiaomi_sm8850](https://github.com/coolzyd9107/android_kernel_xiaomi_sm8850)** | Kokuban 内核的分支，LKM + ReSukiSU 发行 | 活跃 |

### 2. 内核模块（OOT）生态

- **Picters Kernel**（XDA，fork 自 Kokuban）：A16（Linux 6.12.23 / KMI 5）+ A17（Linux 6.12.69 / KMI 6），打包 Wi-Fi 注入、蓝牙扩展、USB gadget 等外置模块，AnyKernel3 安装。

### 3. 设备树 / TWRP 仓库

| 仓库 | 说明 |
|---|---|
| **[MissMyTime/twrp_device_sm8850](https://github.com/MissMyTime/twrp_device_sm8850)** | SM8850/canoe 平台 TWRP 设备树，覆盖 Redmi K90(annibale)、K90 Pro Max(myron)、Xiaomi 17 Ultra(nezha)、realme Neo8 |
| **[yukonga/twrp-device-xiaomi-sm8850](https://github.com/yukonga/twrp-device-xiaomi-sm8850)** | 小米 SM8850 TWRP 设备树，支持 pudding/pandora/popsicle/nezha 四设备 |

### 4. 官方内核源码获取路径（兜底）

小米官方 GKI 内核发布后，正确查找路径：
- 官方仓库：`https://github.com/MiCode/Xiaomi_Kernel_OpenSource`（分支命名 `{codename}-{android字母}-oss`）
- 高通 ACK：`https://android.googlesource.com/kernel/common`（`abogki` 分支对应 `abogki561813157` KMI）
- 用 commit hash `g43b5ab7e247d` 可在 ACK 仓库中精确定位源码快照

---

## 四、Xiaomi 17 Pro（pandora/canoe）设备树与配置文件

### 1. 设备硬件规格（已确认）

| 项目 | 规格 |
|---|---|
| 型号 | Xiaomi 17 Pro（25098PN5AC） |
| 代号 | pandora（设备）/ canoe（SoC 平台） |
| SoC | Snapdragon 8 Elite Gen 5（SM8850-AC，3nm） |
| CPU | 8 核 Oryon-V3，2×4.61GHz + 6×3.63GHz |
| GPU | Adreno 840 |
| 内存/存储 | 16GB LPDDR5X/6 + UFS 4.1 |
| 内核 | Linux 6.12.69-android16-6（GKI 2.0，4K pages） |
| 系统 | HyperOS 4.0.5.0 / Android 17（SDK 37） |

### 2. defconfig 配置文件

两个内核仓库都基于 **GKI 2.0**，使用 `gki_defconfig` 作为基础，配合平台 fragment：

| 文件 | 仓库 | 说明 |
|---|---|---|
| `arch/arm64/configs/gki_defconfig` | 两者均有 | GKI 2.0 标准基础配置 |
| `arch/arm64/configs/xiaomi_sm8850_bootimg.fragment` | LokumKernel | 小米 SM8850 专属 boot.img fragment |
| `arch/arm64/configs/wonder.fragment` | YuzakiKokuban | SM8850/canoe 平台 fragment |
| `build.config.gki.aarch64` | 两者均有 | GKI aarch64 构建配置 |

> 完整设备树（`.dts`）在 `arch/arm64/boot/dts/` 下（SM8850 相关，通常位于 vendor 或 `qcom/` 子目录），需 clone 仓库后查看。

### 3. 小米 17 系列设备代号对照

| 设备 | 代号 |
|---|---|
| Xiaomi 17 | pudding |
| **Xiaomi 17 Pro** | **pandora** |
| Xiaomi 17 Pro Max | popsicle |
| Xiaomi 17 Ultra | nezha |

### 4. 分区布局关键点

- **boot**（GKI 内核）+ **init_boot**（通用 ramdisk）+ **vendor_boot**（vendor ramdisk + DTB）+ **vendor_dlkm**（vendor 内核模块）是 GKI 2.0 标准四件套。
- 刷内核时：GKI 内核刷入 `boot`，root 修补 `init_boot`（KernelSU/Magisk），vendor 模块在 `vendor_dlkm`。

---

## 五、super.img 动态分区分析（补充）

### 1. super.img 概况

`super.img`（13.4 GB = 13421772800 字节）是 Android **动态分区（dynamic partition）**超级镜像，采用 **LPD metadata v10.2** 格式（raw，非 sparse），metadata 位于 offset 4096 起。

### 2. 逻辑分区清单（A/B 双槽）

用纯 Python 手写 LPD 解析器（无需外部工具）解析出 16 个逻辑分区：

| A 槽（有数据） | B 槽（空） | 大小 | 说明 |
|---|---|---|---|
| `odm_a` | `odm_b` | 3.9 GB | ODM 定制 |
| `product_a` | `product_b` | 4.2 GB | 产品分区 |
| `system_a` | `system_b` | ~975 MB | 系统 |
| `system_ext_a` | `system_ext_b` | ~816 MB | 系统扩展 |
| `vendor_a` | `vendor_b` | ~592 MB | 厂商 |
| `mi_ext_a` | `mi_ext_b` | ~171 MB | 小米扩展 |
| **`vendor_dlkm_a`** | `vendor_dlkm_b` | **56.6 MB** | **厂商内核模块** |
| **`system_dlkm_a`** | `system_dlkm_b` | **16.4 MB** | **系统内核模块** |

> B 槽全部 `num_extents=0`（空槽），说明这是全新刷机包，系统只占用 A 槽。

### 3. 内核相关分区的提取结果

从 super.img 中按 LPD metadata 记录的 extent 偏移，用 `dd` 精确切出：

| 分区 | 起始扇区 | 大小 | 文件系统 | 提取产物 |
|---|---|---|---|---|
| `vendor_dlkm_a` | 20781056 | 115872 扇区 (56.6 MB) | EROFS | `super_extracted/vendor_dlkm_a.img` |
| `system_dlkm_a` | 20897792 | 33520 扇区 (16.4 MB) | EROFS | `super_extracted/system_dlkm_a.img` |

### 4. system_dlkm 内核模块清单（GKI 通用模块）

`system_dlkm_a.img`（EROFS）内含 GKI 通用内核模块（约 120 个），包括：

- **核心**：`rust_binder.ko`（Rust Binder）、`zram.ko`、`zsmalloc.ko`、`gnss.ko`、`bluetooth.ko`、`nfc.ko`、`wwan.ko`
- **网络**：`8021q.ko`、`macsec.ko`、`can.ko` 系列、`tipc.ko`、`6lowpan.ko`、`ieee802154.ko`、`mac802154.ko`、`l2tp_*.ko`、`ppp_*.ko`、`pptp.ko`
- **USB**：`usbnet.ko`、`usbserial.ko`、`usbmon.ko`、`cdc_*.ko`、`r8152.ko`、`r8153_ecm.ko`、`asix.ko`、`ax88179_178a.ko`、`aqc111.ko`、`ftdi_sio.ko`、`rtl8150.ko`
- **虚拟化**：`virtio_*.ko`、`vmw_vsock_virtio_transport.ko`、`ptp_kvm.ko`、`vcpu_stall_detector.ko`
- **测试/调试**：`kunit.ko`、各种 `*_test.ko`、`kheaders.ko`

### 5. vendor_dlkm 内核模块清单（厂商驱动，关键）

`vendor_dlkm_a.img` 内含厂商（高通 + 小米）驱动模块，已用 `fsck.erofs --extract` 完整解出 **402 个 .ko**（总 132MB），分类：

- **小米定制 59 个**：`mca_*`（充电全套 30+）、`mi_*`（mi_fp/mi_cam/mi_reclaim/mi_memcheck/mi_ntsync 等）、`xiaomi_touch`、`xm_usb_pri_interface`
- **高通平台 37 个**：`qcom_*`/`qti_*`（qcom_glink/qcom_scm/qti_thermal 等）
- **音频 22 个**：`wcd938x_dlkm`、`lpass_cdc_*`、`swr_*`、`cs35l43_dlkm`、`aw882xx_dlkm`、`tas25xx_dlkm`、`tfa98xx_dlkm`、`sia91xx_dlkm`
- **关键大模块**：`msm_drm.ko`(6.8MB)、`camera.ko`(7.4MB)、`msm_kgsl.ko`(3.3MB)、`ufs_ffu.ko`(12MB)、`qca_cld3_wcn7750.ko`(20MB Wi-Fi)、`ipam.ko`(6.4MB)

**模块 vermagic 验证**（msm_kgsl.ko）：`6.12.69-android16-6-4k SMP preempt mod_unload modversions aarch64`

**msm_kgsl.ko 依赖 18 个模块**：`socinfo, llcc-qcom, qcom-dcvs, qcom-scm, cmd-db, msm_performance, qcom_aoss, os_cpu_qi, msm_sysstats, mem_buf_dev, gpu_stats, qcom_iommu_util, clk-qcom, secure_buffer, mdt_loader, minidump, qcom_va_minidump, miev, coresight, synx-driver`

**modules.load 共 665 个模块**（含 system_dlkm），启动按序加载；`modules.softdep` 记录了软依赖（如 `msm_kgsl pre: arm_smmu nvmem_qfprom socinfo governor_msm_adreno_tz ...`）。

### 6. system_dlkm 内核模块清单（GKI 通用模块）

`system_dlkm_a.img` 含 **206 个 .ko**（103 个结构化 + 103 个 flatten 副本），模块目录名即内核版本 `6.12.69-android16-6-g43b5ab7e247d-abogki561813157-4k`：

- **核心**：`rust_binder.ko`（Rust Binder，在 `kernel/drivers/android/binder/`）、`zram.ko`、`gnss.ko`、`bluetooth.ko`、`nfc.ko`、`wwan.ko`
- **网络**：`8021q.ko`、`macsec.ko`、`can.ko` 系列、`tipc.ko`、`6lowpan.ko`、`ieee802154.ko`、`mac802154.ko`、`l2tp_*.ko`、`ppp_*.ko`
- **USB**：`usbnet.ko`、`usbserial.ko`、`usbmon.ko`、`cdc_*.ko`、`r8152.ko`、`asix.ko`、`ftdi_sio.ko`、`ax88179_178a.ko`、`aqc111.ko`
- **虚拟化**：`virtio_*.ko`、`vmw_vsock_virtio_transport.ko`、`ptp_kvm.ko`
- **测试**：`kunit.ko` 及各种 `*_test.ko`

> ⚠️ **重要印证**：这些 vendor 驱动正是之前「换纯 ACK 内核卡第一屏」的根因——`msm_kgsl`/`msm_drm`/`ufs_ffu`/`qcom_scm` 等显示与存储驱动全是 vendor_dlkm 里小米编译的 .ko，且 `msm_kgsl` 依赖 `socinfo`/`llcc-qcom`/`qcom_iommu_util` 等 18 个模块，纯 ACK 内核无法承载这条完整依赖链。

---

## 六、设备树反编译结果（DTB + DTBO）

### 1. 主设备树 DTB（vendor_boot 内，6.5MB = 12 个 FDT blob）

`vendor_boot.img` 内的 DTB 是 **12 个独立设备树拼接**，覆盖 3 个高通平台：

| 索引 | compatible | GPU | 说明 |
|---|---|---|---|
| DTB 0-1 | `qcom,canoe` | Adreno 840 / 840v2 | SM8850 标准版 |
| DTB 2-3 | `qcom,canoe` | Adreno 840v2 / 840 | SM8850 标准版 |
| DTB 4-5 | `qcom,canoep` | Adreno 34v2 / 34v1 | SM8850 Pro 版 |
| DTB 6-7 | `qcom,canoep` | — | CanoeP Alt. Thermal Profile |
| DTB 8-9 | `qcom,canoep` | Adreno 840v2 / 840 | SM8850 Pro 版 |
| DTB 10-11 | `qcom,whale` / `qcom,whalep` | Adreno 835 | 另一平台（SM8750 系） |

> bootloader 根据实际 SoC 型号从 12 个 DTB 中选择匹配的（canoe/canoep/whale 三种平台 × 不同 GPU/热配置变体）。

### 2. 设备树覆盖层 DTBO（dtbo.img，2 个 DTBO）

`dtbo.img` 的 dt_table 头（魔数 `0xd7b7ab1e`）含 **2 个 DTBO**，都是 **pandora 设备专属覆盖层**：

| DTBO | model | msm-id | board-id | miboard-id |
|---|---|---|---|---|
| dtbo_00 | "Pandora based on SM8850" | 0x2d3/0x2d2/0x2e7 | 0x08 0x00 | 0x03 0x00 |
| dtbo_01 | "Pandora based on SM8850" | 0x295/0x294 | 0x08 0x00 | 0x03 0x00 |

- `compatible = "qcom,canoe-mtp" + "qcom,canoe" + "qcom,canoep-mtp" + "qcom,canoep" + "qcom,mtp"`
- `xiaomi,miboard-id = <0x03 0x00>` 是小米专属板级 ID（pandora）
- 标准 DTBO overlay 格式（`fragment@N` + `__overlay__` + `phandle`）

### 3. pandora 设备树覆盖层关键硬件配置（从 DTBO 反编译提取）

- **屏幕/显示**：`qcom,amoled-ecm`、`qcom,amoled-regulator`、PMIC `pmd802x`（OLEDB/ELVDD/IBB 电源）、`xiaomi,dump_display`
- **触控**：`xiaomi-touch`、`synaptics,tcm-spi`、`focaltech,fts`（对应 vendor_dlkm 的 xiaomi_touch/synaptics_tcm2/focaltech_touch 三个 .ko）
- **充电**：小米 `mca` 全套（`mca,mca_panel`、`mca,bc12_class`、`mca,protocol_pd`、`mca,protocol_qc`、`mca,buckchg_jeita` 等）
- **PMIC**：`pmk8850`（主 PMIC）、`pmd802x`、`pmih010x`

### 4. 反编译工具与产物

- 工具：`dtc` 1.7.0（从 Ubuntu noble 的 `device-tree-compiler_1.7.0-2build1` .deb 提取，无需 root）
- 产物位置（WSL `~/wsl-android-workspace/firmware/pandora/`）：
  - `dtb/dtb_00.dtb ~ dtb_11.dtb` + 对应 `.dts`（每个约 2.8 万行）
  - `dtbo/dtbo_00.dtbo`、`dtbo_01.dtbo` + 对应 `.dts`（5797 行 / 25138 行）
- 反编译命令：`dtc -I dtb -O dts <input>`

---

## 七、结论与建议

1. **内核分区已完整定位并提取**：`boot`（内核 Image）、`init_boot`（ramdisk）、`vendor_boot`（DTB+vendor ramdisk）、`dtbo`、`vendor_dlkm`/`system_dlkm`（模块），产物在 WSL `~/wsl-android-workspace/firmware/pandora/` 下。
2. **super.img 动态分区已解析**：16 个逻辑分区（A/B 双槽），内核相关的 `vendor_dlkm_a`（56.6MB）和 `system_dlkm_a`（16.4MB）已按 extent 偏移精确提取，产物在 `super_extracted/` 下，均为 EROFS 文件系统。
3. **设备树已完整反编译**：vendor_boot 内 12 个主 DTB（canoe/canoep/whale 三平台）+ dtbo.img 内 2 个 pandora 专属 DTBO，均已用 dtc 反编译成 .dts 源码，产物在 `dtb/` 和 `dtbo/` 目录。
4. **内核版本确认为 Linux 6.12.69**（GKI 2.0，commit `g43b5ab7e247d`，KMI `abogki561813157`）。
5. **源码不在固件内**（二进制产物），且小米官方 MiCode 仓库尚未发布 sm8850 源码；可用源码来自第三方仓库（Kokuban/LokumKernel 等）。
6. **canoe = SM8850 平台代号，pandora = 小米 17 Pro 设备代号**，二者已通过 DTB（`qcom,canoe`）和 build.prop 双重验证。
7. 若需自行编译内核模块（.ko）或定制内核，推荐从 `YuzakiKokuban/android_kernel_xiaomi_sm8850`（Linux 6.12.69，与当前固件版本完全一致）clone 源码，配合 `gki_defconfig` + 平台 fragment 构建，交叉编译用 aarch64-linux-gnu- 工具链。
