# android_kernel_xiaomi_pandora

Xiaomi 17 Pro（`pandora` / SM8850）的 **GKI 内核源码树**，配合 LineageOS 移植项目
`android_device_xiaomi_pandora` / `android_device_xiaomi_pandora-kernel` 使用。

> 本仓库 = **谷歌官方 GKI 源码**（`android16-6.12` 分支）+ **pandora 专属提取内容**。

---

## 内核身份

```
Linux version 6.12.69-android16-6-g43b5ab7e247d-abogki561813157-4k
  (kleaf@build-host) ... clang version 19.0.1 ... #1 SMP PREEMPT
```

| 项 | 值 |
|---|---|
| 内核版本 | 6.12.69-android16-6 |
| KMI 世代 | 6.12.69-android16-6 |
| 页大小 | 4k（`CONFIG_ARM64_4K_PAGES=y`） |
| 源码基准 | AOSP `kernel/common` `android16-6.12` 分支（HEAD `4ad1cb12b`） |
| 编译链 | Android clang 19.0.1 + LTO + PGO + BOLT |
| 来源固件 | 原厂 OTA `OS4.0.5.0.XBLCNXM` |

---

## 目录结构

```
├── arch/ ...                    # GKI 官方源码（android16-6.12，未改动）
├── vendor/pandora/              # pandora 专属提取内容
│   ├── configs/
│   │   └── pandora_defconfig    # 从 OTA 内核内嵌 IKCONFIG 提取的完整 .config
│   ├── dts/
│   │   ├── dtb_00~11.dts        # 12 个设备树反编译源码（Canoe/CanoeP/Whale/WhaleP）
│   │   ├── dtb_00~11.dtb        # 原始二进制设备树（备份）
│   │   └── README.md            # dtb → 硬件配置对应关系
│   └── dtbo/
│       ├── dtbo_00.dtbo         # 板级设备树叠加层（二进制）
│       └── dtbo_01.dtbo
└── README.pandora.md            # 本文件
```

---

## 与 LineageOS 移植项目的关系

| 仓库 | 职责 |
|------|------|
| `android_device_xiaomi_pandora` | 机型树（BoardConfig.mk / overlays） |
| `android_device_xiaomi_sm8850-common` | SM8850 通用设备配置 |
| `android_device_xiaomi_pandora-kernel` | 预编译内核提取规格（清单 + 校验脚本，不含二进制） |
| **本仓库** | **GKI 内核源码 + defconfig + 设备树（可编译参考）** |

现有移植项目采用**预编译内核**方案（`TARGET_NO_KERNEL_OVERRIDE := true`），内核二进制
由使用者从自己的 OTA 提取。本仓库提供**源码级参照**，用于：
1. 对照分析内核行为、符号、KMI 接口
2. 二次开发时从源码编译自定义内核
3. 研究设备树（dts）与驱动配置

---

## 从源码编译（可选）

```bash
# 本仓库直接 make 编译参考（需 Android 交叉工具链）
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-   # 或 Android 预编译 clang
make O=out gki_defconfig                  # 官方 GKI 基线
# 或使用提取的完整配置（含高通 vendor 项，部分依赖私有补丁）
make O=out pandora_defconfig
make O=out -j$(nproc)
```

> 注：原厂内核带高通私有的 vendor hook 与模块，完整复现官方内核需高通的
> kernel 补丁与私有头文件，官方 GKI 源码无法 100% 复刻。本仓库以「可编译 GKI
> 基线 + 提取配置/设备树」为定位，用于研究与二次开发。

---

## KMI 说明（重要）

- 内核与模块必须同代（KMI 世代一致），否则加载报 `version magic` / `module_layout` 不匹配。
- 判据是 `module_layout` 的 CRC（`CONFIG_MODVERSIONS=y`），不是 vermagic 字符串。
- 详见 `android_device_xiaomi_pandora-kernel` 仓库的 README（含 `kmi-gate.sh` 验证方法）。

---

## 源码版权说明

- `arch/`、`include/`、`kernel/` 等目录的源码版权归 Linux 内核社区 / Google / 高通，
  遵循各自的 SPDX 许可（以文件头标注为准）。
- `vendor/pandora/` 下的 `pandora_defconfig`、反编译的 `.dts` 由原厂 OTA 提取，
  仅供学习研究使用。
