---
name: bulk-rna-seq
display_name: Bulk RNA-seq 基础分析
display_name_en: Bulk RNA-seq Basic Analysis
description: bulk RNA-seq 下游分析：拿到表达矩阵后，完成数据类型判定、分析前评估（PCA、相关性热图、层次聚类）、差异分析（DESeq2/edgeR/limma）、GO 与 KEGG 的 ORA/GSEA/GSVA 富集，并按统一口径产出图表。扩展可选分析（默认选做）包括免疫浸润与微环境（CIBERSORT、ESTIMATE、ssGSEA、xCell、MCP-counter、TIMER、TMEscore）、乳腺癌 PAM50 分型、EMT 评分，以及药敏预测（CellMiner、pRRophetic、oncoPredict）。bulk RNA-seq 上游分析：FASTQ 质控、比对与定量（fastp、hisat2/STAR、featureCounts），自动生成，慎重选择。关键词：bulk RNA-seq、转录组、差异表达、富集分析、GSEA、GSVA、免疫浸润、CIBERSORT、ESTIMATE、PAM50、EMT、药敏预测、oncoPredict。
description_zh: 从表达矩阵出发，完成 bulk RNA-seq 的数据类型判定、分析前评估、差异分析与 GO/KEGG 富集，并输出可复现图表；免疫浸润、PAM50、EMT 与药敏预测为可选扩展。上游 FASTQ 分析可用，但测试尚不充分。
description_en: "Downstream bulk RNA-seq analysis from an expression matrix: data-type check, pre-analysis assessment (PCA, correlation heatmaps, hierarchical clustering), differential expression (DESeq2/edgeR/limma), and GO/KEGG enrichment (ORA/GSEA/GSVA), with reproducible figures. Optional add-ons: immune infiltration (CIBERSORT, ESTIMATE, ssGSEA, xCell, MCP-counter, TIMER, TMEscore), PAM50 subtyping, EMT scoring and drug-sensitivity prediction (CellMiner, pRRophetic, oncoPredict). Upstream FASTQ QC, alignment and quantification (fastp, hisat2/STAR, featureCounts) are available but less tested."
version: 1.0.0
author: 一条有梦想的咸鱼
examples_zh:
  - "用一份 Count 矩阵做处理组与对照组的差异分析，并预测药敏"
  - "对已有表达矩阵做分析前评估三图与 GSEA"
examples_en:
  - "Run differential expression between treatment and control from a count matrix, then predict drug sensitivity"
  - "Do pre-analysis assessment and GSEA on an existing expression matrix"
---

# bulk-rna-seq 本地全流程分析

本技能指导从原始测序数据（FASTQ）到差异表达与功能富集的完整 bulk RNA-seq 分析。
内容提炼自「被炸熟的虾」公众号贴文，
覆盖上游（Linux 命令行）与下游（R 语言）两段流程。

## 何时使用

- 用户要从 SRA/ENA 下载测序数据并走完整上游流程（质控 → 比对 → 定量）。
- 用户已有表达矩阵（count/TPM/FPKM），需要差异分析、样本评估、功能富集。
- 用户提到 bulk RNA-seq、转录组差异分析、DESeq2、GSEA、GSVA、PCA、热图等。
- 用户要做免疫浸润（CIBERSORT/ESTIMATE/ssGSEA/xCell/TIMER）、分子分型（PAM50）、
  EMT 评分、药敏预测（pRRophetic/oncoPredict）等下游扩展分析。

## 流程总览

上游分析（Linux，逐个样本）：
`下载(FastQC)` → `质控(fastp/TrimGalore)` → `比对(hisat2/STAR)` →
`排序(samtools)` → `定量(featureCounts/RSEM/salmon)` → **count 矩阵**

下游分析（R）：
`数据类型判断` → `定量转换(Count→TPM/FPKM)` → `基因ID转换` → `分析前评估三图` →
`差异分析(DESeq2/edgeR/limma)` → `富集(ORA/GSEA/GSVA/ssGSEA)` →
`扩展分析(免疫浸润/PAM50/EMT/药敏，按需)`

## 分析前的确认（先问，再动手）

开始分析前先问一句：**参考基因组 / 注释版本（或 GTF 文件）是否由你提供**。
**不得默认代用户决定，也不得自行推断**。两种情况的处理如下：

- **用户提供 → 一律以用户提供的为准**。问清物种、build、注释来源与版本号
  （如 GENCODE v38 / Ensembl 104、GENCODE vM25），记录到结果说明，
  并用于取基因长度、ID 转换与全部依赖注释的计算。
- **用户未提供 → 停下来问，不要猜**。把"用哪个 build / 注释来源 / 版本"
  作为明确问题抛给执行人，等答复后再动手；答复前只做与注释无关的准备工作
  （读矩阵、判断数值类型、核对样本名与分组）。
  只有当执行人明确表示"不清楚、由你定"时，才可指定一套，并在结果说明中标注为
  **"执行人授权指定（非数据自带信息）"**，同时列明该选择影响哪些步骤
  （基因长度、ID 转换、FPKM 换算、富集背景）。

**禁止的推断方式（反例，一律不得作为定版依据）：**

- 以"本机存在某个版本的 GTF / 基因长度对象 / 既往项目产物"推断本次数据用的就是该版本；
  例如"`RefGene/` 里有 `Genelength-gencode-v38.Rdata`，说明用户流程跑的是 v38"——**不成立**；
- 以基因 ID 形态、染色体命名（`chr1` vs `1`）、管家基因长度等旁证**直接定版**
  （这些只能用于向执行人提问时提供候选选项，不能替代答复）；
- 把推断结果写成"已确认"、"应该是"、"推测为"后继续往下算。

确需做版本比对（仅限执行人明确要求核对时）时，比对过程与结论一并写入结果说明，
并注明"经执行人确认"或"未经确认，仅记录为假设"，不得含糊表述。

关键原则（务必遵守）：

0. **数据类型判断是差异分析前的强制前置检查**（不可跳过）。拿到表达矩阵后，
   先 `head` 读几行判断数值形态，再据文件名/来源辅助确认：
   - raw count：非负整数、含大量 0、量级跨度大 → 可直接进 DESeq2/edgeR。
   - TPM/FPKM：非负浮点、标准化、量级稳定 → 不能直接进 DESeq2/edgeR。
   - log2 值：有负值或集中在中段（如 -5~15）→ 不能直接进差异分析。
   若用户提供的是 TPM/FPKM/log 值而非 raw count，**必须先告知并阻止直接差异分析**，
   给出处理建议（如回到定量上游取 count、或改用 limma 等适配方法），
   征得用户同意后再继续。
1. 上游定量若用 featureCounts，输出为 raw count，需先做 Count→TPM/FPKM 转换
   （用 GTF 提取基因有效长度）；若用 salmon/RSEM 则可直接拿 TPM，但仍推荐
   经 tximport 导入 DESeq2。
2. 分析前评估是硬性要求：**至少画三张图**（PCA、组间相关性热图、层次聚类）。
   其中**组间相关性热图必须出两版**：全部基因、MAD 最大的 top 2000 基因
   （全基因版相关系数往往整体偏高、组间区分度弱；MAD 版才反映组间结构）。
   两版文件名分别带 `-Cor-AllGenes` / `-Cor-Top2000MAD` 后缀，便于区分。
   **层次聚类热图同样取 MAD top 2000**（与 MAD 版相关性热图共用同一基因集），
   三处口径一致，不得各用一套阈值。
   三张图分不开时，差异分析结果不可信，应先排查离群样本与混杂因素。
3. 基因 ID 转换（ENSEMBL↔SYMBOL↔ENTREZID）在差异分析与富集前完成，
   去重规则：按 SYMBOL 排序后取平均表达最高的一条。
4. 富集分析需要 ENTREZID（clusterProfiler 的 `bitr` 或 `enrichGO/enrichKEGG`
   的 OrgDb 注释），注意物种对应的 OrgDb。
   **ORA 必须显式指定 `universe`（实验检出基因），不能用默认全注释背景**；
   导出全量结果时 `qvalueCutoff = 1`（默认 0.2 会静默截断）。
   **ORA 一律把上调、下调基因分开各做一次独立富集，不做 Up+Down 合并基因集的 ORA**
   （两侧方向相反，合并后信号互相抵消，也无法回答"是哪一侧驱动了该通路"）。
   **不做背景敏感性 / 阈值敏感性 / 留一法三项核查，也不输出相关文件。**

**富集分析的默认范围（务必遵守，默认从简）：**

方法层面：

| 分析 | 默认 | 基因集 |
|---|---|---|
| ORA（`enrichGO` / `enrichKEGG`） | ✅ 默认运行，**上调与下调各做一次** | GO BP/CC/MF + KEGG |
| GSEA（`gseGO` / `gseKEGG` / fgsea） | ✅ 默认运行 | GO + KEGG |
| GSVA / ssGSEA | ❌ 选做 | 用户要求时运行，基因集同样固定为 GO + KEGG |
| 疾病注释（DO/NCG/DGN）、药敏、免疫注释、EMT、PAM50 | ❌ 选做 | 见下 |

ORA 的方向口径（硬性）：**只做两个基因集——Up（上调 DEG）与 Down（下调 DEG），
不做二者合并后的"全部 DEG"富集**。上/下调分别出点图与条形图，
结果表分别命名为 `ORA-Up-*` 与 `ORA-Down-*`。
若需要把两侧放在一张图里对照，只能**把两次独立富集的结果并排展示**，
不得把基因合并成一个列表重新检验。

基因集层面（硬性）：

- **默认只用 GO 与 KEGG**，且 ORA / GSEA / GSVA / ssGSEA 四类方法口径一致：
  不因方法不同而换基因集（例如 GSVA/ssGSEA 不得默认改用 Hallmark 打分）。
- **Hallmark（H / `mh.all`）、Reactome（C2:CP:REACTOME）、其他 MSigDB 集合
  （C2/C5/C6/C7…）默认一律不使用**。它们不属于"选做项"：不主动运行、
  不主动推荐、不在开工确认里作为可选项抛给用户；
  仅当用户明确点名（"加 Hallmark"、"用 Reactome"）时才启用，
  并在结果说明中记为"按要求追加的集合"。
- 开工前用一句话确认范围（"默认做 GO + KEGG 的 ORA 与 GSEA，需要加
  GSVA / ssGSEA 吗？"），用户未明确要求即按默认执行；**确认里不列
  Hallmark / Reactome**。
- 结果说明文档写明"富集分析使用 GO 与 KEGG 基因集"；未运行的选做方法记为
  "未运行（选做项）"，Hallmark / Reactome 则记为"未使用（默认不含该集合）"。
- 脚本中把选做方法写成开关（如 `run_gsva <- FALSE`）；基因集来源集中由
  一个变量控制（如 `gset_src <- c("GO", "KEGG")`），追加集合必须显式手改，
  不要留下"顺手打开"的默认路径。

扩展分析的默认规则（务必遵守）：

- **药敏预测、免疫注释、EMT 评分 → 默认选做**：不主动运行、不主动推荐，
  仅当用户明确要求时才执行。
- **PAM50 分子分型 → 仅针对乳腺癌**：其他癌种不主动提及、不主动运行；
  仅在确认数据为乳腺癌（或用户明确指定）时才使用。

## 详细工作流

完整步骤、命令参数、R 代码与注意事项分模块存放，按需加载：

- `references/upstream-linux.md` — 工具安装、SRA/ENA 数据下载、fastp 质控、
  hisat2/STAR 比对、samtools 排序、featureCounts 定量。
- `references/downstream-prep.md` — Count→TPM/FPKM 定量转换、基因 ID 转换、
  管家基因校验、低表达基因过滤。
- `references/downstream-eval.md` — 分析前评估三图（PCA、相关性热图两版
  ：全基因 + MAD top 2000、层次聚类同样取 MAD top 2000）。
- `references/downstream-deg.md` — DESeq2(tximport)/edgeR/limma 差异分析。
- `references/downstream-enrich.md` — ORA 富集可视化、GSEA、GSVA/ssGSEA、
  疾病注释 DO/NCG/DGN 富集、基因集活性评分，GSEA 逐个通路出图（GseaVis）。
- `references/gsea-visualization.md` — GSEA 深度可视化全解：msigdbr 基因集
  获取（含小鼠 CP:KEGG，解决 gseKEGG 联网失败）、GSEA 四种方法
  （GSEA/gseKEGG/gseGO/fgsea）、GseaVis 全系列、enrichplot 全系列、
  自定义 GMT 制作、Wwp1 KO 实战案例。
- `references/output-conventions.md` — 输出命名与可重复性硬性约定：
  设种子 + 保存中间结果、结果说明文档（含环境配置）、在线数据源失败降级、
  优先 .xlsx + 同步 .txt、**单一交付目录 `Produced by Bioshrimp/` 与唯一说明文档**。
- `references/downstream-immune.md` — 免疫浸润与微环境（CIBERSORT、ESTIMATE、
  ssGSEA、xCell、MCP-counter、TIMER、TMEscore、IOBR）、乳腺癌 PAM50 分子分型、
  EMT 评分、基因功能通路批量注释（MSigDB）。
- `references/downstream-drug.md` — 药敏预测（CellMiner 相关性、pRRophetic、
  oncoPredict）。
- `references/style-guide.md` — 风格参考（模仿用户既有文件）：脚本组织、
  DESeq2 写法、GSEA 逐个通路出图（gseaplot2/gseaNb/patchwork）、富集可视化、
  配色主题。产出代码与结果时**主动对照此模块**，尽量贴近用户既有风格。

按用户当前所处阶段只加载对应模块，避免一次性全部读入。

## 项目脚手架与脚本

若用户需要从零搭建一个可复现的分析项目，参考并复用：

- `references/project-scaffold.md` — 目录结构、脚本命名规范。
- `scripts/` — 关键可复用函数（见 `references/*.md` 中对应说明）。

## 输出与规范

> 产出脚本与结果时，**主动参照 `references/style-guide.md`** 归纳的用户既有
> 代码/出图风格（脚本组织、DESeq2 写法、GSEA 逐个通路出图、配色主题），
> 尽量贴近用户自己的分析套路；但定位为**风格参考，可灵活调整**。

- 结果给具体数字（p 值、fold change、AUC、样本数），不要只说"显著"。
- 图表遵循投稿风格：配色 `#0074b3`（蓝）/ `#982b2b` 或 `#ae202c`（红），
  `theme_bw`，图内英文，字号宁大勿小，图例用实心方块。
- 显著性与图例：不显著统一标注 "Non-Sig"（不写 ns）；仅在图内标显著项，
  但数据表保留完整 p 值与 ns。
- 分组因子默认字母序；仅有序变量（Stage/Grade）与有方向两组
  （Normal→Tumor、Low→High）才显式设置 `levels`。

**可重复性与命名（Bioshrimp 规范，详见 `references/output-conventions.md`）：**

- 所有含随机性的步骤必须 `set.seed()`；中间结果（合并后的表达矩阵、
  dds 对象、GSEA 对象等）必须落盘保存，勿只在内存中。
- 每次完整分析后**必须生成结果说明文档**（含 R 版本、关键包版本、
  参考基因组/基因集版本、方法参数、结果清单、中途变动）。
- 结果表**优先输出 `.xlsx`**，并**同步输出 `.txt`**（防 Excel 把基因名
  如 MARCH1/SEPT9/DEC1 自动改成日期）。
- 文件名统一为 **`<NN>-Bioshrimp-Beta version-<数据集>-<内容>`**，`NN` 为产出该文件的
  脚本步骤序号（01/02/03/04/05…），置于文件名最前，便于按分析顺序排列。
- **中间对象统一存 `.Rdata`（`save()`/`load()`），不要混用 `.rds`**。
- **单一交付目录（硬性）**：一次分析的全部产物——图、表、中间对象、
  本次用到的脚本、本次下载/派生的数据——全部放进项目根目录下新建的
  `Produced by Bioshrimp/`，内部三层：`program/`（代码）、
  `results/`（所有图 + 表）、`data/`（.Rdata 对象与派生数据）；
  不得散落在项目根目录的 `results/`、`program/`、`data/`。
- **`results/` 按步骤建子文件夹（硬性）**：`results/<NN>-<步骤名>/`，
  如 `results/01-数据整理/`、`results/02-分析前评估/`、`results/03-差异分析/`、
  `results/04-ORA/`、`results/05-GSEA/`。每个步骤的图与表只进自己的文件夹，
  不混放、不共用目录；文件名仍保留 `<NN>-` 前缀，单独拷出时仍可溯源。
  `data/` 保持平铺即可（对象名同样带 `<NN>-` 前缀）。
- **说明文档只写一份，但同时给 `.md` 与 `.docx` 两种格式**（内容一致）：
  md 为源文件，docx 供评审/投稿；不另建 `Produced by Bioshrimp.md` 索引，
  输出清单并入说明文档的"输出文件清单"章节。
- **不得移动用户原有文件**：项目既有 `data/`、`program/`、`results/` 与外部参考库
  （如外部参考库）中的文件一律保持原位，只用只读引用；交付目录只收纳本次新产出。
- **富集出图默认规则**：**ORA** 的点图取 p.adjust 最小的 20 条（不按显著性过滤），
  **点图与条形图成对输出**，**两者横坐标一律为 −log10(p.adjust)**：
  - **ORA**：GO BP/CC/MF + KEGG 四套、Up / Down 两方向各一对（共 8 对）；
    条形图**不放图例**，x 轴左侧 `expand` 固定为 0（0 与 Y 轴重合）。
  - **GSEA**：**上调与下调分开各画一张**，各自在该方向内取最显著的 20 条
    （先按 NES>0 / NES<0 拆分再取 top20），横坐标同为 −log10(p.adjust)；
    方向靠配色区分（上调 `#ae202c`、下调 `#0074b3`），不靠图例；
    点图的 NES 用颜色映射保留，避免丢失方向信息。
- **GSEA 逐条通路图输出全部显著通路**（`p.adjust < 0.05` 全画），不再只取前 10 条；
  对 **GO 与 KEGG** 每个做过的集合都要出（用户点名追加 Hallmark / Reactome 时同样覆盖）；
  显著通路为 0 时不出逐条图并在说明中注明。多通路叠加图（GseaVis `gseaNb`）默认不输出，按需定制。
- GSEA/ORA 在线接口（KEGG 等）失败时，降级到本地 MSigDB `.gmt` / `msigdbr` 的
  `C2:CP:KEGG`（仍是 KEGG，口径不变）；确实取不到 KEGG 基因集时，
  **先告知用户并征得同意**才可改用 Reactome，所有变动写进结果说明。

## 文件边界（硬性，优先于任何分析便利）

- **只允许在用户当前工作目录（项目根目录）内创建、修改、删除文件。**
- 工作目录以外的任何写入，包括但不限于：技能自身（`~/.workbuddy/skills/`）、
  外部参考库（如基因集、GTF、公用数据集）、R 包库（`win-library`）、
  系统或软件配置、其他项目目录、桌面与文档中的既有文件——**必须先说明意图、
  影响范围与回退方式，取得用户明确许可后才可执行**。
- 只读引用不受限制：读取外部 GTF、基因集、参考矩阵无需许可，但要在结果说明中
  写明来源绝对路径，且不得改动原件。
- 安装/升级 R 包、修改 `.libPaths`、建立目录联接、改动默认库中的包版本等
  影响全局环境的操作，同样视为越界，按上一条处理。
- 若确需在工作目录外写临时文件，优先用系统临时目录，并在结束前清理。

## 环境假设

- 上游在 Linux（含 conda/bioconda）；下游在 R ≥ 4.x。
- 关键 R 包：DESeq2、edgeR、limma、tximport、clusterProfiler、org.*.eg.db、
  DOSE、GSVA、GSEABase、FactoMineR、factoextra、pheatmap、ggplot2、rtracklayer。
- 参考基因组：人常用 GRCh38、小鼠常用 GRCm38（GENCODE v44 / vM25 等）。
  上述版本**仅作为向执行人提问时的候选示例，不得作为默认值直接采用**；
  必须以执行人的答复为准并写入结果说明（见"分析前的确认"）。
  参考基因组与基因长度来源需与定量转换口径一致。
