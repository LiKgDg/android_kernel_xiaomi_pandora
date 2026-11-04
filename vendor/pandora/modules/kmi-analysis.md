# pandora (SM8850) KMI / vermagic 与关键模块依赖链

> 数据来源：原厂 OTA `OS4.0.5.0.XBLCNXM` 的 `vendor_dlkm` 分区内 `.ko` 模块，
> 用 `modinfo` / `strings` 提取。

## 内核身份（与 boot.img 一致）

```
Linux version 6.12.69-android16-6-g43b5ab7e247d-abogki561813157-4k
vermagic:   6.12.69-android16-6-4k SMP preempt mod_unload modversions aarch64
```

## 关键模块 vermagic（抽查）

| 模块 | vermagic |
|------|----------|
| msm_kgsl.ko | `6.12.69-android16-6-4k SMP preempt mod_unload modversions aarch64` |
| xiaomi_touch.ko | 同上（`-4k` 结尾，无 `+` 脏后缀） |

> 注意：模块 vermagic 以 `-4k` 结尾（非 `-4k+`），说明编译时 `LOCALVERSION`
> 已显式设置、且未触发 `scripts/setlocalversion` 的 dirty 检测。

## msm_kgsl.ko 的完整依赖链（18 项）

GPU 驱动（Adreno 840）依赖以下符号提供者，**全部位于 vendor_dlkm**：

```
depends: socinfo, llcc-qcom, qcom-dcvs, qcom-scm, cmd-db, msm_performance,
         qcom_aoss, os_cpu_qi, msm_sysstats, mem_buf_dev, gpu_stats,
         qcom_iommu_util, clk-qcom, secure_buffer, mdt_loader, minidump,
         qcom_va_minidump, miev, coresight, synx-driver
```

软依赖（modules.softdep）：

```
msm_kgsl pre: arm_smmu nvmem_qfprom socinfo governor_msm_adreno_tz
              governor_gpubw_mon governor_msm_adreno_ro
```

## 换纯 ACK 内核「卡第一屏」的根因

显示与存储的关键驱动全部是 vendor_dlkm 里小米编译的 `.ko`，且依赖链完整：

| 驱动 | 模块 | 关键依赖 |
|------|------|----------|
| GPU | msm_kgsl.ko | socinfo / llcc-qcom / qcom-scm / arm_smmu / nvmem_qfprom |
| 显示 | msm_drm.ko | drm_display_helper / qti_amoled_ecm / hwmon |
| 存储 | ufs_ffu.ko | 高通私有 UFS 固件接口 |
| Wi-Fi | cnss2.ko / qca_cld3_wcn7750.ko | icnss2 / wlan_firmware_service |

纯 ACK（AOSP `kernel/common`）源码不含这些高通私有 vendor 模块，也无法承载
这条依赖链，因此换内核后显示与存储无法初始化，直接卡第一屏。

## 结论

- 本仓库定位为「可编译 GKI 基线 + 提取的 defconfig / 设备树 / 模块清单」，
  用于**研究、对照、二次开发**，而非完整复刻官方内核（需高通私有补丁）。
- 若要编译可加载 vendor 模块的内核，`module_layout` CRC 必须与 `CONFIG_MODVERSIONS`
  下的原厂内核一致（判据是 CRC，不是 vermagic 字符串）。
