# pandora 内核内容审计报告：固件 vs 仓库

> 审计时间：2026-11-04
> 对比对象：原厂 OTA `OS4.0.5.0.XBLCNXM` 固件提取内容 vs `android_kernel_xiaomi_pandora` 仓库

## 结论先行

仓库的**核心内容（内核源码、defconfig、12 设备树、2 DTBO）完整且与固件一致**，但存在
**一类重大遗漏**：`vendor_boot` 分区 ramdisk 里的 **461 个早期启动内核模块**和 **fstab.qcom**
之前完全没提取，本次已补齐。

---

## 一、逐项对比结果

| 项 | 固件提取 | 仓库现状 | 结论 |
|----|---------|---------|------|
| 内核源码（GKI） | 6.12.69-android16-6-g43b5ab7e247d | AOSP kernel/common @4ad1cb12b | ✅ 一致（基线） |
| pandora_defconfig | 223592 字节（IKCONFIG） | 223592 字节 | ✅ 一致 |
| 12 个设备树 (.dtb) | 12 个，581542~503332B | 12 个，大小逐一相等 | ✅ 完全一致 |
| 12 个设备树 (.dts) | 12 个反编译源码 | 12 个 | ✅ 一致 |
| 2 个 DTBO (.dtbo) | 2 个（141842/1134915B） | 2 个，md5 一致 | ✅ 一致 |
| 2 个 DTBO (.dts) | 2 个反编译源码 | 2 个（本次补充） | ✅ 已补齐 |
| **vendor_boot ramdisk 模块** | **461 个 .ko** | **缺失** | ❌ **本次补齐** |
| **fstab.qcom** | 存在（first_stage） | **缺失** | ❌ **本次补齐** |
| **modules.load.recovery** | 存在 | **缺失** | ❌ **本次补齐** |
| vendor_dlkm 模块 | 402 个 | 402 个清单 | ✅ 一致 |
| system_dlkm 模块 | 206 个 | 103+103 清单 | ✅ 一致 |

---

## 二、重大遗漏详情：vendor_boot ramdisk 的 461 个模块

### 为什么这是重大遗漏

内核模块实际分**三套**，之前只提取了两套（vendor_dlkm + system_dlkm），
**遗漏了 vendor_boot ramdisk 这套早期启动模块**：

| 套 | 数量 | 位置 | 加载时机 |
|----|------|------|---------|
| **vendor_boot ramdisk** | **461** | vendor_boot 分区 | **最早**（内核启动即加载） |
| vendor_dlkm | 402 | vendor_dlkm 分区 | first_stage 之后 |
| system_dlkm | 206 | system_dlkm 分区 | 系统启动后 |
| 合集去重 | **671** | — | — |

### 168 个 vendor_boot 独有基础模块（vendor_dlkm 没有）

这些是内核早期初始化的根基，缺失任何一个都会导致无法启动：

**存储 / 内存映射（卡第一屏的直接元凶）**
- `arm_smmu.ko`、`arm_smmu_pmu.ko`、`qcom_iommu_util.ko`、`qcom_iommu_debug.ko`
- `ufs-qcom.ko`、`ufshcd-crypto-qti.ko`、`cqhci.ko`、`phy-qcom-ufs*.ko`
- `mem_buf.ko`、`mem_buf_dev.ko`、`secure_buffer.ko`、`minidump.ko`

**时钟树（clk）**
- `clk-qcom.ko`、`clk-rpmh.ko`、`clk-scmi.ko`、`clk-dummy.ko`
- `gcc-canoe.ko`、`dispcc-canoe.ko`、`gpucc-canoe.ko`、`camcc-canoe.ko`、`videocc-canoe.ko`
- `tcsrcc-canoe.ko`、`cambistmclkcc-canoe.ko`、`debugcc-canoe.ko`、`evacc-canoe.ko`

**SoC 基础**
- `qcom-scm.ko`、`cmd-db.ko`、`socinfo.ko`、`llcc-qcom.ko`、`qcom-rpmh.ko`
- `qcom_aoss.ko`、`qcom-dcvs.ko`、`qcom-ipcc.ko`、`qcom-pdc.ko`
- `pinctrl-canoe.ko`、`pinctrl-msm.ko`、`gdsc-regulator.ko`、`rpmh-regulator.ko`
- `qcom_scmi_client.ko`、`qcom_scmi_vendor.ko`、`smem.ko`、`qrtr.ko`

**小米定制（内存/调度）**
- `millet_binder.ko`、`millet_core.ko`、`millet_hs.ko`、`millet_oem_cgroup.ko`、`millet_pkg.ko`、`millet_sig.ko`
- `mi_memory.ko`、`mi_mempool.ko`、`mi_damon.ko`、`mi_game.ko`、`mi-power.ko`、`mi_extent_pool.ko`
- `sched-walt.ko`（WALT 调度器）、`rtg.ko`、`vip_sched.ko`、`cpu_hotplug.ko`

---

## 三、fstab.qcom（第一阶段分区表）

`vendor_boot` 的 ramdisk 内含 `first_stage_ramdisk/fstab.qcom`，定义了完整的
分区挂载方案，关键点：

- **动态分区**（logical + first_stage_mount）：system/system_ext/product/vendor/
  vendor_dlkm/system_dlkm/odm/mi_ext
- **文件系统**：ext4 + erofs 双配置（当前固件用 erofs）
- **mi_ext overlay**：小米通过 overlay 挂载 mi_ext 到 product/system 的多个路径
- **加密**：userdata 用 `f2fs + inlinecrypt + aes-256-xts/cts + wrappedkey_v0`
- **AVB**：boot/init_boot/vendor_boot/dtbo/recovery 都带 `avb=vbmeta`

这份 fstab 是理解 pandora 分区布局和启动流程的关键，之前遗漏。

---

## 四、本次补齐的文件清单

| 新增文件 | 内容 |
|---------|------|
| `modules/vendor_boot_modules.txt` | 461 个 vendor_boot 早期模块清单 |
| `modules/vendor_boot_modules.load` | vendor_boot 加载顺序 |
| `modules/modules.load.recovery` | recovery 加载顺序 |
| `modules/fstab.qcom` | 第一阶段分区表 |
| `modules/README.md`（更新） | 补充三套模块说明 |
| `modules/kmi-analysis.md`（更新） | 补充 vendor_boot 依赖链 |

---

## 五、仓库仍缺少的内容（可选，非内核核心）

以下内容与内核源码/设备树无关，但属于完整的 pandora 移植参考，仓库里没有：

1. **vbmeta / AVB 结构**：boot.img 尾部的 AVB footer + vbmeta（分区校验相关）
2. **init_boot 的完整 ramdisk**（2.5MB，含 init/snapuserd，GKI 通用）
3. **vendor 分区本体**（589MB，含 sepolicy、firmware 等，非内核）

这些通常在 `android_device_xiaomi_pandora` 或 `-kernel` 仓库处理，不属于本内核仓库职责。
