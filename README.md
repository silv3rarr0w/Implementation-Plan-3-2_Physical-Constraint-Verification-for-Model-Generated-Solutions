# 无机粉体材料热力学-动力学高通量筛选平台

基于 Materials Project 数据库 + MACE-MP-0 通用力场，对目标化学体系进行**凸包热力学初筛** → **温度/气氛校验** → **CI-NEB 动力学反应路径**三级递进分析。

---

## 目录结构

```
粉体项目调研/
├── api/
│   └── myapi.env                 # API Key 配置（不纳入版本控制）
├── 方案1/                         # 凸包热力学初筛
│   ├── 凸包计算.ipynb             # ★ 主 notebook
│   ├── output.md                  # 输出报告
│   └── 校准/
│       └── 凸包计算验证.ipynb     # 校准验证
├── 方案2/                         # 温度 + 气氛校验
│   ├── 温度气氛校验.ipynb         # ★ 主 notebook
│   └── temperature_analysis.md    # 输出报告
├── 方案3/                        # CI-NEB 动力学反应路径
│   └── 动力学计算.ipynb           # ★ 主 notebook
├── ref/                           # 参考文献与方法论说明
├── requirements.txt               # 完整 Python 依赖
└── README.md
```

---

## 快速开始

### 1. 环境准备

```bash
# 安装依赖
pip install -r requirements.txt

# GPU 加速（可选，MACE-MP-0 弛豫会显著加速）
pip uninstall torch -y
pip install torch --index-url https://download.pytorch.org/whl/cu124
```

### 2. 配置 API Key

在目录 `api/` 下创建 `myapi.env`：

```
MP_API_KEY=你的Materials_Project_API密钥
```

> API Key 可在 [Materials Project](https://next-gen.materialsproject.org/) 注册获取。

### 3. 运行顺序

三个 notebook 按流水线顺序运行，**方案1的输出是方案2的输入**：

```
方案1 凸包计算.ipynb        →  方案2 温度气氛校验.ipynb    →  方案3 动力学计算.ipynb
     ↓                               ↓                              ↓
 scheme1_export.json           (读取方案1的JSON)              (独立运行)
(214条条目含MACE预测)
```

#### 方案1: 凸包热力学初筛

| 步骤 | 说明 |
|------|------|
| 修改 `system` 变量 | 目标化学体系（默认 `["Li","Ge","P","S"]`） |
| 运行 Cell 5 | 从 Materials Project 拉取数据 + 构建 0K 凸包（含重连机制） |
| 运行 Cell 7 | MACE-MP-0 预测 MP 缺失结构 + 合并凸包 |
| 运行 Cell 8 | 导出 `scheme1_export.json` 供方案2使用 |
| 输出 | `output.md` — 形成能 + e_above_hull 表格 |

> **调试建议**：首次运行可将 Cell 2 的 `DEBUG_MODE` 设为 `True`，仅计算 3 个缺失结构快速验证。

#### 方案2: 温度 + 气氛校验

| 步骤 | 说明 |
|------|------|
| 修改 `temperatures_C`、`ATMOSPHERES` | 目标烧结温度和气氛条件 |
| 运行 Cell 6 | 自动检测并加载方案1的 `scheme1_export.json`（若不存在则回退 MP API） |
| 运行后续 Cell | Bartel SISSO 高温修正 + 气氛依赖凸包分析 |
| 输出 | `temperature_analysis.md` — 温度-气氛稳定性趋势 + 分解路径 |

#### 方案3: CI-NEB 动力学反应路径

| 步骤 | 说明 |
|------|------|
| 修改 `TARGET_FORMULA` | 目标化合物化学式 |
| 修改 `DEBUG_MODE` | `True`=调试模式(仅跑前N条边), `False`=完整运行 |
| 运行全部 Cell | 两阶段 CI-NEB（粗筛→精修）+ KSP 图搜索 |
| 输出 | `kinetics_network_report.md` — 合成/分解最优路径 |

★该步骤耗时较长，强烈建议打开gpu加速

★初次运行建议打开debug mode可以显著缩短运行时间，以验证代码可行性，之后再关闭以得到完整结果

---

## 方法概要

| 方案 | 方法 | 核心工具 | 参考文献 |
|------|------|----------|----------|
| 方案1 | 0K 凸包 + ML 力场补全 | pymatgen PhaseDiagram + MACE-MP-0 | — |
| 方案2 | SISSO 高温 Gibbs 修正 + 气氛化学势 | Bartel et al. (2018) | [*Nat. Commun.* 2018](https://doi.org/10.1038/s41467-018-06682-4) |
| 方案3 | 粗→细 CI-NEB + KSP 反应网络 | ASE NEB + rxn-network | [*Nat. Commun.* 2021](https://doi.org/10.1038/s41467-021-23339-x) |

---

## 配置说明

| 文件 | 配置项 | 说明 |
|------|--------|------|
| `api/myapi.env` | `MP_API_KEY` | Materials Project API 密钥 |
| 方案1 Cell 2 | `system`, `HULL_THRESHOLD`, `DEBUG_MODE` | 化学体系 + 近稳阈值 + 调试开关 |
| 方案2 Cell 2 | `temperatures_C`, `ATMOSPHERES` | 烧结温度 + 气氛条件 |
| 方案三 Cell 3 | `TARGET_FORMULA`, `DEBUG_MODE` | 目标化合物 + 调试开关 |

---

## 故障排查

| 问题 | 原因 | 解决 |
|------|------|------|
| MACE-MP-0 运行慢 | CPU 模式 | 安装 CUDA 版 torch |
| `MP_API_KEY missing` | API Key 未配置 | 检查 `api/myapi.env` |
| 获取 MP 数据报错 | 网络不稳定 | 重新运行该cell |
