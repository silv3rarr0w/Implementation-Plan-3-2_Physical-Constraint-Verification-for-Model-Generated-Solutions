# 无机粉体材料热力学-动力学高通量筛选平台

Materials Project + MACE-MP-0：**凸包热力学初筛 → 温度/气氛 Gibbs 校验 → 反应网络动力学**（Softplus 成本替代 CI-NEB，对齐 McDermott 2021）。

## 1. 项目简介

- 输入：目标化合物（如 `MgMo3(PO4)3O`、`Fe2SiS4`）
- 输出：0K 凸包（方案1）、温度/气氛稳定性（方案2）、KSP + 线性组合报告（方案3）
- 模式：已知总反应（复现论文路径）/ 探索模式（新化合物路线设计）

## 2. 工作流总览

```
方案1 (0K 凸包 + MACE) → scheme1_export.json
   ├── 方案2 稳定性分析（不向方案3 传数据）
   └── 方案3 由 scheme1 现场构建 Gibbs → KSP / 线性组合
```

- 方案1 是唯一数据源；方案2 不导出 scheme2；方案3 在 `T_SYNTHESIS_C`（°C，内部转 K）下用 Bartel SISSO + FactSage + NIST 构建 Gibbs。

目录：`api/myapi.env`、`方案1/凸包计算_v2.ipynb`（+`scheme1_export.json`）、`方案2/温度气氛校验_v2.ipynb`、`方案3/动力学计算_v2.ipynb`（+`kinetics_network_v2_report.md`）、`ref/`、`reaction-network/`、`requirements.txt`。

## 3. 环境与依赖

```bash
conda create -n DFT python=3.11 && conda activate DFT
pip install -r requirements.txt
# GPU（建议安装）：pip install torch --index-url https://download.pytorch.org/whl/cu124
```

依赖：pymatgen/mp-api、mace-torch/ase、rxn_network、networkx/scipy/matplotlib。API Key 放 `api/myapi.env`：`MP_API_KEY=...`。

## 4. 快速开始

```
方案1 → 方案2 → 方案3
```

方案2/3 直接读方案1 的 `scheme1_export.json`；配置示例见 §9。

## 5. 方案1：凸包计算

| 配置 | 说明 |
|---|---|
| `TARGET_SYSTEM` / `PRECURSOR_ELEMENTS` | 目标 / 扩展体系 |
| `MACE_ENABLED` | MACE-MP-0 缺失结构预测 |
| `HULL_THRESHOLD` | 近稳定阈值 (eV/atom) |
| `AUTO_EXTEND_FROM_MP_COVERAGE` | 按 MP 覆盖度自动扩展 |

输出：`scheme1_export.json` + `output.md`。

## 6. 方案2：温度气氛校验

| 配置 | 说明 |
|---|---|
| 化学体系 | 自动同步方案1 |
| `temperatures_C` / `ATMOSPHERES` | 温度 (°C) / 气氛 |
| `E_ABOVE_HULL_DECOMP_THRESH` | 分解判定阈值 |

输出：`temperature_analysis.md`；不参与方案3 数据传递。

## 7. 方案3：动力学计算

| 配置 | 说明 |
|---|---|
| `TARGET_FORMULA` / `NET_REACTION_KNOWN` | 目标 / 已知或探索 |
| `T_SYNTHESIS_C` | 合成温度 (°C，内部转 K) |
| `E_ABOVE_HULL_CUTOFF` / `_UNKNOWN` | 凸包截断 |
| `MANUAL_PRECURSORS` / `KNOWN_BYPRODUCTS` | 已知模式 |
| `MAX_REACTANTS` / `K_SHORTEST` | 网络 / KSP |
| `MAX_LINCOMB_COMBO` / `MAX_LINCOMB_CANDIDATES` / `MAX_INTERMEDIATE_RXNS` | 线性组合参数 |
| `USE_OPEN_ELEMENT` | 开放元素（处理气体时打开） |

流程：加载方案1 → Gibbs 现场构建 → 凸包过滤 → 反应枚举 → 官方 `ReactionNetwork` + Yen KSP → 交叉反应 + 线性组合（Numba + joblib，Ray-free）→ 报告（含 Balanced Pathways）。

## 8. 已知 vs 探索模式

| 维度 | 已知总反应 | 探索模式 |
|---|---|---|
| 前驱体 / 副产物 | 手动指定 | 不指定 / 自动 |
| 搜索范围 | 前驱体 → 目标 | 所有生成目标的反应 |
| 用途 | 验证配方 | 新化合物路线 |
| 典型体系 | Fe₂SiS₄ | MgMo₃(PO₄)₃O\* |
| 开关 | `NET_REACTION_KNOWN=True` | `False` |

探索额外过滤：副产物不含目标元素（metathesis）。
\* MgMo₃ 是论文探索案例，本工作 §9 用已知模式复现其路径（Eq. 7）。

## 9. 验证案例

| 体系 | 模式 | 论文对照 |
|---|---|---|
| MgMo₃(PO₄)₃O | 已知模式复现探索路径 | Eq. 7/8/9 |
| Fe₂SiS₄ | 已知总反应 | Table 3 |

### MgMo₃(PO₄)₃O

```python
# 方案1
TARGET_SYSTEM=["Mg","Mo","P","O"]; PRECURSOR_ELEMENTS=["Na","S"]
MACE_ENABLED=False; AUTO_EXTEND_FROM_MP_COVERAGE=False; ENABLE_QUATERNARY=False; HULL_THRESHOLD=0.1
# 方案2
temperatures_C=[25,100,200,300,400,500,600,700]
ATMOSPHERES={"惰性气氛":{"pO2":1e-6,"pH2O":0,"pCO2":0}}
# 方案3
TARGET_FORMULA="MgMo3P3O13"; NET_REACTION_KNOWN=True; T_SYNTHESIS_C=527
E_ABOVE_HULL_CUTOFF=0.11; COMPOSITION_WINDOW=False; FILTER_HYPOTHETICAL=False
MANUAL_PRECURSORS=["NaMo3P3O13","MgS2"]; KNOWN_BYPRODUCTS=["NaS2"]
K_SHORTEST=20; ENABLE_CROSSOVER=True
```

结果：KSP 第 1 条 = `MgS2 + NaMo3P3O13 → MgMo3P3O13 + NaS2`，ΔG +0.028 vs 论文 +0.044 eV/atom，反应完全一致。
Eq. 8：`MANUAL_PRECURSORS=["Mo(PO3)3","Mg(MoO2)2"]`,`KNOWN_BYPRODUCTS=[]`；Eq. 9：方案1 加 `"N"`，`MANUAL_PRECURSORS=["MgMoN2","Mo2P3O13"]`,`KNOWN_BYPRODUCTS=["N2"]`。

### Fe₂SiS₄

```python
# 方案1
TARGET_SYSTEM=["Fe","Si","S"]; PRECURSOR_ELEMENTS=[]; MACE_ENABLED=False; HULL_THRESHOLD=0.5
# 方案2
temperatures_C=[25,100,200,300,400,500,700]
# 方案3
TARGET_FORMULA="Fe2SiS4"; NET_REACTION_KNOWN=True; T_SYNTHESIS_C=627
E_ABOVE_HULL_CUTOFF=0.5; COMPOSITION_WINDOW=False; FILTER_HYPOTHETICAL=False
MANUAL_PRECURSORS=["Fe5Si3","Fe3Si","S"]; KNOWN_BYPRODUCTS=[]
K_SHORTEST=75; MAX_LINCOMB_COMBO=6; MAX_INTERMEDIATE_RXNS=15
```

结果：最优配平成本 0.1694 vs 论文 0.177（约 -4%）；4095 条配平路径；总反应一致；中间相 SiS₂/FeS/FeSi/FeS₂/S 均出现。Top-1 用 Fe₄S₅ 路线（论文偏好 SiS₂/FeS），差异来自 MP 数据版本与 Gibbs 能量。

## 10. 探索模式：与论文的差异与限制

| 维度 | 论文（MgMo₃） | 本工作探索模式 |
|---|---|---|
| 化学空间 | 43 元素全空间 | 方案1 扩展体系 |
| 相空间 | 21,564 entries / 660,909 节点 | 方案1 导出 + Gibbs 过滤 |
| 温度 / 截断 | 800 K / 0.11 | `T_SYNTHESIS_C` / `E_ABOVE_HULL_CUTOFF_UNKNOWN` |
| 反应枚举 | n=2 | n=2 |
| 副产物过滤 | metathesis | metathesis |
| 输出 | 2270 → 186 条 | 按成本排序候选 |

限制：
1. 化学空间受限 → 非全局最优（需在方案1 加元素并重跑）
2. 截断选择敏感
3. 只输出直接反应候选（与论文一致；多步路径属已知模式）
4. metathesis 可能漏路线
5. MP 数据版本差异

建议：元素一次配全、先小体系、用方案2 选温度、重要路线用已知模式验证。

## 11. 方法学说明

| 方法 | 说明 |
|---|---|
| MP 能量修正 | `MaterialsProjectDFTMixingScheme` |
| Gibbs 修正 | Bartel SISSO + FactSage + NIST |
| 成本函数 | Softplus：`ln(1 + (273/T)·exp(Δg))` |
| 路径搜索 | 官方 `rxn_network.ReactionNetwork` + Yen KSP |
| 线性组合 | Numba 批量配平 + joblib 并行（Ray-free） |
| 探索过滤 | metathesis 式 |

## 12. 常见问题

| 问题 | 解决 |
|---|---|
| MP 数据版本差异（YClO 等） | 接受差异或强制保留相关相 |
| Windows numpy/OpenBLAS 崩溃 | 在 Jupyter 中运行 |
| 前驱体不在网络中（KeyError） | 检查 `FILTER_HYPOTHETICAL` / `COMPOSITION_WINDOW` |
| 线性组合慢 | 调小 `MAX_REACTANTS` / `MAX_LINCOMB_COMBO` / `MAX_LINCOMB_CANDIDATES` / `MAX_INTERMEDIATE_RXNS` |
| 官方 PathwaySolver（Ray）崩溃 | 已用 Numba + joblib 替代 |
| 探索模式找不到路径 | 扩展体系加入所需元素 |

## 13. 参考

- McDermott et al., *Nat. Commun.* 2021, 12, 3097. https://doi.org/10.1038/s41467-021-23339-x
- Bartel et al., *Nat. Commun.* 2018, 9, 4168. https://doi.org/10.1038/s41467-018-06682-4
- 参考代码：`reaction-network/`（rxn_network）
