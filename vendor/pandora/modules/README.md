# pandora (SM8850) 内核模块（.ko）分析

以下内容由原厂 OTA `OS4.0.5.0.XBLCNXM` 的多个分区提取：

- `vendor_boot` 的 **vendor_ramdisk**（早期启动模块，461 个，LZ4+cpio）
- `vendor_dlkm_a`（厂商驱动模块，402 个，EROFS）
- `system_dlkm_a`（GKI 通用模块，206 个，EROFS）

用 `unpackbootimg` 解 vendor_boot → `lz4 -d` + `cpio` 解 ramdisk；用 `payload-dumper-go`
提取 dlkm 分区 → `fsck.erofs --extract` 解 `.ko`。

## 文件清单

| 文件 | 内容 |
|------|------|
| `vendor_boot_modules.txt` | **vendor_boot ramdisk 的 461 个早期启动模块**（含 arm_smmu/clk-qcom/ufs-qcom 等基础模块） |
| `vendor_modules.txt` | vendor_dlkm 的 402 个厂商驱动模块名（高通 + 小米） |
| `system_modules.txt` | system_dlkm 的 GKI 通用模块名（103 个，另有 103 个 flatten 副本） |
| `modules.load` | vendor_dlkm 开机加载顺序（665 个模块，vendor + system 合并） |
| `vendor_boot_modules.load` | vendor_boot ramdisk 的加载顺序 |
| `modules.load.recovery` | recovery 模式的模块加载顺序 |
| `modules.dep` | 模块间依赖关系 |
| `modules.softdep` | 软依赖（softdep）声明 |
| `fstab.qcom` | 第一阶段分区挂载表（first_stage_ramdisk/fstab.qcom） |
| `kmi-analysis.md` | KMI/vermagic 与关键模块依赖链分析 |

## 模块分类统计

内核模块分**三套**（来源不同，缺一不可）：

| 套 | 数量 | 位置 | 说明 |
|----|------|------|------|
| **vendor_boot ramdisk** | **461** | vendor_boot 分区 ramdisk | 早期启动基础模块（时钟/MMU/存储/SoC） |
| vendor_dlkm | 402 | vendor_dlkm 分区 | 厂商驱动（GPU/显示/相机/充电/音频） |
| system_dlkm | 206 | system_dlkm 分区 | GKI 通用（网络/USB/虚拟化） |
| **三套合集去重** | **671** | — | 唯一模块总数 |

> ⚠️ **关键**：vendor_boot ramdisk 的 168 个模块是 vendor_dlkm **没有的**基础模块，
> 包括 `arm_smmu`、`ufs-qcom`、`clk-qcom`、`gcc-canoe`、`qcom-scm`、`socinfo`、
> `llcc-qcom`、`pinctrl-canoe`、`qcom-rpmh` 等 —— 这些是内核早期初始化（存储、
> 时钟、内存映射）的根基，缺失则无法启动。

## 关键大模块

| 模块 | 大小 | 用途 |
|------|------|------|
| qca_cld3_wcn7750.ko | 20 MB | Wi-Fi 固件（CNSS） |
| ufs_ffu.ko | 12 MB | UFS 存储固件升级 |
| camera.ko | 7.4 MB | 相机 |
| msm_drm.ko | 6.8 MB | 显示 DRM |
| ipam.ko | 6.4 MB | 内存管理 |
| msm_kgsl.ko | 3.3 MB | GPU（Adreno 840） |

详见 `kmi-analysis.md`。
