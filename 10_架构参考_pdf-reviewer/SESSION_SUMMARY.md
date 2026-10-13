# 当前会话状态

## 🤖 快速导航（Agent 进入时只需读这一段）

- **当前阶段**：v4.2.0 运营阶段 + 账号体系已部署到服务器
- **当前焦点**：prompt 统一 schema + 前端 UI 大改已完成并部署
- **项目位置**：`D:\AAAAAAAA\projects\PDF`
- **Git 远程**：`git@10.18.203.174:full-stack-frontend/hisense/agents/ai_applications/pdf-reviewer.git`
- **相关文档**：
  - `docs/architecture.md` — 审查逻辑（双模型交叉验证 + 仲裁）
  - `docs/rules-system.md` — 规则分类体系（8 分类 + 多维标签）
  - `docs/web-api.md` — Web API 接口契约（含最新认证接口）
  - `README.md` — 启动/部署相关（含账号管理说明）
  - `账号密码表.md` — 当前所有账号明文密码（不提交 git，仅供分发）

---

## 当前任务

- [x] pdf_review 从 my-agent 独立为完整项目
- [x] 前后端 monorepo + Git 推送 GitLab
- [x] 统一子路径 /pdf-reviewer/ 访问方案（本地+nginx）
- [x] 文档迁移（architecture/rules/web-api/model-api/CHANGELOG）
- [x] 服务器 SSH 配置 + 首次 clone 部署
- [x] nginx 代理验证（`http://10.19.98.208:4657/pdf-reviewer/` 已确认可访问）
- [x] 规则描述精炼 + 元数据重新识别导入（用更强模型逐条优化）
- [x] ProductAgentPlatform 接入（注册+上报埋点+服务器部署）
- [x] 用户账号体系第一步：用户名+密码 + JWT（后端 + 前端 + 服务器均已部署）
- [x] 预置 19 个账号（1 admin + 12 chan-pin + 6 实名），密码表见 `账号密码表.md`
- [x] 任务存储改为按部门分区 + 老任务迁移脚本
- [x] 路径 cwd 依赖 bug 全面排查修复（P1-P5）
- [x] 反馈积累自动触发 learn 分析功能（`LEARN_AUTO_THRESHOLD=5`）
- [x] Prompt 统一输出 schema（文本路+视觉路统一字段）
- [x] 前端 UI 大改（字号提升 + 规则展示规范 + 反馈提示优化）
- [x] 服务器部署完成（git pull + .env 配置 + users.yaml + 重启）
- [ ] 看板累计执行次数显示验证（需管理员触发 aggregate）
- [ ] (可选) 补中国市场法规规则: CCC / 中国RoHS / 制造商信息

---

## 项目当前状态

- **可运行入口**：
  - CLI: `uv run python -m pdf_review "xxx.pdf"`
  - Web: `uv run python -m pdf_review.webapi` → `http://localhost:8000/pdf-reviewer/`
  - ⚠️ 必须在**项目根目录**运行，不要 cd 进 `pdf_review/` 子目录

- **已部署环境**：
  - 本地 Windows: `D:\AAAAAAAA\projects\PDF\`
  - 远程 Linux: `hisense@10.18.87.44:~/ai-team/new-pdf-agent/backend_v5/pdf-reviewer/`
  - nginx: `http://10.19.98.208:4657/pdf-reviewer/`（已验证可用）

- **访问地址统一**：所有场景均为 `/pdf-reviewer/` 子路径

---

## 已知问题

- **中国市场法规规则缺口**：缺 CCC / 中国RoHS / 制造商信息等 needs_ref=false 规则，暂缓
- **视觉仲裁 JSON 偶尔不规范**：英寸符号 `"` 未转义（已有兜底，待加容错重试）
- **转曲 PDF 图片标注定位**：定位统一重构后尚未实测
- **前端更新后浏览器白屏**：部署新前端后用户需 Ctrl+Shift+R 强刷（JS/CSS 文件名 hash 变了，旧缓存 404）

---

## 设计决策记录（防止重蹈覆辙）

### rule_ref 字段设计（2026-07-16 定稿）
- **有规则时**（rule_id > 0）：rule_ref 填规则核心审查要点，**不带"规则N："编号前缀**（编号由 rule_id 承载）
- **无规则时**（rule_id = 0）：rule_ref 固定填 `"模型自主发现"`，实际推理原因写在 suggestion 里
- **前端展示**：有规则显示 `规则 {rule_id}: {rule_ref}`；无规则显示 `模型自主发现`
- **前端兼容老数据**：正则 strip 掉 rule_ref 中 `规则 N：` 前缀；纯"逻辑推测"显示为"模型自主发现"
- **根因**：LLM 对 rule_ref 输出格式不一致（5 种幻觉格式），通过 prompt zero-shot 示例 + 前端清洗双保险

### 输出 schema 统一（2026-07-16 定稿）
- 所有审查路（文本 7 路 + 视觉 3 路）统一输出字段：`id, page, block_id, original, issue_type, suggestion, rule_ref, rule_id`
- **去掉 description 字段**：原视觉路的 description 内容合并到 suggestion（"分析+建议"一体）
- 前端对老数据兼容：如果 checkpoint 中有 description，拼到 suggestion 前展示

### 前端字号规范（2026-07-16）
- 全站只用三种字号：`text-lg`(18px) / `text-xl`(20px) / `text-2xl`(24px)
- 不再使用 text-xs(12px) / text-sm(14px) / text-base(16px)

### 部署注意事项
- 前端构建产物 `pdf_review/frontend/` 纳入 git 同步，服务器 git pull 后直接可用
- 部署新前端后告知用户 Ctrl+Shift+R 强刷（或等缓存自然过期）
- 服务器 `.env` 必须有 `JWT_SECRET`（签名密钥）和 `LEARN_AUTO_THRESHOLD=5`
- `users.yaml` 不提交 git，服务器需手动同步
- 服务器产生的反馈/规则变更需要 `git add -A && git commit && git push` 同步回来

### 踩坑清单（防止重蹈覆辙）
| 现象 | 根因 | 解法 |
|------|------|------|
| LLM rule_ref 输出格式混乱（5种） | prompt 指令歧义，LLM 有时照做有时发挥 | 固定值 + zero-shot 示例引导 |
| 前端规则链接跳空白页 | `<a href="/rules?...">` 没带 base path | 子路径部署所有手写 href 必须拼 `import.meta.env.BASE_URL` |
| 服务器 git pull 冲突 | rule_stats.yaml 服务器存在但未跟踪 | 新纳入 git 的文件需先在服务器 stash/rm 再 pull |
| 部署后前端白屏 | 浏览器缓存旧 JS hash 文件名 | 更新前端后通知用户 Ctrl+Shift+R |
| 标注图序号和列表不一致 | 旧版代码生成的图（重排在标注之后） | 新版已修复；旧任务需从 checkpoint 重跑标注 |
| 视觉路把文本截断误判为"图文不符" | LLM 看到文本块不完整就乱报 | prompt 明确"文本块不完整可能是提取截断，不是图片问题" |

---

## 下一步

### 短期
- 跑一次新审查验证 prompt 改版后的输出质量（rule_ref 是否统一为"模型自主发现"，视觉路是否不再乱报文本截断）
- 补中国市场法规规则（CCC / 中国RoHS / 制造商信息，needs_ref=false）
- 视觉仲裁 JSON 英寸符号容错重试

### 中期
- 用户账号体系 — 第二步：飞书 OAuth 登录（替换用户名密码）
- 多租户隔离（按部门）：任务/反馈/规则库按部门隔离
- 视觉审查升级（图片裁剪/prompt 精简）
- 文档结构检查归位到视觉路

---

## 工作流

- **本地开发** → `git push` → GitLab
- **服务器更新** → `git pull` + 重启（+ 告知用户强刷浏览器）
- **服务器产生规则/反馈** → `git add -A && git commit -m "sync: ..." && git push`（或 `./sync_rules.sh`）
- **本地同步** → `git pull`
