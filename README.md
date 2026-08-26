# 无机粉体材料热力学-动力学高通量筛选平台

基于 Materials Project 数据库 + MACE-MP-0 通用力场，对目标化学体系进行**凸包热力学初筛** → **温度/气氛 Gibbs 校验** → **图论反应网络动力学评价**三级递进分析。

方案3 采用 McDermott et al. (2021) 的图论方法：用 Softplus 热力学成本函数替代 CI-NEB 势垒计算，大幅降低计算开销。

---

## 1. 项目简介

- 输入：目标化合物（如 `YMnO3`、`YBa2Cu3O6.5`、`MgMo3(PO4)3O`）及其化学体系
- 输出：
  - 0K 凸包稳定性（方案1）
  - 多温度/气氛稳定性与分解路径（方案2）
  - 反应网络 + KSP 路径 + 线性组合报告（方案3）
- 支持两种场景：
  - **已知总反应**：验证给定配方/复现论文路径
  - **未知总反应（探索）**：为新化合物设计合成路线

---

## 2. 工作流总览

```
方案1 (0K 凸包 + MACE-MP-0) → scheme1_export.json
   ↓
方案2 (多温度/气氛稳定性分析) → temperature_analysis.md
   ↓
方案3 (从 scheme1 0K 现场构建 Gibbs → 反应网络/KSP/线性组合)
```

- 方案1 是唯一的数据源头（0K DFT/MACE 条目）；
- 方案2 只做稳定性筛选与报告，**不再导出** scheme2 给方案3；
- 方案3 在 `T_SYNTHESIS_C`（°C，内部自动转整数 K）下用参考管线（Bartel SISSO + FactSage + NIST）现场构建 Gibbs 条目，再进行路径搜索。

### 目录结构

```
粉体项目调研/
├── api/
│   └── myapi.env                              # API Key 配置
├── 方案1/
│   ├── 凸包计算_v2.ipynb                       # ★ 主 notebook
│   ├── scheme1_export.json                    # 0K 条目（方案3 唯一数据源）
│   └── output.md                              # 形成能 + e_above_hull 报告
├── 方案2/
│   ├── 温度气氛校验_v2.ipynb                   # ★ 主 notebook
│   └── temperature_analysis.md                # 温度-气氛稳定性报告
├── 方案3/
│   ├── 动力学计算_v2.ipynb                     # ★ 主 notebook（图论方法）
│   └── kinetics_network_v2_report.md          # KSP 路径报告
├── legacy/                                    # v1 旧版本
├── ref/                                       # 参考文献 PDF
├── reaction-network/                          # 参考代码仓库（rxn_network）
├── 方案要求/                                  # 课题需求文档
├── requirements.txt
└── README.md
```

---

## 3. 环境与依赖

```bash
# conda 环境（示例名 DFT）
conda create -n DFT python=3.11 -y
conda activate DFT
pip install -r requirements.txt

# GPU 加速（可选）
pip uninstall torch -y
pip install torch --index-url https://download.pytorch.org/whl/cu124
```

主要依赖：

- pymatgen / mp-api
- mace-torch / ase
- rxn_network（含 NIST 数据）
- networkx / scipy / matplotlib

### API Key

在 `api/` 下创建 `myapi.env`：

```
MP_API_KEY=你的Materials_Project_API密钥
```

---

## 4. 快速开始

### 运行顺序

```
方案1 凸包计算_v2.ipynb → 方案2 温度气氛校验_v2.ipynb（可选）→ 方案3 动力学计算_v2.ipynb
```

> 方案3 直接读取方案1 的 `scheme1_export.json`，不需要方案2 导出文件。

### 最小示例：YBCO 已知模式验证

方案3 配置（化学体系无需配置，自动从方案1 scheme1_export.json 同步）：

```python
TARGET_FORMULA = "Ba4Y2Cu6O13"
NET_REACTION_KNOWN = True
MANUAL_PRECURSORS = ["Y2O3", "BaO2", "CuO"]
KNOWN_BYPRODUCTS = ["O2"]
T_SYNTHESIS_C = 927                    # 单位：°C（≈1200 K，四舍五入整数）
E_ABOVE_HULL_CUTOFF = 0.1
```

运行后输出 `kinetics_network_v2_report.md`，可对比论文 Table 4。

---

## 5. 方案1：凸包计算

**文件**：`方案1/凸包计算_v2.ipynb`

| 配置项 | 说明 |
|--------|------|
| `TARGET_SYSTEM` | 目标产物体系 |
| `PRECURSOR_ELEMENTS` | 前驱体特有元素（用于扩展体系） |
| `MACE_ENABLED` | MACE-MP-0 缺失结构预测开关 |
| `HULL_THRESHOLD` | 近稳定相筛选阈值 (eV/atom) |
| `AUTO_EXTEND_FROM_MP_COVERAGE` | 陌生体系按 MP 覆盖度自动扩展 |

输出：`scheme1_export.json`（0K 绝对总能 + 结构）+ `output.md`。

---

## 6. 方案2：温度气氛校验

**文件**：`方案2/温度气氛校验_v2.ipynb`

| 配置项 | 说明 |
|--------|------|
| 化学体系 | 自动从方案1 scheme1_export.json 同步（无需配置） |
| `temperatures_C` | 温度扫描范围（摄氏度） |
| `ATMOSPHERES` | 气氛条件（pO2/pH2O/pCO2） |
| `E_ABOVE_HULL_DECOMP_THRESH` | 分解判定阈值 |

功能：多温度/气氛稳定性曲线、分解路径分析，输出 `temperature_analysis.md`。

> 本方案不参与方案3 的数据传递，仅用于选择合成温度/气氛。

---

## 7. 方案3：动力学计算

**文件**：`方案3/动力学计算_v2.ipynb`

| 配置项 | 说明 |
|--------|------|
| `TARGET_FORMULA` | 目标化合物（pymatgen 约化式） |
| 化学体系 | 自动从方案1 scheme1_export.json 同步（无需配置） |
| `NET_REACTION_KNOWN` | 已知/探索模式 |
| `T_SYNTHESIS_C` | 合成温度，单位：°C（方案3 唯一温度入口，内部自动转 K） |
| `E_ABOVE_HULL_CUTOFF` / `_UNKNOWN` | 凸包过滤阈值 |
| `MANUAL_PRECURSORS` / `KNOWN_BYPRODUCTS` | 已知模式专用 |
| `MAX_REACTANTS` / `K_SHORTEST` | 网络构建与 KSP 参数 |
| `MAX_LINCOMB_COMBO` / `MAX_LINCOMB_CANDIDATES` / `MAX_INTERMEDIATE_RXNS` | 线性组合：最大步数 / 候选上限 / 中间相反应补充上限 |
| `USE_OPEN_ELEMENT` | 开放元素（巨势） |

流程：

1. 从方案1 加载 0K 条目；
2. `GibbsEntrySet.from_computed_entries(entries, T_SYNTHESIS, ...)` 现场构建 Gibbs；
3. 凸包过滤（`filter_by_stability`）；
4. 反应枚举（≤2 反应物）；
5. 官方 `rxn_network.ReactionNetwork` + Yen KSP（只搜索已知目标，对齐论文）；
6. 交叉反应 + 线性组合（Numba 批量配平 + joblib 并行，Ray-free，Windows 兼容）；
7. 输出报告（含 Balanced Pathways，与论文 Table 同口径）。

---

## 8. 已知反应模式 vs 探索模式

| 维度 | 已知总反应模式 | 探索模式 |
|------|--------------|---------|
| 前驱体 | 手动指定（`MANUAL_PRECURSORS`） | 不指定 |
| 副产物 | 手动指定（`KNOWN_BYPRODUCTS`） | 自动推导 |
| 搜索范围 | 前驱体 → 目标(+副产物) | 所有能生成目标的反应 |
| 用途 | 验证配方 / 复现论文 | 新化合物路线设计 |
| 典型体系 | YMnO₃、YBCO | MgMo₃(PO₄)₃O |
| 关键开关 | `NET_REACTION_KNOWN = True` | `NET_REACTION_KNOWN = False` |

探索模式额外过滤：**metathesis 式过滤**——保留副产物不含目标元素的反应。

---

## 9. 验证案例

| 体系 | 模式 | 论文对照 |
|------|------|---------|
| YMnO₃ | 已知总反应 | Table 1 |
| YBa₂Cu₃O₆.₅ (YBCO) | 已知总反应 | Table 4 |
| MgMo₃(PO₄)₃O | 探索模式 / 定向验证 | Eq. 7/8/9 |
| Fe₂SiS₄ | 已知总反应 | Table 3 |

### Fe₂SiS₄ 验证结果（Table 3）

关键配置：

```python
TARGET_FORMULA = "Fe2SiS4"
NET_REACTION_KNOWN = True
T_SYNTHESIS_C = 627                    # 单位：°C（≈900 K）
E_ABOVE_HULL_CUTOFF = 0.5
COMPOSITION_WINDOW = False
FILTER_HYPOTHETICAL = False
MANUAL_PRECURSORS = ["Fe5Si3", "Fe3Si", "S"]
KNOWN_BYPRODUCTS = []
K_SHORTEST = 75
MAX_LINCOMB_COMBO = 6
MAX_INTERMEDIATE_RXNS = 15
```

结果：

| 项 | 论文 Table 3 | 本工作 |
|----|------|--------|
| 最优配平成本 | 0.177 | 0.1694 |
| 配平路径数 | 若干 | 4095 |
| 总反应 | `0.25Fe₅Si₃+0.25Fe₃Si+4S→Fe₂SiS₄` | 一致 |
| 中间相 | SiS₂/FeS/FeSi/FeS₂/S | 均在相空间/路径中出现 |

> 成本与论文同量级（约 -4%）；最优路径排序偏好不同（本工作 Top-1 用 Fe₄S₅ 路线），差异来自 MP 数据版本与 Gibbs 能量细节。

### MgMo₃(PO₄)₃O 验证结果（复现论文 Eq. 7）

#### 方案1 配置

```python
TARGET_SYSTEM = ["Mg", "Mo", "P", "O"]
PRECURSOR_ELEMENTS = ["Na", "S"]       # 论文路径额外元素（Eq.7 需 Na/S）
MACE_ENABLED = False
AUTO_EXTEND_FROM_MP_COVERAGE = False   # 关闭自动扩阳离子，保持体系可控
ENABLE_QUATERNARY = False
HULL_THRESHOLD = 0.1
```

#### 方案2 配置

```python
# 化学体系自动从方案1 scheme1_export.json 同步，无需配置
temperatures_C = [25, 100, 200, 300, 400, 500, 527, 600, 700]   # 含 800 K = 526.85 °C
ATMOSPHERES = {"惰性气氛": {"pO2": 1e-6, "pH2O": 0.0, "pCO2": 0.0}}
```

#### 方案3 配置

```python
TARGET_FORMULA = "MgMo3P3O13"
NET_REACTION_KNOWN = True              # 已知模式验证论文 Eq.7
T_SYNTHESIS_C = 527                    # 单位：°C（≈800 K，四舍五入整数）
E_ABOVE_HULL_CUTOFF = 0.11
E_ABOVE_HULL_CUTOFF_UNKNOWN = 0.11
COMPOSITION_WINDOW = False             # 必须关闭，否则会裁剪掉 Na/S 相
FILTER_HYPOTHETICAL = False            # 必须关闭，否则 163 相会被删到 27 相
USE_MANUAL_PRECURSORS = True
MANUAL_PRECURSORS = ["NaMo3P3O13", "MgS2"]
KNOWN_BYPRODUCTS = ["NaS2"]
USE_OPEN_ELEMENT = False
MAX_REACTANTS = 2
K_SHORTEST = 20
ENABLE_CROSSOVER = True
```

KSP 结果（第 1 条路径）：

```
MgS2 + NaMo3P3O13 -> MgMo3P3O13 + NaS2    dG = +0.028 eV/atom
```

与论文 Eq. 7 对比：

| 项 | 论文 | 本工作 |
|----|------|--------|
| 反应 | `NaMo₃(PO₄)₃O + MgS₂ → MgMo₃(PO₄)₃O + NaS₂` | 完全一致 |
| ΔG | +0.044 eV/atom | +0.028 eV/atom |
| 排序 | 代表性 metathesis 路线 | KSP 最优路径 |

> 说明：已知模式只从指定前驱体出发；若验证论文 Eq. 8（`Mo(PO₃)₃ + Mg(MoO₂)₂ → target`），将方案3 的 `MANUAL_PRECURSORS` 改为 `["Mo(PO3)3", "Mg(MoO2)2"]`、`KNOWN_BYPRODUCTS = []`；若验证 Eq. 9（含 N₂），方案1 的 `PRECURSOR_ELEMENTS` 需加 `"N"`，方案3 的 `MANUAL_PRECURSORS = ["MgMoN2", "Mo2P3O13"]`、`KNOWN_BYPRODUCTS = ["N2"]`。

注意事项：

- 论文使用 MP 2020.09.08 数据；当前 MP 数据版本差异会导致某些相（如 YClO）稳定性不同；
- 成本数值可能与论文有偏差（数据/Gibbs 能量差异），但路径结构和中间相应一致；
- 定向验证时需在扩展体系加入路径所需额外元素（如 MgMo₃ 路线需 Na/S）。

---

## 10. 探索模式：与论文的差异与限制

### 10.1 与论文探索模式的差异

| 维度 | 论文（MgMo₃(PO₄)₃O） | 本工作探索模式 |
|------|---------------------|----------------|
| 化学空间 | 43 种元素全空间（Ag…Zr） | 方案1 扩展体系（目标 + 前驱体元素，如 Mg-Mo-P-O-Na-S） |
| 相空间 | 21,564 entries / 660,909 节点 | 方案1 导出 + Gibbs 过滤后的相 |
| 温度 | 800 K | `T_SYNTHESIS_C` 可配置（°C） |
| 截断 | 0.11 eV/atom | `E_ABOVE_HULL_CUTOFF_UNKNOWN` 可配置 |
| 反应枚举 | n=2 组合 | n=2 组合（`MAX_REACTANTS`） |
| 副产物过滤 | metathesis：副产物不含目标元素 | 相同（metathesis 式过滤） |
| 输出 | 候选反应列表（2270 → 186 条） | 候选反应列表（按成本排序） |

### 10.2 探索模式的限制

1. **化学空间受限（非全局最优）**：只搜索方案1 扩展体系内的相。若路径需要体系外元素（如 MgMo₃ 的 Na/S/N），必须先把这些元素加入方案1 的 `PRECURSOR_ELEMENTS` 并重跑方案1，否则无法发现该路径；因此在缩小后的空间里找到的路径，不等同于论文 43 元素全空间的最优路线；
2. **截断选择敏感**：`E_ABOVE_HULL_CUTOFF_UNKNOWN` 太低会排除目标/亚稳中间相，太高会导致组合爆炸；
3. **探索模式只输出直接反应候选**（与论文一致）：输出“能生成目标”的直接反应列表，不组装多步路径；多步路径组装属于已知模式的 KSP/线性组合功能；
4. **metathesis 过滤可能漏路线**：副产物含目标元素（如 O）的反应会被过滤掉，可能漏掉一些合理路线（与论文同款启发式）；
5. **数据版本差异**：MP 当前版本与论文 2020.09.08 的相稳定性能不同，导致候选集合和成本有偏差。

### 10.3 使用建议

- 明确目标体系包含哪些元素，尽量一次配全（目标 + 可能的前驱体/副产物元素）；
- 先跑通小体系（如 Mg-Mo-P-O）验证逻辑，再逐步扩展；
- 结合方案2 的多温度稳定性结果选择 `T_SYNTHESIS_C`；
- 探索模式输出为候选反应，重要路线建议再用已知模式做多步路径验证。

---

## 11. 方法学说明

| 方法 | 说明 |
|------|------|
| MP 能量修正 | `MaterialsProjectDFTMixingScheme` |
| Gibbs 修正 | Bartel SISSO 描述符 + FactSage 元素化学势 + NIST-JANAF 气体/固体数据 |
| 成本函数 | Softplus：`ln(1 + exp((T_scale/T_synth)·Δg_rxn))` |
| 路径搜索 | 官方 `rxn_network.ReactionNetwork` + Yen KSP |
| 多步路径 | 交叉反应 + 线性组合（已知模式） |
| 线性组合实现 | Numba 批量配平 + joblib 并行（官方内核，Ray-free） |
| 探索过滤 | metathesis 式：副产物不含目标元素 |

---

## 12. 常见问题与已知限制

| 问题 | 原因/解决 |
|------|----------|
| MP 数据版本差异（如 YClO e_above_hull 变化） | 论文用 2020.09.08；需接受差异或强制保留相关相 |
| Windows 下 numpy/OpenBLAS 崩溃（0xc06d007f） | Git Bash 子进程问题；在 Jupyter 中运行正常 |
| 前驱体不在网络中（KeyError） | 检查 `FILTER_HYPOTHETICAL`、`COMPOSITION_WINDOW` 是否误删相 |
| 大体系反应枚举/线性组合慢 | 减小 `MAX_REACTANTS`、`MAX_LINCOMB_COMBO`、`MAX_LINCOMB_CANDIDATES`、`MAX_INTERMEDIATE_RXNS` |
| Windows 下官方 PathwaySolver（Ray）崩溃 | 已用 Numba 内核 + joblib 并行替代 Ray |
| 探索模式找不到论文路径 | 扩展体系需包含路径所需额外元素（Na/S/N 等） |

---

## 13. 参考

- McDermott, M. J.; Dwaraknath, S. S.; Persson, K. A. *A Graph-Based Network for Predicting Chemical Reaction Pathways in Solid-State Materials Synthesis*. **Nat. Commun.** 2021, 12, 3097. https://doi.org/10.1038/s41467-021-23339-x
- Bartel, C. J. et al. *Physical descriptor for the Gibbs energy of inorganic crystalline solids and temperature-dependent materials chemistry*. **Nat. Commun.** 2018, 9, 4168. https://doi.org/10.1038/s41467-018-06682-4
- 参考代码：`reaction-network/`（rxn_network）
