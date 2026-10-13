# PDF 审查流水线 — 使用说明

## 基本用法

```bash
# 审查一个 PDF 文件
uv run python -m pdf_review "doc/说明书.pdf"

# 审查指定路径的文件
uv run python -m pdf_review "/home/hisense/ai-team/pdf-reviewer/doc/错误样本.pdf"

# 从检查点重跑标注（不重新调 LLM，秒出）
uv run python -m pdf_review --checkpoint "错误样本_checkpoint_0624-1530.json"

# 查看帮助
uv run python -m pdf_review
```

## 产出物

运行完成后，在 PDF 同目录下生成：

| 文件 | 说明 |
|------|------|
| `xxx_审查报告_MMDD-HHMM.md` | Markdown 格式审查报告，列出所有问题 |
| `xxx_标注审查_MMDD-HHMM.pdf` | 标注后的 PDF（红框+序号+批注） |
| `xxx_checkpoint_MMDD-HHMM.json` | 检查点文件（可重跑标注，不花 token） |

## 审查流程

流水线自动执行以下步骤：

```
1. 预处理 — 提取文本块 + 渲染页面图片 + OCR(可选)
2. A-local 拼写/语法审查 — 5页滑动窗口，快速模型
3. A-local 数值/格式审查 — 同上
4. A-global 全局审查 — 全文送强模型，检查跨页不一致
5. B-layout 版面结构审查 — 逐页看图，检查裁切/遮挡/错位
6. B-content 图文内容审查 — 逐页看图+上下文，检查标注/图文不符
7. C 终审 — 逐页验证所有问题，过滤误报，去重，纠正分类
8. 标注+报告+PDF导出
```

## 配置说明

所有配置通过项目根目录的 `.env` 文件管理：

```bash
# 编辑配置
nano .env
```

### 关键配置项

| 变量 | 说明 | 必填 |
|------|------|------|
| `LLM_BASE_URL` | 主文本模型 API 地址 | 是 |
| `LLM_API_KEY` | 主文本模型密钥 | 是 |
| `LLM_MODEL` | 主文本模型名 (如 qwen3-5-397b) | 是 |
| `LLM_USER_KEY` | 聚好看 user_key | 是 |
| `LLM_FAST_MODEL` | 快速模型名 (如 deepseek-v4-flash) | 是 |
| `VISION_BASE_URL` | 视觉模型 API 地址 | 是 |
| `VISION_API_KEY` | 视觉模型密钥 | 是 |
| `VISION_MODEL` | 视觉模型名 (如 sx-qwen3-vl-flash) | 是 |
| `PADDLEOCR_TOKEN` | PaddleOCR 云 API token | 否(无则跳过OCR) |
| `SSL_VERIFY` | SSL 验证，内网设 false | 否(默认true) |

### 无 OCR 模式

如果目标机器无法访问外网（PaddleOCR 云 API），流水线会自动跳过 OCR：
- A 路文本审查：不受影响（用 pymupdf 提取的文本块）
- B/C 视觉审查：正常工作（看图分析）
- 标注定位：精度略降（缺少 OCR bbox，用文字搜索定位）

也可以手动关闭 OCR：把 `.env` 里的 `PADDLEOCR_TOKEN` 留空即可。

## 运行示例

```
$ uv run python -m pdf_review "doc/错误样本.pdf"

PDF: doc/错误样本.pdf
模型: text=qwen3-5-397b, fast=deepseek-v4-flash, vision=sx-qwen3-vl-flash

>> PDF 审查流水线启动 - doc/错误样本.pdf
>> 1/8 预处理
   ⚠ OCR 全部失败 (...)，跳过 OCR，仅用 pymupdf 文本块
   18 页, 规则已加载
>> 2/8 A-local 拼写/语法审查
   A-local-spelling: 12 处
>> 2.5/8 A-local 数值/格式审查
   A-local-numeric: 3 处
>> 3/8 A-global 全局审查
   A-global: 2 处
>> 4/8 B-layout 版面结构审查
   B-layout: 1 处
>> 4.5/8 B-content 图文内容审查
   B-content: 4 处
>> 5/8 C 终审 (验证 22 处)
   C 终审: 18 处最终问题
>> 6/8 标注 (18 处)
>> 7/8 报告: doc/错误样本_审查报告_0624-1530.md
>> 8/8 标注PDF: doc/错误样本_标注审查_0624-1530.pdf
>> 完成: 18 处问题
```

## 从检查点重跑

检查点保存了所有中间结果（各路审查的 issues、页面图片路径等）。用途：

- **调整标注样式**后重新生成标注 PDF，不需要重新调 LLM
- **导出不同格式的报告**
- **调试**：查看各路审查器分别报了什么

```bash
uv run python -m pdf_review --checkpoint "doc/错误样本_checkpoint_0624-1530.json"
```

## 常见问题

### Q: 运行很慢？
A: 主要耗时在 LLM API 调用。18 页 PDF 大约需要 3-5 分钟。瓶颈是 B/C 视觉审查（逐页看图）。

### Q: 某些问题是误报？
A: C 终审已经在过滤误报，但不完美。可以通过调整 `rules/data/` 下的规则文件来减少特定类型的误报。

### Q: 支持中文 PDF 吗？
A: 支持。流水线对多语言 PDF 都能处理，规则库覆盖中英文。

### Q: 转曲的 PDF（文字变成图片）怎么办？
A: 流水线会自动检测。转曲 PDF 的 pymupdf 文本块会很少，此时 A 路审查依赖 OCR 文本（如果 OCR 可用），B/C 视觉审查不受影响。标注 PDF 会用图片标注方式而非批注方式。
