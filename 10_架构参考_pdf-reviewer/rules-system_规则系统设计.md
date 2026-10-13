# 规则系统

## 🤖 Agent 阅读指南

本文档描述 PDF 审查的规则系统设计、规则格式、加载器接口。
修改 `pdf_review/rules/` 或规则数据时必读。

> 2026-07-06 重建：规则库用 ChatGPT 从原始 Excel 重新生成，采用新 8 分类 + 多维数组标签体系。

---

## 概述

按**审查目标**分 8 个分类文件 + YAML 标签化索引，支持按分类/语言/品牌/市场动态筛选。
共 115 条规则。核心设计：**category 管审查目标，其他标签管加载范围**。

---

## 目录结构

```
pdf_review/rules/data/               # 规则数据（权威源）
├── index.yaml                       # 规则元数据索引 (115条)
├── language_localization.md         # 语言与本地化质量 (2条)
├── product_specs_parameters.md      # 产品规格与参数 (32条)
├── feature_configuration.md         # 功能配置一致性 (15条)
├── legal_certification_compliance.md# 法规认证与合规声明 (21条)
├── brand_ip_terminology.md          # 品牌/版权/禁用术语 (11条)
├── document_identity_structure.md   # 文档身份与结构一致性 (16条)
├── safety_installation_accessory.md # 安全/安装/附件 (15条)
├── visual_layout_assets.md          # 视觉版式与图形元素 (3条, 走视觉路)
├── rule_stats.yaml                  # 规则效果统计（运行时生成，不打包）
└── avoidance/                       # 反向规则库（误报防护，独立体系）

rules/                               # 根级副本（导入脚本自动同步 .md + index.yaml）
```

---

## 8 个分类 (category)

| key | 中文名 | 审查目标 | 审查路 |
|-----|--------|---------|--------|
| `language_localization` | 语言与本地化 | 错别字/语法/翻译一致性 | 文本 |
| `product_specs_parameters` | 产品规格参数 | 分辨率/尺寸/重量/单位/数值逻辑 | 文本(含numeric) |
| `feature_configuration` | 功能配置一致性 | 功能描述与规格书/BOM 一致 | 文本(cross_ref) |
| `legal_certification_compliance` | 法规认证合规 | 各市场认证/法规声明 | 文本 |
| `brand_ip_terminology` | 品牌/版权/术语 | 禁用词/品牌/授权声明 | 文本 |
| `document_identity_structure` | 文档身份与结构 | 型号/物料号/页码一致性 | 文本(cross_section) |
| `safety_installation_accessory` | 安全/安装/附件 | 安全警示/安装/附件清单 | 文本 |
| `visual_layout_assets` | 视觉版式与图形 | 二维码/图标/页码位置 | 视觉 |

---

## 规则格式

### index.yaml 条目（多维数组标签）

```yaml
- id: 2
  category: product_specs_parameters
  language: en                       # 单值: all/zh/en/ja/tr/uk/multilingual_compare
  brand: all                         # 单值: all/hisense/toshiba
  market: ['global']                 # 数组: global/cn/na/mx/eu/uk/au/nz/jp/kr/tr/ua/ap/sa/ph
  doc_type: ['paper_manual']         # 数组: paper_manual/quick_guide/e_manual/cb_report
  product_scope: ['flat_tv']         # 数组: flat_tv/laser_projector/microled/led_display/girlfriend_tv/e_manual_product/all
  document_area: ['specifications']  # 数组: full_text/cover/back_cover/specifications/safety/accessories/appearance_connection/toc/file_name/page_footer/e_manual_section
  check_route: ['text', 'numeric']   # 数组: text/numeric/cross_ref/cross_section/visual/metadata
  needs_ref: false
  reviewers: ['A', 'B']              # 由 check_route 自动派生（兼容旧字段）
```

### .md 块（人话内容，给 LLM 看）

```
## 规则 2
```yaml
(上述 index 字段)
```
- 适用范围: 通用英文说明书
- 适用语言: 英文
- 审查内容: 审查英文规格页中的 Screen resolution 是否只使用允许的三种分辨率。
- 审查规则: 正确：...；错误：...
- 示例: 正确：3840 x 2160；错误：38 x 2160
```

### 标签维度说明

- **market/doc_type/product_scope/document_area/check_route** 均为**数组**（一条规则可适用多个值）
- **language/brand** 为单值字符串
- **document_area** 已标注但**系统暂未用于筛选**（待后续做文档区域识别后启用）
- **check_route** → reviewers 派生映射：text/cross_ref/cross_section/metadata→A，numeric→B，visual→C

---

## 加载器接口

```python
from pdf_review.rules.loader import RuleFilter, load_rules

# 按分类 + 元数据筛选（pipeline 用法）
rules_text = load_rules(
    RuleFilter(language="en", brand="all", market="eu",
               needs_ref=False, categories=["product_specs_parameters"]),
    rules_dir,
)
```

**market 数组匹配**：规则 market 含 `global`/`all` 视为通用（始终匹配）；否则文档 market 必须在规则 market 数组中。

---

## 分类 → 审查路（pipeline C-全）

- **7 个文本分类**：α(397b) 和 β(deepseek) 各跑一遍（相同 prompt 交叉验证）
- **visual_layout_assets**：走视觉路（V-版面/V-图文/V-内容 三路）
- 详见 `MODULES/PDF_REVIEW_PIPELINE.md`

---

## 规则库重建流程（2026-07-06）

```
原始 Excel → ChatGPT (RULE_REFINE_GUIDE.md 指导) → 逐批输出 JSON
          → docs/refined_rules_full.json
          → scripts/import_refined_rules.py（备份+重建 index/md+同步双副本）
```

导入脚本自动：备份旧规则到 `rules_backups/`、删旧分类 md、按新分类重建 `index.yaml` + 各 `{category}.md`、派生 reviewers、同步到根级 `rules/`。

---

## 反向规则库 (avoidance/)

误报防护知识库，独立于正向规则。按 category + language/market/brand 加载，追加到 prompt 作为"以下场景不要报为问题"。详见 loader.py 的 `load_avoidance_rules`。

---

## 待办

- 规则描述精炼 + 元数据重新识别（用更强模型逐条优化，回填缺失的 language/market 标签）
- document_area 字段的加载筛选启用（需先做文档区域识别）
- 反向规则库也迁移到新分类体系
