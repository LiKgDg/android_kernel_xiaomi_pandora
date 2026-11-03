# pandora (SM8850) 设备树 (dts) 清单

以下 `.dts` 由原厂 OTA `OS4.0.5.0.XBLCNXM` 的 `vendor_boot` 分区中的 12 个 `.dtb`
二进制设备树反编译而来（`dtc -I dtb -O dts`）。原始二进制见同目录 `*.dtb` 备份。

## dtb → 硬件配置对应关系

| 文件 | model (SoC 变体) | 说明 |
|------|------------------|------|
| dtb_00 | Canoe Alt. Thermal Profile v2 | 标准热配置 v2 |
| dtb_01 | Canoe Alt. Thermal Profile | 标准热配置 |
| dtb_02 | Canoe v2 | 标准 v2 |
| dtb_03 | Canoe | 标准 |
| dtb_04 | CanoeP SG v2 | 性能版 v2 |
| dtb_05 | CanoeP SG | 性能版 |
| dtb_06 | CanoeP Alt. Thermal Profile v2 | 性能版热配置 v2 |
| dtb_07 | CanoeP Alt. Thermal Profile | 性能版热配置 |
| dtb_08 | CanoeP v2 | 性能版 v2 |
| dtb_09 | CanoeP | 性能版 |
| dtb_10 | Whale | 标准大屏 |
| dtb_11 | WhaleP | 性能大屏 |

> 注：Canoe/CanoeP/Whale/WhaleP 是高通 SM8850 平台的四个硬件变体代号，
> pandora（Xiaomi 17 Pro）量产机通常匹配其中某一种（以 `bootloader` 运行时
> 选择的 `board_id` 为准），完整保留 12 个可确保不同批次硬件均能启动。

## 反编译说明

反编译得到的 `.dts` 是「展开后的扁平设备树」（已含 include 的展开内容），
不含原始 `.dtsi` 头文件的 include 关系。用于阅读、对照、二次开发参考；
若要重新编译回 `.dtb`，需配合 GKI 源码树中的 `scripts/dtc/dtc`：

```bash
dtc -I dts -O dtb dtb_00.dts -o dtb_00.dtb
```
