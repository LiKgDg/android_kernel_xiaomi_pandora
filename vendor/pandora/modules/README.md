# pandora (SM8850) 内核模块（.ko）分析

以下内容由原厂 OTA `OS4.0.5.0.XBLCNXM` 的 `super.img` 动态分区中提取：

- `vendor_dlkm_a`（厂商驱动模块，56.6 MB，EROFS）
- `system_dlkm_a`（GKI 通用模块，16.4 MB，EROFS）

用 `payload-dumper-go` 提取分区 → `fsck.erofs --extract` 解出 `.ko` 文件。

## 文件清单

| 文件 | 内容 |
|------|------|
| `vendor_modules.txt` | vendor_dlkm 的 402 个厂商驱动模块名（高通 + 小米） |
| `system_modules.txt` | system_dlkm 的 GKI 通用模块名（103 个，另有 103 个 flatten 副本） |
| `modules.load` | 开机加载顺序（665 个模块，vendor + system 合并） |
| `modules.dep` | 模块间依赖关系 |
| `modules.softdep` | 软依赖（softdep）声明 |
| `kmi-analysis.md` | KMI/vermagic 与关键模块依赖链分析 |

## 模块分类统计

| 分类 | 数量 | 说明 |
|------|------|------|
| vendor_dlkm 总计 | 402 | 厂商驱动 |
| ├─ 小米定制（mi_/mca_/xiaomi_） | 59 | mca 充电全套、mi_fp/mi_cam 等 |
| ├─ 高通平台（qcom_/qti_） | 37 | glink/scm/thermal 等 |
| ├─ 音频（wcd/lpass/swr） | 22 | codec、宏、swr |
| └─ 其他 | 284 | GPU/显示/相机/存储/Wi-Fi 等 |
| system_dlkm 总计 | 206 | GKI 通用（103 结构化 + 103 flatten） |

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
