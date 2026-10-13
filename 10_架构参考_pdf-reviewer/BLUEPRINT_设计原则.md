# 蓝图 - PDF 审查系统

## 🤖 Agent 阅读指南

本文档是项目最高层级文档。定义项目身份、核心原则和顶层规划。
任何其他文档不得违背此处定义的原则。遇到两难决策时以此为准。

---

## 项目身份

- **名称**：PDF 审查流水线（pdf-reviewer）
- **定位**：自动审查多语言 PDF 文档（电视说明书），双模型交叉验证 + 独立仲裁
- **用户**：海信集团说明书质量审查团队
- **运行环境**：远程 Linux 服务器（Web 服务）+ 本地 Windows（开发 + CLI）
- **技术栈**：Python 3.11+ / PydanticAI / FastAPI / pymupdf / Vite+React（前端）

---

## 核心原则（优先级从高到低）

1. **审查准确第一** — 宁可漏报不误报，每条问题必须有依据（规则编号或推理说明）
2. **规则驱动** — 审查行为由规则库约束，不靠模型自由发挥；规则可追踪、可学习、可回滚
3. **双模型交叉验证** — 强模型(α) + 快模型(β) 跑相同分类，交集高置信，单方中/低置信
4. **独立可移植** — 零依赖外部项目，一个 `git clone` + `uv sync` 就能跑
5. **Pydantic 建模** — 数据模型、配置、API schema 统一用 Pydantic
6. **可观测** — 每步记录 trace（类型+参数+结果+耗时+token），问题可回溯

---

## 项目顶层规划

- **目标**：为说明书审查团队提供稳定的 Web 审查服务，持续通过反馈闭环提升审查精度
- **核心能力**：
  - 文本审查（7 分类 × α+β 交叉验证）
  - 视觉审查（3 路逐页看图）
  - 仲裁去重（文本+视觉独立仲裁）
  - 标注 PDF 输出（红框+序号）
  - 规则学习闭环（反馈→积累→建议→审批→落库）
  - Web 管理（上传/审查/反馈/规则编辑/备份回滚）

---

## 架构要点

- **8 分类规则体系**：language_localization / product_specs_parameters / feature_configuration / legal_certification_compliance / brand_ip_terminology / document_identity_structure / safety_installation_accessory / visual_layout_assets
- **115 条正向规则 + 16 条反向规则**（误报防护）
- **前后端一体**：后端 serve 前端静态文件，统一 `/pdf-reviewer/` 子路径
- **双向同步**：本地开发 push → GitLab ← 服务器规则变更 sync_rules.sh push

---

## 决策偏好

- 模型选择：优先用内网聚好看模型 API，不依赖外网
- 国内网络：uv 用清华镜像
- 输出文件名带时间戳（MMDD-HHMM）
- 规则变更必须备份（自动全量快照）
- 新能力优先加规则（低成本），其次改 prompt，最后改代码
- LLM 输出不可信：每个输出解析都要 try/except + 类型检查 + 缺省值

---

## 工程规则

1. **外科手术式修改**：只改必须改的，不顺手"改进"周边代码
2. **改完就验**：每步有明确验证方式，不攒到最后
3. **容错先枚举畸形**：修解析类问题先列举可能的输入畸形种类再定方案
4. **中文用 fs_write**：不用 PowerShell Set-Content 写中文文件
5. **并发有限**：全局 Semaphore 控制，防打爆 LLM 配额
