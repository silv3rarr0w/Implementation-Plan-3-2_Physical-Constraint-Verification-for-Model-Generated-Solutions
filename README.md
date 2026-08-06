# 无机粉体材料热力学-动力学高通量筛选平台

基于 Materials Project 数据库 + MACE-MP-0 通用力场，对目标化学体系进行**凸包热力学初筛** → **温度/气氛 Gibbs 校验** → **图论反应网络动力学评价**三级递进分析。

方案三采用 McDermott et al. (2021) 的图论方法：用 Softplus 热力学成本函数替代 CI-NEB 势垒计算，大幅降低计算开销。

---

## 目录结构

```
粉体项目调研/
├── api/
│   └── myapi.env                              # API Key 配置
├── 方案1/
│   ├── 凸包计算_v2.ipynb                       # ★ 主 notebook
│   ├── scheme1_export.json                    # 方案1→方案2 中间数据
│   ├── output.md                              # 形成能 + e_above_hull 报告
│   └── 2023-12-03-mace-128-L1_epoch-199.model # MACE-MP-0 模型文件
├── 方案2/
│   ├── 温度气氛校验_v2.ipynb                   # ★ 主 notebook
│   ├── scheme2_gibbs_export.json              # 方案2→方案3 中间数据
│   └── temperature_analysis.md                # 温度-气氛稳定性报告
├── 方案3/
│   ├── 动力学计算_v2.ipynb                     # ★ 主 notebook（图论方法）
│   ├── kinetics_network_v2_report.md          # Gibbs 修正 + KSP 报告
│   └── kinetics_network_v2_report_0k.md       # 0K DFT 参照报告
├── legacy/
│   ├── 方案1旧/                                 # 方案1 v1 (Legacy)
│   ├── 方案2旧/                                 # 方案2 v1 (Legacy)
│   └── 方案3旧/                                 # 方案3 v1 CI-NEB (Legacy)
├── ref/                                        # 参考文献 PDF
├── 方案要求/                                    # 课题需求文档
├── requirements.txt
└── README.md
```

---

## 快速开始

### 1. 环境准备

```bash
pip install -r requirements.txt

# GPU 加速（可选）
pip uninstall torch -y
pip install torch --index-url https://download.pytorch.org/whl/cu124
```

### 2. 配置 API Key

在目录 `api/` 下创建 `myapi.env`：

```
MP_API_KEY=你的Materials_Project_API密钥
```

> API Key 可在 [Materials Project](https://next-gen.materialsproject.org/api) 注册获取。

### 3. 运行顺序

三个 notebook 按流水线顺序运行，**前一个的输出是后一个的输入**：

```
方案1 凸包计算_v2.ipynb   →   方案2 温度气氛校验_v2.ipynb   →   方案3 动力学计算_v2.ipynb
        ↓                              ↓                              ↓
  scheme1_export.json          scheme2_gibbs_export.json        (优先加载方案2 Gibbs 数据；
  (0K 凸包 + MACE 补全)        (SISSO Gibbs 温度修正)            回退加载方案1 0K 数据)
```

#### 方案1：凸包热力学初筛

| 步骤 | 说明 |
|------|------|
| 修改 `system` | 目标化学体系（示例 `["Y", "Mn", "O"]`） |
| 修改 `HULL_THRESHOLD` | 近稳阈值（eV/atom） |
| `DEBUG_MODE = True` | 调试模式：仅计算少量缺失结构，快速验证 |
| 依次运行全部 Cell | MP 数据获取 → MACE-MP-0 补全 → 凸包构建 → 导出 |
| 输出 | `output.md` + `scheme1_export.json` |

#### 方案2：烧结温度 + 气氛热力学校验

| 步骤 | 说明 |
|------|------|
| 修改 `system`、`temperatures_C`、`ATMOSPHERES` | 化学体系 + 烧结温度范围 + 气氛条件 |
| 依次运行全部 Cell | 加载方案1数据 → SISSO Gibbs 修正 → 气氛依赖凸包 → 分解路径 |
| 新增步骤 11 | 导出 `scheme2_gibbs_export.json` 供方案3_v2 优先使用 |
| 输出 | `temperature_analysis.md` + `scheme2_gibbs_export.json` |

#### 方案3_v2：图论反应网络动力学评价

| 步骤 | 说明 |
|------|------|
| 修改 `TARGET_FORMULA`、`system` | 目标化合物（示例 `"YMnO3"`） |
| 修改 `T_SYNTHESIS` | 合成温度（K），影响 Softplus 成本函数 |
| 修改 `E_ABOVE_HULL_CUTOFF` | 中间相热力学过滤阈值 |
| `COMPOSITION_WINDOW = False` | 跨体系验证时关闭 |
| 依次运行全部 Cell | 数据加载 → 反应枚举 → Softplus 成本 → 图构建 → KSP → 交叉反应 → 报告 |
| 输出 | `kinetics_network_v2_report.md` + `kinetics_network_v2_report_0k.md` |

> **数据源优先级**：方案3_v2 优先加载方案2的 Gibbs 温度修正数据；若不可用则回退方案1的 0K 数据；若均不可用则直接从 MP API 获取。

---

## 方法概要

| 方案 | 方法 | 核心工具 | 参考文献 |
|------|------|----------|----------|
| 方案1 | 0K 凸包 + MACE-MP-0 力场补全 | pymatgen PhaseDiagram + MACE-MP-0 | — |
| 方案2 | SISSO 高温 Gibbs 修正 + 气氛化学势 | Bartel et al. (2018) | [*Nat. Commun.* 2018](https://doi.org/10.1038/s41467-018-06682-4) |
| 方案3 | 图论反应网络 + Softplus 热力学成本 | McDermott et al. (2021) | [*Nat. Commun.* 2021](https://doi.org/10.1038/s41467-021-23339-x) |

---

## 配置说明

| 文件 | 配置项 | 说明 |
|------|--------|------|
| `api/myapi.env` | `MP_API_KEY` | Materials Project API 密钥 |
| 方案1 Cell 3 | `system`, `HULL_THRESHOLD`, `DEBUG_MODE` | 化学体系 + 近稳阈值 + 调试开关 |
| 方案2 Cell 3 | `system`, `temperatures_C`, `ATMOSPHERES` | 化学体系 + 温度范围 + 气氛条件 |
| 方案3_v2 Cell 3 | `TARGET_FORMULA`, `system`, `T_SYNTHESIS`, `E_ABOVE_HULL_CUTOFF` | 目标化合物 + 合成温度 + 过滤阈值 |

---

## 故障排查

| 问题 | 原因 | 解决 |
|------|------|------|
| MACE-MP-0 运行慢 | CPU 模式 | 安装 CUDA 版 torch |
| `MP_API_KEY missing` | API Key 未配置 | 检查 `api/myapi.env` |
| 方案3_v2 与 v1 报告不一致 | v1 用 CI-NEB 动力学势垒，v2 用 Softplus 热力学成本 | v2 为推荐方案，v1 保留供对比 |
| 获取 MP 数据报错（SSLError） | 网络不稳定 | 重新运行该 Cell |
| 方案3_v2 数据加载失败 | 上游未导出 | 确保方案1/方案2 已运行完毕 |

---

## Legacy（v1 版本）

> 方案3 的 v1 版本采用**粗→细两阶段 CI-NEB + KSP 图搜索**，基于显式 NEB 势垒计算。适合已有候选路径的精细动力学分析，但计算成本较高（完整运行约需 1 小时）。v1 notebook 保留在 `legacy/方案3旧/动力学计算.ipynb`。

**方案3 v1 工作流：**

```
MP数据 → e_above_hull 过滤 → 反应枚举 → 全边粗筛 CI-NEB → 能垒建图
  ↓
KSP → Top-3 路径边精修 CI-NEB → 最终合成/分解路径 → 报告
```

| 配置项 | 说明 |
|--------|------|
| `TARGET_FORMULA` | 目标化合物 |
| `VALIDATION_MODE = True` | 验证模式（仅跑少量边） |
| `COARSE_N_IMAGES` | 粗筛帧数（默认 5） |
| `FINE_N_IMAGES` | 精修帧数（默认 7） |
