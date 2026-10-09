# personal-workflow

个人工作流（SOP）沉淀仓库。把日常反复用到的流程固化成文档，做硬件 / 写软件时直接照方抓药，减少重复决策与踩坑。

## 目录结构

```
personal-workflow/
├── README.md
└── workflows/
    ├── 01-pcb-full/                 # PCB 全流程：设计 → 采购装配 → 上电验证
    │   ├── README.md
    │   ├── 01-circuit-design.md
    │   ├── 02-procurement-assembly.md
    │   └── 03-bringup-validation.md
    └── 02-board-software/           # 板级软件开发流程
        └── README.md
```

## 工作流清单

| 编号 | 名称 | 说明 |
| :--: | :-- | :-- |
| 01 | [PCB 全流程](workflows/01-pcb-full/README.md) | 从原理图到板子上电的完整落地流程（设计 / 采购装配 / 上电验证） |
| 02 | [板级软件开发](workflows/02-board-software/README.md) | 板级软件开发流程（待补充） |

> 约定：每个工作流放在 `workflows/<编号>-<名称>/` 下，主说明放该目录的 `README.md`，各阶段拆成 `NN-<阶段>.md`。
