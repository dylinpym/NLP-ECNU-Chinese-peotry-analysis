version https://git-lfs.github.com/spec/v1
oid sha256:cb802c198ea785256408f7d51687c9aac41d4230b7a56f54ea2856ab2ad193e1
size 19061


# COPE — Classical Poetry Editor

古典诗歌辅助写作与分析系统 —— 融合统计语言模型与深度学习的唐诗创作工具。

## 目录

- [功能概览](#功能概览)
- [界面与交互](#界面与交互)
- [安装与运行](#安装与运行)
- [使用指南](#使用指南)
  - [主编辑器](#主编辑器)
  - [风格分析](#风格分析)
  - [逐句近邻](#逐句近邻)
  - [自动补全](#自动补全)
  - [导出与快捷键](#导出与快捷键)
- [技术架构](#技术架构)
- [NLP 技术栈](#nlp-技术栈)
- [项目结构](#项目结构)
- [数据文件](#数据文件)
- [训练模型](#训练模型)
- [数据来源](#数据来源)
- [License](#license)

---

## 功能概览

| 功能 | 说明 |
|------|------|
| **格律检测** | 实时检查平仄、押韵、孤平、三平调、失粘对、重韵等格律违规，逐字标注合律/出律/待审状态 |
| **输入校验** | 方格输入模式下，出律字会被实时拦截并弹出警告，确保输入即合律 |
| **风格分析** | TF-IDF BoW + Word2Vec 语义扩展 + KNN 余弦距离，匹配 Top-256 唐代诗人中风格最接近者 |
| **逐句近邻** | 两阶段检索（BoW 粗筛 + Transformer 语义重排），从全唐诗 ~86K 联句中检索最相似诗句 |
| **自动补全** | N-gram（Kneser-Ney 平滑）+ Transformer 融合，结合断句感知动态权重，根据已输入字自动生成整首诗 |
| **断句强化** | 五言 2\|3、七言 2\|2\|3 停顿边界感知，动态调整 n-gram 权重与位置先验 |

---

## 界面与交互

COPE 提供 PyQt6 桌面应用，采用深色主题（Dark Mode），主要界面元素：

- **Ribbon 工具栏**：顶部固定工具栏，可实时切换韵书（平水韵 / 词林正韵）、体裁（五言 / 七言 / 词牌）及具体格律形式（如"五律平起首句不入韵"），切换即时生效并刷新当前格律报告
- **笔记本编辑区**：每首诗一个单元格（Cell），左侧为纯文本编辑器，右侧为实时格律报告面板
- **方格输入模式**：可选启用的替代输入方式，每个字独立方格，逐字输入时自动跳格并实时拦截出律字
- **分析对话框**：独立窗口，包含风格分析、逐句近邻、自动补全三个选项卡，支持二次编辑诗句后重新分析。自动补全选项卡自带体裁/格律/韵书选择器，修改后自动同步回主编辑器

---

## 安装与运行

### 环境要求

- Python 3.10+
- PyTorch（推荐 2.0+，CPU 或 CUDA）
- PyQt6 ≥ 6.5
- NumPy ≥ 1.24

### 安装

```bash
# 安装基础依赖
pip install -r requirements.txt

# 单独安装 PyTorch（根据 CUDA 版本选择）
pip install torch               # CPU
pip install torch --index-url https://download.pytorch.org/whl/cu118  # CUDA 11.8
pip install torch --index-url https://download.pytorch.org/whl/cu124  # CUDA 12.4
```

### 运行

```bash
python main.py
```

> **注意**：Anaconda 环境下如遇 PyQt6 DLL 路径冲突，`main.py` 已内置处理逻辑，会自动将 PyQt6 的 Qt6 bin 目录置于 PATH 最前端。

---

## 使用指南

### 主编辑器

- **单元格管理**：每首诗为一个独立单元格，支持新建（`Ctrl+N`）、删除（`Ctrl+D`）、撤销/重做（`Ctrl+Z` / `Ctrl+Y`）
- **体裁与格律**：通过顶部 Ribbon 工具栏下拉菜单选择：
  - **韵书**：平水韵（诗）或词林正韵（词）
  - **体裁**：五言、七言、词牌
  - **格律**：根据体裁动态加载可用格律，如"五律平起首句不入韵""七绝仄起首句入韵"及各词牌名
- **编辑模式**：左侧文本编辑器直接输入，或通过方格逐字填入（出律字实时拦截）
- **格律报告**：右侧面板实时显示：
  - 逐字平仄符号（○ 平 / ● 仄 / ◉ 中）与要求对照
  - 韵脚标注（韵部名称）
  - 合律/出律/待审判定
  - 错误汇总（孤平、三平调、落韵、失粘对、重韵等）
- **查找**：`Ctrl+F` 全局搜索文本
- **自动保存**：每 2 秒自动保存至 `savefiles/mypoems.json`

### 风格分析

通过"分析"菜单 → "风格分析"或 `Ctrl+1` 打开分析对话框。

**工作流程**：
1. 提取诗句中的汉字（过滤标点，繁转简）
2. 构建 TF-IDF 归一化的字频向量（维度 K=1024，取频率最高的 1024 个汉字为索引）
3. **Word2Vec 语义扩展**：每个字向与其最相似的 5 个字扩散部分权重（如"月"也部分计入"星""夜""魄"），捕捉语义关联
4. 与 256 位唐代诗人的预计算 BoW 向量进行 KNN 余弦距离匹配
5. 以进度条柱状图展示 Top-10 最相似诗人及其匹配度百分比

### 逐句近邻

通过"分析"菜单 → "逐句近邻"或 `Ctrl+2` 打开分析对话框。

**两阶段检索**：
1. **BoW 粗筛**：将诗句按联分组，计算字面重叠度（含 bigram 加分 + 字频加权），从全唐诗 ~86K 联句中筛选 Top-200 候选
2. **语义重排**：利用预计算的 Transformer 隐层平均池化向量（`couplet_vectors.npy`，220 MB），计算余弦相似度并与 BoW 分数融合排序

**结果展示**：每联最相似的若干诗句、出处（作者 + 诗题）、词袋分数与语义相似度。

### 自动补全

通过"分析"菜单 → "自动补全"或 `Ctrl+Enter` 打开分析对话框。

> 自动补全界面自带**独立的体裁/格律/韵书选择器**，初始默认"五律平起首句不入韵 + 平水韵"，不受主编辑器当前设定影响。修改选择器后会自动同步到主编辑器当前单元格，保持两边一致。

**操作步骤**：
1. 在顶部选择器中设定体裁（五言/七言/词牌）、格律形式和韵书
2. 在方格中输入已知字（留空需补全的位置）
3. 出律字会被实时拦截丢弃，无需手动检查
4. 点击"运行"，系统生成多个候选补全
5. 通过"上一个/下一个"按钮切换候选结果
6. 切换时自动将当前结果同步到主编辑器单元格，右侧格律检测面板即时刷新

**生成算法** —— N-gram + Transformer 混合模型 + 断句感知：

| 组件 | 权重策略 |
|------|----------|
| **Trigram** (Kneser-Ney 平滑) | 停顿组内权重 0.80，跨停顿边界权重降低 |
| **Bigram 回退** | 跨停顿边界时权重升至 0.50，辅以位置先验 |
| **Unigram 回退** | 最低优先级的回退层 |
| **Transformer 神经 LM** | 融合权重 0.30，提供语义连贯性 |
| **位置 Bigram 先验** | 停顿边界位置混入位置特定 bigram 统计 |
| **重复惩罚** | 已出现字乘以 0.70 惩罚系数 |
| **虚词惩罚** | 虚词（之、也、而、以等）乘以 0.85 惩罚系数 |

**约束条件**：押韵约束、平仄匹配、禁止三连重复字。搜索采用重试策略确保候选多样性，最多返回 5 个候选。

### 导出与快捷键

| 操作 | 快捷键 / 路径 |
|------|--------------|
| 新建单元格 | `Ctrl+N` |
| 删除当前单元格 | `Ctrl+D` |
| 撤销 / 重做 | `Ctrl+Z` / `Ctrl+Y` |
| 查找文本 | `Ctrl+F` |
| 打开分析对话框 | `Ctrl+1` 风格分析 / `Ctrl+2` 逐句近邻 / `Ctrl+Enter` 自动补全 |
| 复制格律报告 | `Ctrl+Shift+C` |
| 导出为纯文本 | 文件 → 导出笔记本为纯文本 |
| 导出为 Markdown | 文件 → 导出笔记本为 Markdown |
| 复制分析结果 | 分析对话框 → 复制结果 |
| 保存分析结果 | 分析对话框 → 保存结果为文本 |

---

## 技术架构

```
┌─────────────────────────────────────────────────────┐
│                   PyQt6 桌面 GUI                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │ 主编辑器  │  │ 方格输入  │  │ 分析对话框（三合一）│   │
│  │CellWidget │  │CharGrid  │  │ 风格·近邻·补全    │   │
│  └────┬─────┘  └────┬─────┘  └────────┬─────────┘   │
├───────┼─────────────┼─────────────────┼─────────────┤
│       ▼             ▼                  ▼              │
│  ┌────────────────────────────────────────────────┐  │
│  │              核心分析引擎 (analytics.py)         │  │
│  │  BoW嵌入 │ LineNN两阶段检索 │ MarkovSug混合生成 │  │
│  └──────────────────────┬───────────────────────┘  │
│       │                  │                  │        │
│       ▼                  ▼                  ▼        │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Word2Vec │  │  Transformer  │  │  Trigram LM   │  │
│  │ 词嵌入   │  │  神经语言模型  │  │  Kneser-Ney   │  │
│  │ d=256   │  │  6层 d=256   │  │  平滑 N-gram  │  │
│  └──────────┘  └──────────────┘  └──────────────┘  │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │        格律规则引擎 (meter.py)                 │   │
│  │  平水韵平仄 │ 韵部匹配 │ 孤平/三平调/失粘对   │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

---

## NLP 技术栈

| 技术领域 | 实现方案 | 关键参数 |
|----------|----------|----------|
| **N-gram 语言模型** | Kneser-Ney 平滑 Trigram，bigram/unigram 回退插值 | D=0.75，词汇量 7,211 |
| **神经语言模型** | 从零训练的 Decoder-only Transformer | 6 层，d_model=256，4 heads，FFN=1024，~6.2M 参数 |
| **词嵌入** | Skip-gram Word2Vec + Negative Sampling | d=256，窗口 5，5 负样本 |
| **混合生成** | N-gram + Transformer 加权融合 + 断句边界感知动态权重 | Transformer 权重 0.30，组内 trigram 权重 0.80 |
| **文本分类** | TF-IDF BoW + Word2Vec 语义扩展 + KNN | Top-256 诗人，余弦距离，K=1024 维度 |
| **语义相似度** | Transformer 隐层平均池化 + 余弦相似度 + 两阶段检索 | BoW 粗筛 200 候选 → 语义重排 |
| **格律规则** | 平水韵 106 韵部 + 平仄表 + 格律模板 | 16 种诗格律 + 若干词牌 |

---

## 项目结构

```
Chinese poetry analysis/
├── main.py                              # 应用入口（PyQt6 桌面应用）
├── requirements.txt                     # Python 依赖清单
├── README.md                            # 项目文档（本文件）
│
├── src/                                 # 核心源代码
│   ├── app.py                           # 主 UI 逻辑
│   │   ├── MainWindow                   #   主窗口、Ribbon 工具栏、菜单
│   │   ├── CellWidget                   #   单首诗编辑单元格（编辑器 + 格律报告）
│   │   ├── PreviewWidget                #   格律实时报告面板
│   │   └── AnalysisDialog               #   分析对话框（风格/近邻/补全三选项卡）
│   ├── analytics.py                     # 核心分析引擎
│   │   ├── do_umap_embedding()          #   风格分析（BoW + W2V 扩展 + KNN）
│   │   ├── do_line_nn_with_semantic()   #   逐句近邻（两阶段检索）
│   │   └── do_markov_sug()             #   自动补全（混合语言模型生成）
│   ├── transformer.py                   # Decoder-only Transformer 语言模型
│   │   ├── PoetryTransformer            #   6 层 Transformer + 正弦位置编码
│   │   └── transformer_next()          #   推理接口：给定前缀，返回下一字概率分布
│   ├── meter.py                         # 格律检查引擎
│   │   ├── check_meter()               #   全面格律检测
│   │   ├── get_tone_pattern()           #   单字平仄查询
│   │   └── split_text_to_lines()       #   文本分行
│   ├── constants.py                     # 常量定义与数据懒加载
│   │   ├── get_meters() / get_cimeters()#   格律数据
│   │   ├── get_rhymebooks()            #   韵书数据
│   │   └── get_kangxi()                #   康熙字典（繁简转换）
│   ├── char_grid.py                     # 字符方格编辑控件
│   │   ├── CharGridWidget               #   格律网格布局
│   │   └── CharCell                     #   单字编辑格（实时出律拦截）
│   └── io.py                            # 文件 I/O 工具（笔记本读写、导出）
│
├── scripts/                             # 训练与数据预处理脚本
│   ├── build_trigram.py                 # 构建 Kneser-Ney 平滑 Trigram 模型
│   ├── train_transformer.py             # 训练 Transformer 语言模型（30 epoch）
│   ├── train_word2vec.py                # 训练 Skip-gram Word2Vec 词嵌入
│   ├── rebuild_bow.py                   # 重建 Top-256 诗人 BoW 向量（TF-IDF 归一化）
│   ├── precompute_couplet_vectors.py    # 预计算全唐诗联句语义向量（加速 LineNN）
│   ├── precompute_poet_w2v.py           # 预计算诗人平均 Word2Vec 向量
│   ├── precompute_position_stats.py     # 预计算位置特定 Bigram 统计（断句强化）
│   └── get top256 poet.py               # 提取全唐诗中诗作数量 Top-256 的诗人
│
├── data/                                # 数据文件、模型检查点、预计算缓存（~425 MB）
│   ├── quantang-lines.json              # 全唐诗语料（~53,000 首诗，按行解析）
│   ├── meters.json                      # 诗格律模板（五言/七言 律/绝，平起/仄起）
│   ├── ci-meters.json                   # 词牌格律模板
│   ├── rhymebooks.json                  # 平水韵字典（106 韵部）
│   ├── TC2SC.json                       # 繁简汉字转换表
│   ├── chars-freq.tsv                   # 字符频率表（Top-1024）
│   ├── trigram_model.json               # Kneser-Ney Trigram 模型（65 MB）
│   ├── transformer_model.pt             # Transformer 模型检查点（34 MB）
│   ├── word2vec.pt                      # Word2Vec 词向量（11.3 MB）
│   ├── poets-bow.json                   # 诗人 BoW 向量（5.3 MB）
│   ├── poets_w2v.npy                    # 诗人 Word2Vec 平均向量（2.2 MB）
│   ├── poets_w2v_idx.json               # 诗人索引
│   ├── couplet_vectors.npy              # 联句语义向量缓存（220 MB）
│   ├── couplet_vectors_idx.json         # 联句索引（26 MB）
│   └── position_bigrams.json            # 位置 Bigram 统计（34 MB）
│
├── test_data/                           # 测试用例（按功能分类）
│   ├── 自动补全/                         #   空白生成、部分填充测试
│   ├── 逐句相邻分析/                     #   化用检测、诗句溯源测试
│   └── 词嵌入与词袋  风格分析/            #   风格模仿、诗人相似度测试
│
└── savefiles/                           # 用户笔记本自动保存目录
    ├── example.json                     #   示例笔记本（杜甫诗选）
    └── mypoems.json                     #   用户创作自动保存
```

---

## 数据文件

| 文件 | 大小 | 说明 |
|------|------|------|
| `quantang-lines.json` | 15.8 MB | 全唐诗 ~53,000 首诗，解析为 `{title, author, data: [lines]}` 格式 |
| `rhymebooks.json` | 89 KB | 平水韵 106 韵部，每韵部下列举所属汉字 |
| `meters.json` | 1.2 KB | 16 种诗格律模板（五言/七言 × 律/绝 × 平起/仄起 × 首句入韵/不入韵） |
| `ci-meters.json` | 5.5 KB | 词牌格律模板 |
| `TC2SC.json` | 38 KB | 繁体中文 → 简体中文汉字映射表 |
| `trigram_model.json` | 65 MB | Kneser-Ney 平滑后的 unigram / bigram / trigram 概率表 |
| `transformer_model.pt` | 34 MB | 6 层 Decoder-only Transformer 完整检查点（含词汇表） |
| `word2vec.pt` | 11.3 MB | Skip-gram 词向量（7,211 字 × 256 维） |
| `poets-bow.json` | 5.3 MB | Top-256 诗人 TF-IDF 归一化 BoW 向量 |
| `couplet_vectors.npy` | 220 MB | 全唐诗 ~86K 联句的 Transformer 隐层语义向量（预计算） |
| `couplet_vectors_idx.json` | 26 MB | 联句向量索引（映射至标题/作者/文本） |
| `position_bigrams.json` | 34 MB | 五言/七言各位置（2|3 或 2|2|3 停顿边界）的字符 bigram 频率 |

---

## 训练模型

所有模型均可从零训练。建议按以下顺序执行：

```bash
# 1. 构建 Kneser-Ney 平滑 Trigram 模型（基础 N-gram，其他模型依赖其词汇表）
python scripts/build_trigram.py

# 2. 训练 Skip-gram Word2Vec 词嵌入（d=256，风格分析语义扩展）
python scripts/train_word2vec.py

# 3. 训练 Decoder-only Transformer 语言模型（CPU 约 30 分钟 / 30 epoch）
python scripts/train_transformer.py

# 4. 预计算联句语义向量（加速逐句近邻，需先训练 Transformer）
python scripts/precompute_couplet_vectors.py

# 5. 重建诗人 BoW 向量（需先训练 Word2Vec）
python scripts/rebuild_bow.py

# 6. 预计算诗人 W2V 平均向量（风格分析辅助）
python scripts/precompute_poet_w2v.py

# 7. 预计算位置 Bigram 统计（断句强化，需先构建 Trigram）
python scripts/precompute_position_stats.py
```

---

## 数据来源

| 数据 | 来源 | 用途 |
|------|------|------|
| **全唐诗** | 清康熙年间编《全唐诗》，含 ~53,000 首唐代诗歌 | 训练 N-gram、Transformer、Word2Vec；LineNN 检索语料 |
| **平水韵** | 《平水韵》106 韵部体系 | 平仄判定、押韵检测、自动补全韵脚约束 |
| **词林正韵** | 清代戈载《词林正韵》 | 词牌填词押韵检测 |
| **康熙字典** | 清代《康熙字典》 | 繁简汉字转换表 |

---

## License

MIT License — 详见 [LICENSE](LICENSE) 文件。
