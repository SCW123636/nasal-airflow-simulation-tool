---
team: 剑穹小队
solution: 鼻腔气流重构自动化流水线
status: 原型开发中
summary: 匿名化鼻腔 CT 自动重建气道三维模型、自动定位鼻孔与出口、生成四面体网格并跑 OpenFOAM 层流求解，三例患者全链跑通，输出鼻阻力、压差与壁面剪切应力
repo: https://github.com/SCW123636/nasal-airflow-simulation-tool
---

# 鼻腔气流重构自动化流水线

输入一例匿名化的鼻腔 CT，输出可直接进 CFD 的气道模型与气流结果。目标是把需求书里「分割 10 分钟 + 修复 30–60 分钟 + 求解 1–4 小时」的人工环节换成一条可批量、同一标注标准的自动链路。

当前状态：**三例患者（b15 / b01 / b02）全链跑通**，含几何、网格、求解、流线与三维检视；几何层在 24 例公开批次上 auto_pass 22/24（opt-in 臂）。三例之外的求解未做，不作任何泛化声明。

## 一、自动化到哪一步

| 环节 | 人工基线 | 本流水线 | 是否自动 |
|---|---|---|---|
| DICOM 序列选择 | 手工挑序列 | `batch27_series_choice.py` 按 `ImageType` + 层厚选 | 自动 |
| 气道分割 | 3D Slicer 手工描画 | TotalSegmentator `head_glands_cavities` 四标签 + HU 阈值 | 自动 |
| 解剖定位 | 人工找鼻孔/鼻咽平面 | `auto_anatomy_landmarks.py` 提案三点位 | 自动 |
| 表面重建 | Blender 修破洞毛刺 | marching cubes + 并集 ∩ HU<−300，闭合度检查 | 自动 |
| 开孔与裁剪 | 手工切平面 | `auto_opening_pipeline.py` 按端盖面积二分 | 自动 |
| 体网格 | 手工设参数 | netgen，参数随域给定 | 半自动（脚本给定参数，需在 Linux 侧跑） |
| 求解 | 手工配边界条件 | `run_openfoam_case.py` 生成 case 并驱动 simpleFoam | 半自动（WSL2 内执行） |
| 可视化 | ParaView 手工出图 | 浏览器三维检视页（模型 + 流线同视口） | 自动 |

「半自动」的含义是：命令可一键执行，但需要用户自备 OpenFOAM v1912 与 WSL2 环境，且网格必须在 native Linux 文件系统上跑（`/mnt/c` 上跑会慢一个量级）。

## 二、实测结果（三例）

总流量 15 L/min（双侧各 7.5），出口表压 0 Pa，层流。数值来自各例求解记录，未做任何四舍五入美化。

| 指标 | b15 | b01 | b02 |
|---|---|---|---|
| 压差 Δp (Pa) | 17.120 | 15.460 | 43.231 |
| 鼻阻力 R (Pa·s/mL) | 0.06848 | 0.06184 | 0.17292 |
| 出口面积 (mm²) | 125.85 | 208.46 | 84.57 |
| 气道容积 (mL) | 21.33 | 27.53 | 21.13 |
| WSS 中位 (Pa) | 0.0678 | 0.0503 | 0.0897 |
| WSS 最大 (Pa) | 3.488 | 2.346 | 22.865 |
| WSS 积分力 (N) | 0.001485 | 0.001274 | 0.002370 |
| 网格单元 | 653,789 | 710,101 | 590,961 |
| 求解收敛 | 四判据全过 | 四判据全过 | 四判据全过 |

收敛判据：连续性不平衡 < 1e-6、监测量波动 < 0.5%、残差 1e-5、600 步正常 End。三例实测不平衡 6.0e-8 / 4.5e-7 / 4.6e-7，监测量 0.011% / 0.010% / 0.076%。

完整数值与逐条出处见 [`results/metrics.json`](results/metrics.json)（原样搬运自求解记录，未手抄）。

> ⚠️ R 的绝对值不可与临床鼻测压对标。层流模型系统性低估阻力，且我们的域在口咽底截断、缺下咽与喉部阻力。**只有同一管线下的相对变化可比**。三例 R 均低于 ISCOANA 0.25 Pa·s/mL 参考，方向已知。

## 三、目录结构

```
solutions/剑穹小队-鼻腔气流重构/
├── src/pipeline/   几何链路：DICOM → STL（27 个模块，含依赖闭包）
├── src/cfd/        netgen → OpenFOAM：case 生成、求解驱动、8 份配置
├── src/viewer/     浏览器三维检视：模型与流线同视口、双例对比
├── docs/           方法、偏差说明、结果口径
├── results/        三例指标（原样搬运，未手抄）
├── requirements.txt
└── LICENSE
```

## 四、快速开始

### 环境

- Windows + Python 3.10（`pydicom` / `SimpleITK` / `numpy` / `scipy` / `vtk`）
- TotalSegmentator（`pip install TotalSegmentator`，首次运行会下载 `head_glands_cavities` 权重）
- netgen（体网格）
- WSL2 Ubuntu 22.04 + OpenFOAM v1912（原生部署，非 Docker）

依赖版本见 [`requirements.txt`](requirements.txt)。

### 输入布局

```
<_work>/<case>/ct.nii.gz
<_work>/<case>/masks/nasal_cavity_left.nii.gz
<_work>/<case>/masks/nasal_cavity_right.nii.gz
<_work>/<case>/masks/nasopharynx.nii.gz
<_work>/<case>/masks/oropharynx.nii.gz
```

`<_work>` 默认为本方案目录下的 `_work/`，可用环境变量 `NASAL_TS_BASE` 指向别处。所有输出落在同一目录，不写回原始影像目录。

### 跑几何

```bash
# 全链自动（含序列选择 → 分割 → 定位 → 表面 → 开孔）
python src/pipeline/nasal_pipeline.py --cases <case> --jobs 2 --threads 6

# 已有分割结果时跳过前两步
python src/pipeline/nasal_pipeline.py --cases <case> --skip-series --from-stage 3
```

24 核机上实测 `--jobs 2 --threads 6` 峰值 42%、均值 36% CPU。分割约 136 s/例（CPU）。

每一例产出 `auto_pass` 或 `escalated`。**escalated 不是"再试一次"的信号**，是这一例超出了本方法的设计范围（见 `docs/方法.md` §3），应转人工而不是调参绕过。

### 跑 CFD

```bash
# 在 WSL2 内，指向已生成的 auto_placed.vtp
python src/cfd/run_openfoam_case.py --surface <auto_placed.vtp> --config src/cfd/configs/total_flow_laminar_medium.json
```

### 看三维

```bash
cd src/viewer && python -m http.server 8769
```

浏览器打开 **`http://127.0.0.1:8769/t058_cut.html?case=b15`** —— 模型与流线在同一 3D 视口（页面左下角可切 b01 / b02；切到没有流线的病例时会自动只显示模型，不冒充）。

| 入口 | 是什么 |
|---|---|
| `t058_cut.html?case=<b15\|b01\|b02>` | 三维检查（几何 + 开口 + 切除楔形 + 可选流线层），带 `&sl=<lines.json>` 叠加该例流线 |
| `t058_cmp.html?a=b15&b=b02` | 双例并排，两侧共用相机轨道，可各自带 `&sa=` / `&sb=` 流线层 |
| `pro/index.html` | 工程工作台：患者树、功能指标、三维检查同框、对比 |

- **`src/viewer` 就是站点根**：所有页面用根绝对路径（`/vendor/...`、`/pro/...`），因此直接 `http.server` 起在本目录即可，不需要额外的反向代理或 `/workspace` 路由。若部署到子路径下，需同步改这些根路径。
- 渲染栈已随仓库附带：`src/viewer/vendor/` 为 three.js **0.180.0**（`source.json` 记录 tarball 与各文件 sha256，MIT，见 `THREE_LICENSE.txt`），**看页面不需要联网**。
- 几何数据 `t058_cut_data.json` + `t058_b<case>.json`（每例约 1.5 MB，抽稀显示网格），由 `outputs/T-058/build_cut_assets.py` 生成，带来源 sha256。
- `src/viewer/pro/artifacts/` 内嵌三例已核验的流线数据（base64 顶点 + 速度），**无需重跑求解**即可查看；`.lines.json` 是给上面两个页面叠加用的坐标数据（LPS / mm）。
- `src/viewer/index.html` 是本地七步处理控制台，依赖工作台后端路由（`/viewer/*`、`/api/*`），**静态部署下不可用**，与本目录里三个检视页无关。

## 五、与需求书的偏差

**必须提前说明**：本方案有一处与产品需求书口径不同的地方——出口位置。需求书写「鼻咽出口沿冠状面切平整、统一为软腭游离缘水平」，我们采用的是**垂直局部气道轴的解剖斜切**，并把域延伸到可见口咽、在扫描 FOV 底面做工程截断。这个改动是有意的，理由与影响见 [`docs/与需求书的偏差.md`](docs/与需求书的偏差.md)。其余口径（入口 15 L/min、出口 0 Pa、鼻阻力 = Δp/Q）与需求书一致。

## 六、已知局限

以下都是已知且有方向的，不是待发现的：

1. **层流**。需求书推荐定量用 k-ω SST，我们在当前网格上没跑通（退化网格 + 高非正交），因此全部结果来自层流。雷诺数约 1000–2500 处于层流到过渡区间，层流会低估压降与 WSS。
2. **R 绝对值不可对标临床**。原因见上；配对相对变化是唯一站得住的口径。
3. **固定 50:50 分流**。三例按双鼻孔各 7.5 L/min 给定。自然分流（按鼻腔阻力自动分配）口径尚未实现，它会让左右流量分配更接近生理，但需要重新适配边界条件后重跑。
4. **域在口咽底截断**。截断面不是解剖结构，面积由扫描 FOV 与气道轴倾角共同决定，因此**该面积不能与 choana 文献带（297–362 mm²）比较**。
5. **温湿度/热通量未出患者级结果**。合成验证已通过，但取值截面须在后鼻孔而非域出口，患者级仍待重跑。
6. **正常参考队列未建立**。所以没有「患者 vs 正常人」的分组对照，只有单例绝对量与例间对比。
7. **只跑到三例**。几何层在 24 例上验过，求解层只有 b15 / b01 / b02。不可据此声明批量临床可用。
8. **k-ω / 湍流、批量调度、云端并发**均未实现。

## 七、数据与隐私

- 仓库内**不含任何 DICOM、NIfTI、患者影像**。`.gitignore` 逐项拦截。
- 三例结果中的 `b01 / b02 / b15` 是项目内部匿名编号，不含姓名、病历号、检查日期。
- 三维资产是**分割后的气道几何**（纯几何，无面部软组织渲染），不含可识别影像。
- 所有产物携带 `clinical_status: generalisation_test_only_no_human_review`。

## 八、许可

MIT，见 [LICENSE](LICENSE)。医疗免责声明一并写在该文件末尾。
