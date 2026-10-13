# PDF 审查流水线 (pdf_review)

## 🤖 Agent 阅读指南

本文档描述 PDF 审查独立包的架构、接口和使用方式。
修改 pdf_review/ 下代码时必读。与 BLUEPRINT 的关系：实现 Phase 6 后期的 PDF 审查能力。

---

## 概述

独立的异步流水线，用于自动审查多语言 PDF 文档（说明书）。
v4 架构：双模型交叉验证 + 独立仲裁，零依赖 my-agent 主框架，可整目录打包移植。

---

## 架构 (v4)

```
预处理 (文本块 + 图片 + OCR + 元数据检测 + 按分类规则加载)
  │
  ├─→ 文本审查α (qwen3-397b no_think, 7个分类并行)
  │     ├── 语言本地化   (language_localization)
  │     ├── 产品规格参数 (product_specs_parameters)
  │     ├── 功能配置     (feature_configuration)
  │     ├── 法规认证     (legal_certification_compliance)
  │     ├── 品牌术语     (brand_ip_terminology)
  │     ├── 文档结构     (document_identity_structure)
  │     └── 安全安装附件 (safety_installation_accessory)
  │
  ├─→ 文本审查β (deepseek-flash, 跑相同 7 个分类, 同一套 prompt)
  │     └── (与 α 完全相同的 7 分类，仅模型不同 → 交叉验证)
  │
  └─→ 视觉审查 (qwen3-vl, 3路×逐页并行, 规则=visual_layout_assets)
        ├── V-版面   (裁切/遮挡/错位/溢出)
        ├── V-图文   (标注指向/图文不符/图注错误)
        └── V-内容   (文字可读性/信息完整性)
            │
            ▼
合并分类 (代码: α∩β=高置信, 仅α=中, 仅β=低)
            │
  ├─→ 文本仲裁 (qwen3-397b no_think, 不加载规则)
  │     去重 + 删误报 + 纠正分类
  │
  └─→ 视觉仲裁 (qwen3-vl, 逐页, 不加载规则)
        去重 + 删误报 + 补充遗漏
            │
            ▼
输出 (审查报告 + 审查详情 + 反馈模板 + 标注PDF + 检查点)
```

> **架构演进 (2026-07-06, C-全)**：审查路由从"按检查动作(α4路查一致性/逻辑/认证/市场，β4路查拼写/数值/命名)"
> 改为"**按规则分类**(7 个文本分类)"。α 和 β **跑相同的 7 个分类、用同一套 prompt**，只是模型不同 →
> 全面交叉验证（旧版 α/β 查不同主题，只有部分重叠）。deepseek 可无缝替换成任意第二模型。
> 对应函数：`_run_text_review(model_kind)`（旧 `_run_text_alpha`+`_run_text_beta` 合并）。

---

## 模型分工

| 角色 | 模型 | 用途 | 关键参数 |
|------|------|------|---------|
| 文本α / 文本仲裁 | qwen3-5-397b | 深度推理、跨页关联 | enable_thinking=False |
| 文本β | deepseek-v4-flash（可替换） | 交叉验证第二模型 | temperature=0 |
| 视觉审查 / 视觉仲裁 | sx-qwen3-vl-flash | 看图、版面、终审确认 | — |
| OCR (可选) | PaddleOCR-VL-1.6 | 文字坐标定位 | 可选，跳过不影响审查 |

---

## 预处理: DocumentMeta 检测

在审查开始前自动检测文档属性，用于规则精确筛选（值已对齐新规则代码）:

| 属性 | 检测方式 | 示例值 |
|------|---------|--------|
| language | 中英日字符比例 + 假名检测 | zh / en / ja / multi / all |
| market | 关键词匹配(CCC/FCC/CE/NOM/假名等) | cn / eu / uk / na / au / jp / kr / tr / ua / ap / all |
| brand | 文件名+文本搜索 | hisense / toshiba / all |
| doc_type | 文件名关键词 | paper_manual / quick_guide / e_manual / cb_report |
| is_outlined | 平均文本块数 <5 | True / False |
| model_name | 文件名正则提取 | 65E8S(02) |
| page_scale | 页数归类 | small(≤4) / medium(5-30) / large(30+) |

---

## 规则系统

### 规则加载

- 按 **category + language + market(数组) + brand + needs_ref** 精确筛选（不再按 reviewer）
- 通用规则(language=all, market 含 global/all)始终保留
- 特定市场/语言规则只在匹配时加载
- **条件加载**: 有 `context_file` 时 `needs_ref` 不过滤(允许加载需要参考文档的规则)
- 详见 `MODULES/RULES_SYSTEM.md`

### 规则分类 (8类, 115条)

| 分类 | 条数 | 审查路 |
|------|------|--------|
| `language_localization` | 2 | 文本(α+β) |
| `product_specs_parameters` | 32 | 文本(α+β) |
| `feature_configuration` | 15 | 文本(α+β) |
| `legal_certification_compliance` | 21 | 文本(α+β) |
| `brand_ip_terminology` | 11 | 文本(α+β) |
| `document_identity_structure` | 16 | 文本(α+β) |
| `safety_installation_accessory` | 15 | 文本(α+β) |
| `visual_layout_assets` | 3 | 视觉 |

### 反向规则库 (avoidance/)

误报防护知识库，按 category + language/market/brand 加载，追加到 prompt 末尾作"以下场景不要报为问题"。

### rule_ref / rule_id 字段

每个问题附带 `rule_ref`（规则描述）+ `rule_id`（关联的规则编号，无明确规则填 null）。

---

## 已知误报模式与修复 (2026-07-06)

| 误报 | 根因 | 修复 |
|------|------|------|
| 断句错误暴增 | 语言路用 397b 强模型 + 宽 prompt（旧版拼写只走 flash 弱模型+窄 prompt 压制） | 语言 prompt 收窄，明确"PDF 提取的跨块换行不是断句错误" |
| 视觉裁切/文字重叠误报 | 喂给视觉模型的 blocks_info 内容被截断到 40 字，模型把"我们的截断"误判为"文档裁切" | blocks_info 给完整文本 + prompt 明确"以图片为准，文本块内容完整" |
| 标注框只框前几个字 | search_for 用前缀命中，rect 只覆盖前缀宽度 | locate.py 命中后扩展到整个文本块 bbox |

---

## 规则学习系统

### 完整流程

```
审查 → 生成反馈模板 → 人工编辑 → 提交反馈 → 积累数据
                                              ↓
                              自动检测阈值 → learn → LLM建议 → 人工确认 → 更新规则
```

### 5 个阶段

| 阶段 | 功能 | 命令 |
|------|------|------|
| 1 | 反馈模板生成 + 解析 | `feedback <路径>` |
| 2 | 排除名单 | 编辑 `rules/data/exclusions.yaml` |
| 3 | 经验积累统计 | `learn --stats` |
| 4 | LLM 辅助规则提炼 | `learn` |
| 5 | 自动提醒 | 审查完成后检测阈值 |

### 反馈模板

每次审查自动生成 `*_反馈.md`，人工标记每个问题:
- ✅ 正确 — 确认是真问题
- ❌ 误报 — 不是问题 (填写原因)
- ⚠️ 部分正确 — 描述不准确 (填写修正)
- 底部可补充遗漏

### 学习触发条件

- ≥5 条新反馈 → 直接提醒
- ≥3 条新反馈 且误报≥3 条 → 提醒

---

## 包结构

```
pdf_review/
├── __init__.py              # 对外接口: run_pipeline / run_from_checkpoint
├── __main__.py              # CLI 入口 (v4默认, --v3回退, feedback, learn)
├── config.py                # PdfReviewConfig (独立配置)
├── document_meta.py         # DocumentMeta 预处理检测 + LLM 增强 + 合并
├── models.py                # LLM 模型构建 (user_key + no_think)
├── pipeline_v4.py           # v4 流水线主逻辑
├── pipeline.py              # v3.1 旧版 (--v3 时使用)
├── trace.py                 # 调用级链路追踪 (全调用落盘)
├── pyproject.toml           # 独立部署配置
├── setup.sh                 # 远程部署初始化脚本
├── prompts/                 # 所有 prompt 模板 (v4.1.0 从代码拆出)
│   ├── __init__.py          # load(name) → str
│   ├── settings.py          # 模型调用参数 (NO_THINK / FAST)
│   ├── meta_detect.txt      # 元数据 LLM 检测 prompt
│   ├── text_review/         # 7 个文本分类审查 prompt
│   │   ├── _output_suffix.txt  # 共享输出格式 + 字段约束
│   │   ├── language_localization.txt
│   │   ├── product_specs_parameters.txt
│   │   ├── feature_configuration.txt
│   │   ├── legal_certification_compliance.txt
│   │   ├── brand_ip_terminology.txt
│   │   ├── document_identity_structure.txt
│   │   └── safety_installation_accessory.txt
│   ├── vision/              # 3 个视觉审查 prompt
│   │   ├── layout.txt
│   │   ├── content.txt
│   │   └── readability.txt
│   └── arbitration/         # 仲裁 prompt
│       ├── text.txt
│       └── vision.txt
├── frontend/                # 前端静态文件 (build 产物, 部署时自动 serve)
├── data/
│   ├── feedback/            # 人工反馈 JSON
│   └── suggestions/         # LLM 规则建议 (.md + .json 结构化)
├── rules/
│   ├── loader.py            # 规则加载器 + 反向规则加载
│   ├── stats.py             # 规则效果统计 (rule_stats.yaml 读写)
│   ├── data/                # 正向规则源文件 (2026-07-06 重建为新 8 分类)
│   │   ├── index.yaml       # 规则索引 (多维数组标签)
│   │   ├── rule_stats.yaml  # 规则效果统计 (运行时生成, 不打包)
│   │   ├── language_localization.md
│   │   ├── product_specs_parameters.md
│   │   ├── feature_configuration.md
│   │   ├── legal_certification_compliance.md
│   │   ├── brand_ip_terminology.md
│   │   ├── document_identity_structure.md
│   │   ├── safety_installation_accessory.md
│   │   └── visual_layout_assets.md
│   └── avoidance/           # 反向规则库 (误报防护)
│       ├── index.yaml       # 反向规则索引 (id/category/language/market/brand)
│       ├── numeric_format.yaml
│       ├── spelling_grammar.yaml
│       ├── naming_consistency.yaml
│       ├── certification.yaml
│       ├── global.yaml      # category=all, 全局生效
│       └── text_exclusions.yaml  # 从 exclusions.yaml 迁入的文本排除
├── webapi/                  # Web 服务 (FastAPI, 可选依赖)
│   ├── app.py              # 路由装配 + 统一错误信封
│   ├── schemas.py          # Pydantic 请求/响应模型
│   ├── auth.py             # 用户认证 (UserStore + 密码哈希 + JWT 签发/验证)
│   ├── jobs.py             # JobRecord + JobRegistry
│   ├── runner.py           # 后台执行 run_pipeline
│   ├── rules.py            # 规则浏览 + 反向规则浏览 + 反馈统计
│   ├── suggestions.py      # 建议审批 (含 avoidance 落库)
│   ├── rule_store.py       # 规则 CRUD + 备份/回滚
│   ├── storage.py          # 产物路径 + 解析辅助
│   ├── security.py         # JWT 鉴权 (2026-07-15 起替代两级静态 Token)
│   └── settings.py         # Web 配置
├── scripts/
│   ├── run_pipeline.py      # v3 入口 (旧)
│   ├── run_from_checkpoint.py
│   ├── parse_feedback.py    # 反馈解析 + 规则统计更新
│   └── learn_rules.py       # 规则学习 (输出 avoidance + 正向规则建议)
└── tools/
    ├── ocr.py               # PaddleOCR-VL 云 API
    ├── vision.py            # 视觉模型调用 (SSE, raise_on_error 开关)
    ├── locate.py            # 统一问题定位器 (6层策略→_bbox_pt, PNG/PDF 共用)
    ├── annotate.py          # 标注图生成
    ├── pdf_parse.py         # pymupdf 文本块提取 + 渲染
    └── pdf_export.py        # 标注 PDF 输出
```

---

## CLI 命令

```bash
# 审查 PDF (v4 默认)
uv run python -m pdf_review "文件.pdf"

# 使用旧版 v3.1
uv run python -m pdf_review --v3 "文件.pdf"

# 从检查点重跑标注
uv run python -m pdf_review --checkpoint "xxx_checkpoint.json"

# 提交人工反馈
uv run python -m pdf_review feedback "输出目录/*_反馈.md"

# 查看反馈统计
uv run python -m pdf_review learn --stats

# 分析反馈 + LLM 生成规则建议
uv run python -m pdf_review learn
```

---

## 输出目录结构

每次审查在 PDF 同级创建独立文件夹:

```
{PDF文件名}_{MMDD-HHMM}/
├── pages/                    # 切片 PNG (每页一张, dpi=150)
├── traces/                   # 失败/空返回调用的原始输出 (排查用)
├── *_审查报告_*.md           # 最终问题清单 (含 rule_ref)
├── *_审查详情_*.md           # 全过程记录 (顶部含调用 trace 表)
├── *_反馈.md                 # 人工复核模板
├── *_标注审查_*.pdf          # 标注 PDF (红框+序号)
├── *_trace_*.json            # 结构化调用链路 trace (机器可读)
└── *_checkpoint_*.json       # 检查点 (可重跑标注)
```

---

## 并发策略

- 文本α 4路: 全部并行 (Semaphore=5)
- 文本β: 拼写/数值用滑动窗口(5页步长4)并发, 命名/一致性全文并行
- 视觉 3路×逐页: 全部并行 (Semaphore=5)
- 仲裁: 文本仲裁(一次调用) + 视觉仲裁(逐页并发) 并行
- OCR: Semaphore=3, 429 重试
- 全局重试: 所有 LLM 调用最多重试 2 次，指数退避 (3s, 6s)
- 实际耗时瓶颈: 视觉审查逐页 (~10s), 文本审查被完全遮蔽 (~3s)

---

## 标注定位 (统一定位器, 6层降级)

定位由 `tools/locate.py` 的 `locate_issues()` **统一完成一次**，与输出格式无关。
结果以 PDF points 写入 `iss["_bbox_pt"]` + `iss["_locate_method"]`，PNG 渲染器与
PDF 批注器共用同一坐标，不再各自重新搜索。

定位后按阅读顺序（页码→纵向 y→横向 x）`_reorder_issues()` 重排并重新编号 id。

6 层策略 (全部归一到 PDF points):
1. pymupdf 精确搜索 original (剥离前导项目符号, 30→20→12→8 前缀, 窄框防护)
2. block_id 匹配 (pdf_blocks 优先, OCR 其次, 像素→points 换算)
3. 关键词宽松搜索文本块
4. 图号关键词 (图一/图二) 关联最近图片
5. 该页最大图片
6. 文本区域中点 / 页面中心兜底

真实数据可定位率 ~100%；统计显示 block_id(~96%) 与 search 是主要命中来源。

---

## 部署方式

```bash
# 打包
uv run python scripts/pack_module.py pdf_review

# 远程部署
unzip pdf_review_v4.0.0.zip
cp .env.example .env && vim .env
uv sync
uv run python -m pdf_review "文件.pdf"
```

---

## 已知问题

1. β 偶发全 0 = deepseek-flash 合法空返回(保守)，非 bug；β-数值正常
2. OCR 在无外网环境不可用（已做优雅降级）
3. 视觉仲裁 JSON 偶尔不规范 → 已定位为英寸符号 `"` 未转义破坏 json.loads (待加容错重试)
4. glm-5.1 无权限，方案D暂只能用现有双模型
5. 转曲 PDF 图片标注路径在定位统一重构后尚未实测
6. 视觉路规则过少 — 当前仅 3 条 layout_structure 规则(筛选后通常 1~2 条), 视觉审查基本靠模型自身能力, 缺乏约束

---

## 待做 (按优先级)

### P1.5 审查精度提升
1. ~~用户补充信息注入~~ ✅ 已完成
2. ~~视觉路规则补充~~ ✅ 已完成 (feature_matching 拆分归入, 视觉路从3→6条)
3. **规则描述精炼** — 用更先进的大模型从原始规则库重新提取更精准的规则描述(逐条优化)

### 规则分类变更记录 (2026-07-02)
原 `feature_matching` 类别(26条)已拆分归入:
- `technical_params` +10条: #10,#15,#16,#22,#50,#54,#112,#113,#114
- `naming_consistency` +5条: #5,#13,#19,#20,#21
- `certification` +4条: #12,#17,#48,#49
- `market_specific` +5条: #38,#39,#47,#51,#77
- `layout_structure` +3条: #31,#35,#74
- `numeric_format` +1条: #72

条件加载: 有 `context_file` 时 `needs_ref=True` 规则也加载(+28% 规则量)。

---

## 附录: 架构图

### v3.1 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         PDF 文件输入                              │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. 预处理                                                       │
│     • pymupdf 文本块提取                                         │
│     • pymupdf 渲染每页 PNG                                       │
│     • PaddleOCR 结构化识别 (可选)                                 │
│     • 规则全量加载                                                │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. 五路并行初审                                                  │
│                                                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────────┐   │
│  │ A-spelling  │ │ A-numeric   │ │ A-global                │   │
│  │ ds-flash    │ │ ds-flash    │ │ qwen3-397b              │   │
│  │ 滑动窗口    │ │ 滑动窗口    │ │ 全文输入                │   │
│  │ 拼写/语法   │ │ 数值/格式   │ │ 跨页一致性/参数矛盾     │   │
│  └─────────────┘ └─────────────┘ └─────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────┐ ┌─────────────────────────────┐   │
│  │ B-layout                │ │ B-content                    │   │
│  │ qwen3-vl               │ │ qwen3-vl                     │   │
│  │ 逐页看图                │ │ 逐页看图 + 3页上下文         │   │
│  │ 裁切/遮挡/错位/溢出     │ │ 标注指向/图文不符/图注错误   │   │
│  └─────────────────────────┘ └─────────────────────────────┘   │
└────────────────────────────────┬────────────────────────────────┘
                                 │ 合并所有初审结果
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. C 终审 (逐页)                                                │
│     • 模型: qwen3-vl (看图)                                      │
│     • 输入: 页面图片 + OCR块 + 该页全部初审issues                 │
│     • 职责: 过滤误报 → 去重 → 纠正分类 → 补充遗漏                │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. 输出                                                         │
│     • 审查报告.md                                                │
│     • 标注PDF (红框+序号)                                        │
│     • 检查点.json                                                │
└─────────────────────────────────────────────────────────────────┘
```

### v4 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         PDF 文件输入                              │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. 预处理                                                       │
│     • pymupdf 文本块提取                                         │
│     • pymupdf 渲染每页 PNG → pages/ 子目录                       │
│     • PaddleOCR 结构化识别 (可选)                                 │
│     • DocumentMeta 检测 (language/market/brand/doc_type)          │
│     • 规则精确加载 (按元数据筛选) + 排除名单                      │
└───────────┬───────────────────┬───────────────────┬─────────────┘
            │                   │                   │
            ▼                   ▼                   ▼
┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐
│ 2a. 文本审查 α    │ │ 2b. 文本审查 β    │ │ 2c. 视觉审查      │
│ qwen3-397b        │ │ deepseek-flash    │ │ qwen3-vl          │
│ (no_think)        │ │                   │ │                   │
│                   │ │                   │ │                   │
│ ┌───────────────┐ │ │ ┌───────────────┐ │ │ ┌───────────────┐ │
│ │ α-一致性      │ │ │ │ β-拼写 (窗口) │ │ │ │ V-版面        │ │
│ │ 跨页参数矛盾  │ │ │ │ 拼写/语法     │ │ │ │ 裁切/遮挡     │ │
│ ├───────────────┤ │ │ ├───────────────┤ │ │ │ 错位/溢出     │ │
│ │ α-逻辑        │ │ │ │ β-数值 (窗口) │ │ │ ├───────────────┤ │
│ │ 尺寸/重量逻辑 │ │ │ │ 数值/格式     │ │ │ │ V-图文        │ │
│ ├───────────────┤ │ │ ├───────────────┤ │ │ │ 标注指向      │ │
│ │ α-认证        │ │ │ │ β-命名 (全文) │ │ │ │ 图文不符      │ │
│ │ 认证声明版本  │ │ │ │ 品牌/型号     │ │ │ ├───────────────┤ │
│ ├───────────────┤ │ │ ├───────────────┤ │ │ │ V-内容        │ │
│ │ α-市场        │ │ │ │ β-一致性(全文)│ │ │ │ 可读性/完整性 │ │
│ │ 市场合规要求  │ │ │ │ (独立复核α)   │ │ │ └───────────────┘ │
│ └───────────────┘ │ │ └───────────────┘ │ │                   │
└─────────┬─────────┘ └─────────┬─────────┘ └─────────┬─────────┘
          │                     │                     │
          └──────────┬──────────┘                     │
                     ▼                                │
┌─────────────────────────────────────┐               │
│  3. 合并分类 (代码逻辑)             │               │
│                                     │               │
│  • α∩β 交集 → 高置信               │               │
│  • 仅α → 中置信                    │               │
│  • 仅β → 低置信                    │               │
└──────────────────┬──────────────────┘               │
                   │                                  │
                   ▼                                  ▼
┌──────────────────────────┐    ┌──────────────────────────┐
│  4a. 文本仲裁             │    │  4b. 视觉仲裁             │
│  qwen3-397b (no_think)   │    │  qwen3-vl (逐页看图)     │
│                          │    │                          │
│  不加载规则              │    │  不加载规则               │
│  • 去重                  │    │  • 去重                   │
│  • 删误报                │    │  • 删误报 (看图确认)      │
│  • 纠正分类/建议         │    │  • 补充遗漏              │
└────────────┬─────────────┘    └────────────┬─────────────┘
             │                               │
             └───────────────┬───────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. 输出                                                         │
│     • 审查报告.md (含 rule_ref 规则依据)                          │
│     • 审查详情.md (全过程记录)                                    │
│     • 反馈模板.md (人工复核用)                                    │
│     • 标注PDF (红框+序号)                                        │
│     • 检查点.json                                                │
└─────────────────────────────────────────────────────────────────┘
```

### 规则学习闭环

```
┌────────────────────────────────────────────────────────────────┐
│  审查完成 → 自动生成 *_反馈.md                                  │
└───────────────────────────┬────────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────────┐
│  人工复核                                                       │
│  • ✅正确 / ❌误报(填原因) / ⚠️部分正确(填修正)                 │
│  • 补充遗漏                                                     │
└───────────────────────────┬────────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────────┐
│  提交反馈: uv run python -m pdf_review feedback "反馈.md"       │
│  → 保存到 pdf_review/data/feedback/*.json                       │
└───────────────────────────┬────────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────────┐
│  积累 → 自动检测阈值 (≥5条新反馈 或 ≥3条误报)                   │
│  → 提示: 💡 建议运行 learn                                      │
└───────────────────────────┬────────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────────┐
│  规则学习: uv run python -m pdf_review learn                    │
│                                                                 │
│  1. 统计分析 (准确率/误报模式/遗漏模式)                          │
│  2. LLM 分析反馈 → 生成建议                                     │
│     • 新增排除模式 (pattern + reason)                           │
│     • 新增规则 (category + description)                         │
│     • 修改建议 (target + change)                                │
│  3. 保存到 pdf_review/data/suggestions/*.md                     │
└───────────────────────────┬────────────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────────────┐
│  人工确认 (必须)                                                 │
│  • 合理的排除模式 → 加入 exclusions.yaml                        │
│  • 合理的新规则 → 加入 {category}.md + index.yaml               │
│  • 合理的修改 → 编辑对应规则文件                                 │
└───────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
              ┌──────────────────────────┐
              │  下次审查自动生效         │
              │  → 更少误报, 更少遗漏    │
              └──────────────────────────┘
```
