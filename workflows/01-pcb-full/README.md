# 01 · PCB 全流程

把一块 PCB 从「想法 / 原理图」做到「上电可用」的完整流程，分三个阶段。

## 阶段划分

| 文件 | 中文定位 | 核心内容 |
| :-- | :-- | :-- |
| [01-circuit-design.md](01-circuit-design.md) | 电路与 PCB 设计 | 器件选型、原理图、PCB Layout、DRC |
| [02-procurement-assembly.md](02-procurement-assembly.md) | 采购与装配 | BOM、元件匹配、PCB 打板、PCBA、焊接 |
| [03-bringup-validation.md](03-bringup-validation.md) | 上电调试与验证 | 上电检查、程序烧录、软硬件联调、功能验收 |

## 全流程

```
① 电路与 PCB 设计  ──▶  ② 采购与装配  ──▶  ③ 上电调试与验证
   01-circuit-design      02-procurement      03-bringup
```

## 阶段交接物

- **① → ②**：BOM 清单、坐标文件（覆盖全部贴片件）、Gerber。
- **② → ③**：焊接 / 贴片完成的实物板卡。
- **③ → 完成**：功能验收记录、故障 / 返工记录。
