# A题《面向算电协同的多目标调度优化研究》项目说明（README）

本目录为 2026 暑期培训第二次模拟题 A 题（面向算电协同的多目标调度优化研究）的完整建模与求解工程，包含四问的模型代码、结果数据、图表与论文生成流程。本文档说明**目录结构、环境配置、代码运行指南**，并给出**修改前后的详细对比**，方便他人快速复现与理解。

---

## 一、项目概述

以六个典型区域（RegionA–F）、5 万个 AI 任务轨迹与逐时电力参数为输入，构建"统计 → 预测 → 基础调度 → 碳感知调度 → 储能协同 → 算—储—电协同"的递进式模型体系，四个问题分别求解并输出调度方案、指标与图表，最终生成竞赛论文（docx / tex / pdf）。

- 时间口径：主时域 0–2399 时，收尾 2400–2405 时，2406 时仅做电力/储能结算
- 任务规则：实时推理到达即开工（时延≤20ms）；批量（≤80ms）与训练（≤150ms）可延迟，须在 2406 前完成；不可抢占/拆分/中途迁移
- 功率映射：实时 0.08 / 批量 0.10 / 训练 0.16 MW·GPU⁻¹

---

## 二、目录结构

```
A题 面向算电协同的多目标调度优化研究/
├── 附件数据/                      # 赛题原始数据（6 张 xlsx，只读）
├── A题 面向算电协同的多目标调度优化研究.pdf   # 赛题正文
├── README.md                     # 本文档
└── cumcm_work/                   # 工作区（本工程）
    ├── code/                     # 全部模型代码（Python）
    ├── results/                  # 结果数据（CSV/JSON）
    ├── figures/                  # 图表（F1–F14，PDF+PNG）
    ├── paper/                    # 论文源（manuscript.md + tables + references.bib）
    ├── build/                    # 构建输出（paper.docx / .pdf / .tex / .resolved.md）
    ├── submission/               # 支撑材料包（supporting-materials.zip）
    ├── validation/               # 校验报告（数据审计/溯源/跨格式/PDF规则/匿名/视觉）
    └── project-state.json        # cumcm-modeling 技能门禁状态（7 项全绿）
```

### 代码文件清单（`code/`）

| 文件 | 作用 | 输出 |
|---|---|---|
| `common.py` | 统一数据加载与口径（BASE 路径、公式） | — |
| `style.py` | 全局绘图样式（中文字体、配色） | — |
| `q1_stats.py` | 问题一统计画像 | F1、F2；q1_stats.csv、q1_arrival.csv |
| `q1_forecast.py` | 问题一短期预测（分位数 LightGBM） | F5、F6；q1_forecast.csv、q1_metrics.csv |
| `q1_schedule.py` | 问题一末窗口 MILP 调度（HiGHS） | F3、F4；q1_schedule.csv、q1_util.csv |
| `q2_schedule.py` | 问题二碳感知调度（启发式+修复）+ 扫描 | F7–F9；q2_schedule.csv、q2_compare.csv、q2_pareto.csv、q2_carbon_sweep.csv |
| `q2_outputs.py` | 从已存调度快速重生成 Q2 输出（不用重跑贪心） | q2_compare.csv、q2_pareto.csv、F7–F9 |
| `q3_storage.py` | 问题三储能 LP（每区域独立） | F10、F11；q3_storage.csv、q3_compare.csv |
| `q4_joint.py` | 问题四场景矩阵（分解求解） | q4_scenarios.csv、q4_sched_*.csv |
| `q4_figures.py` | 问题四图表 | F12–F14 |
| `make_ledger.py` | 生成结果台账 summary.json / result-ledger.json | results/summary.json |
| `make_tables.py` | 生成论文表格源（T3–T6 从结果 CSV） | paper/tables/T*.md |

---

## 三、环境准备

### 3.1 Python 依赖

推荐 Python 3.13（已用 3.13 验证）。安装依赖：

```powershell
python -m pip install numpy pandas scipy matplotlib openpyxl lightgbm
```

> 说明：MILP 用 `scipy.optimize.milp`（HiGHS）；储能 LP 用 `scipy.optimize.linprog`（HiGHS）；预测用 LightGBM 分位数回归。

### 3.2 论文构建工具（仅生成 docx/tex/pdf 需要）

- **pandoc**：`python -m pip install pypandoc-binary`（自带 pandoc 二进制）
- **xelatex**：安装 TinyTeX 或 TeX Live（需含 xeCJK/ctex 以支持中文），并将 `bin/windows` 加入 PATH
- **poppler**（PDF 校验用，可选）：提供 `pdftotext/pdftoppm/pdfinfo`，加入 PATH

构建论文时的关键环境变量（Windows）：

```powershell
$env:HOME = "C:\Users\<用户名>"
$env:TEXMFHOME = "<TinyTeX 路径>\texmf-home"   # 可写目录
$env:TEXMFVAR  = "<TinyTeX 路径>\texmf-var"
$env:TEXMFCONFIG = "<TinyTeX 路径>\texmf-config"
$env:PYTHONUTF8 = "1"
$env:PYTHONIOENCODING = "utf-8"
```

---

## 四、数据说明

数据位于 `附件数据/`，共 6 张表：

| 表 | 内容 |
|---|---|
| workload_trace.xlsx | 5 万任务：类型、到达时、GPU 需求、时长、来源区域、时延上限、最晚完成 |
| GPU_information.xlsx | 区域 GPU 容量、IT/设施功率、PUE |
| network_latency.xlsx | 区域间单向时延矩阵 |
| region_time_data.xlsx | 逐时电价/售电价/碳强度/可用新能源/基准负荷与运行结果（0–2406） |
| power_mapping.xlsx | 三类任务单位 GPU 功率 |
| storage_information.xlsx | 储能容量、SOC 边界、充放电功率/效率、外送与购售电上限 |

`common.py` 中的 `BASE` 指向附件数据绝对路径，若目录移动需同步修改。

---

## 五、代码运行指南（按顺序）

> 所有脚本在 `cumcm_work/code/` 下执行；结果写入 `cumcm_work/results/`，图写入 `cumcm_work/figures/`。

### 第 1 步：问题一（统计 + 预测 + 调度）

```powershell
python code/q1_stats.py        # 统计画像 -> F1 F2
python code/q1_forecast.py     # 预测 -> F5 F6（需要 lightgbm）
python code/q1_schedule.py     # 末窗口 MILP 调度 -> F3 F4（约 10 分钟）
```

### 第 2 步：问题二（碳感知调度）

```powershell
python code/q2_schedule.py     # 完整运行：主方案 + 时延扫描 + 碳扫描（较慢，约 1–2 小时）
```

> ⚠️ **耗时提示**：`q2_schedule.py` 内部调用 5 次贪心+修复（每次约 13–26 分钟）。若只更新输出/图表而不重算调度，用快速脚本：

```powershell
python code/q2_outputs.py      # 读取已存 q2_schedule.csv，重建 q2_compare/pareto 与 F7–F9（约 30 秒）
```

### 第 3 步：问题三（储能 LP）

```powershell
python code/q3_storage.py      # 每区域 2 个 LP（无储能/储能最优），约 8–10 分钟
```

### 第 4 步：问题四（协同 + 场景矩阵）

```powershell
python code/q4_joint.py        # 7 个场景（含 2 次重调度），约 20–40 分钟
python code/q4_figures.py      # F12–F14
```

### 第 5 步：台账、表格与论文

```powershell
python code/make_ledger.py     # 生成 results/summary.json 与 result-ledger.json
python code/make_tables.py     # 生成 paper/tables/T3–T6.md
```

论文构建（需 pandoc + xelatex，见 3.2）：

```powershell
python "<skill>/scripts/build_paper.py" --manifest cumcm_work/paper-manifest.json --root cumcm_work --report cumcm_work/validation/paper-build.json
```

构建产物：`cumcm_work/build/paper.docx`、`paper.pdf`、`paper.tex`、`paper.resolved.md`。

### 运行顺序小结

```
q1_stats → q1_forecast → q1_schedule
        → q2_schedule（或 q2_outputs 快速路径）
        → q3_storage
        → q4_joint → q4_figures
        → make_ledger → make_tables → build_paper
```

---

## 六、修改前后详细对比

本轮迭代针对初版（修改前）的缺陷进行了三处核心修复与若干增强。下表为**修改前（原始交付版）与修改后（当前版）**的逐项对比。

### 6.1 问题一：调度负载均衡（核心修复）

| 维度 | 修改前 | 修改后 |
|---|---|---|
| 调度目标 | 最小化"最晚完成 + 微小负载项"（负载项形同虚设） | **最小化峰值利用率 $U_{\max}$**（显式负载均衡） |
| 末 24 时利用率 | A 11% / **B 0% / C 0%** / D 90% / E 50% / F 5% | **A 37% / B 35% / C 37% / D 39% / E 40% / F 40%**（六区均衡） |
| 峰值利用率 | 局部接近 100% | 全部 43%–47% |
| 迁移率 | 71.9% | 74.0% |
| 可行性 | 0 越限 | 0 越限 |

### 6.2 问题二：西迁策略与双口径（核心修复）

| 维度 | 修改前 | 修改后 |
|---|---|---|
| 成本口径 | 仅可再生优先（边际购电≈0） | **双口径**：可再生优先 + 敏感性口径（按基准可再生剖面） |
| 主方案迁移率 | **≈0%（任务全留本地）** | **63.4%（λ=3）**；λ=0 时 76.4% |
| GPU-h 加权平均时延 | 5.0ms | 23.3ms（λ=3）；28.3ms（λ=0） |
| 成本/碳（可再生优先） | −4.55 亿元 / 碳 0 | −4.38 亿元 / 碳 0 |
| 敏感性成本/碳 | 未报告 | **14.95 亿元 / 133.4 万吨**（较基准降 17.0% / 34.7%） |
| 算法机制 | 正序贪心，迁移仅由容量被迫 | **倒序处理晚到任务 + 容量压力 + 收尾预留 + 占位腾挪修复** |
| 可行性 | 0 越限（但不迁移） | **0 越限（在真实西迁方案下）** |

### 6.3 问题三：储能口径与利用率输出

| 维度 | 修改前 | 修改后 |
|---|---|---|
| 储能目标 | 归一化加权（成本/碳/峰值/波动） | 纯成本目标（结果基本一致，成本本就主导） |
| 储能最优成本 | −39,143 万元 | −39,143 万元（量级不变） |
| 无储能成本 | −34,549 万元 | −34,549 万元 |
| 新能源利用率 | **未输出** | **东部 78%、西部/新能源区 97%–99%** |
| 约束核验 | 未展示 | 外送≤SellLimit、SOC 界内、购电=0 均已核验 |

### 6.4 问题四：协同与场景

| 维度 | 修改前 | 修改后 |
|---|---|---|
| 基准场景 | 迁移 0%、时延 5ms、成本 −3.91 亿 | **迁移 63.4%、时延 23.3ms、成本 −3.62 亿** |
| 新能源 −30% | 碳 3.34 万吨 / 峰值 214MW | 碳 3.30 万吨 / 峰值 509MW |
| 碳约束 60% | 几乎不降碳（3.34→3.34 万） | **3.30→3.20 万吨（真实生效）** |
| 可行性 | 0 越限 | 全部场景 0 越限 |

### 6.5 论文与过程

| 维度 | 修改前 | 修改后 |
|---|---|---|
| 论文公式 | **无数学公式** | **16 个 LaTeX 公式块**（统一口径、Q1 MILP、Q2 多目标+敏感性、Q3 LP、Q4 加权+ε-约束） |
| 论文页数 | 14 页 | 17 页 |
| 合规校验 | 通过（当时版本） | 全部通过（跨格式 38 锚点 / PDF 规则 17 页 / 匿名 0 / 视觉 17/17 / 支撑包 74 文件） |
| 门禁（cumcm-modeling） | — | 7/7 全绿（含 export） |

### 6.6 不变的部分

- 统一口径公式（AI IT 负荷、GPU-hour、碳排、电费、新能源利用率、SOC）
- Q1 预测模型与指标（聚合 MAPE 34.8%、区间覆盖率）
- Q1 最晚完成时点 2405.62
- Q3 基准成本 18.02 亿元、储能收益量级

---

## 七、常见问题（FAQ）

1. **Q2 运行太慢怎么办？**
   使用 `python code/q2_outputs.py` 基于已存调度快速重建输出；若要重算调度，可接受较长耗时或调小 `run_schedule` 的 `max_delay_batch/train` 与修复迭代次数。

2. **跑 q1_forecast.py 报 lightgbm 缺失？**
   执行 `python -m pip install lightgbm`。

3. **论文 PDF 中文乱码/编译失败？**
   确认 xelatex 可用且已装 `xeCJK`/`ctex`（`tlmgr install xeCJK ctex`），并设置 3.2 节的环境变量（HOME/TEXMF*）。

4. **路径变更？**
   `common.py` 的 `BASE` 为附件数据绝对路径；`q2_schedule.py`/`q3_storage.py` 内对附件数据的读取也基于同一 BASE，移动目录时统一修改 `common.py` 即可。

5. **如何复现论文中的数字？**
   依次运行第 5 节脚本生成 `results/` 数据，再运行 `make_ledger.py` 生成 `summary.json`（论文中的头条数值由台账占位符绑定，构建时自动填入）。

---

## 八、许可与致谢

- 数据来源：赛题附件（仅用于本次模拟训练）
- 论文构建与门禁流程基于 cumcm-modeling 技能
- 求解依赖：scipy/HiGHS、LightGBM、pandoc、TinyTeX/TeX Live、poppler
