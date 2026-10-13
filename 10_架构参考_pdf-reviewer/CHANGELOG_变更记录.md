# 变更日志

## 🤖 Agent 阅读指南

线性变更历史，最新在上。了解"改了什么+为什么"时查阅此文件。
禁止在此写当前进度（那是 SESSION_SUMMARY 的职责）。

---

## 2026-07-17

### 视觉 prompt 防幻觉 + 前端体验修复 + 解密默认开

- [修复] **视觉 content.txt 防幻觉约束** — 加强"文本块看起来不完整可能是提取截断，不代表图片有问题"的约束，禁止因文本块截断判定"图文不符"
- [修复] **管理页 TypeBadge 英文标签改中文** — exclusion→排除模式，avoidance→新增反向规则，new_rule→新增规则，modification→规则修改
- [修改] **解密默认开启** — WorkspacePage decrypt 默认从 false 改为 true（大多数用户上传公司加密 PDF）
- [新增] `tests/` 目录 + 加入 .gitignore（本地测试样本和服务器下载产物，不提交）
- [原因] 视觉路 LLM 看到文本上下文中"不完整的句子"就误判图文不符；用户分不清英文 type 标签含义；加密 PDF 是常态
- [影响] pdf_review/prompts/vision/content.txt, pdf-review-web/src/pages/(AdminPage.tsx, WorkspacePage.tsx), .gitignore
- [关联] 视觉误报率降低；管理页体验优化

---

## 2026-07-16 (第二次更新)

### Prompt 统一输出 schema + 前端 UI 大改 + 反馈提示优化 + 服务器部署

- [重构] **Prompt 输出 schema 统一** — 文本路(7路)+视觉路(3路)统一字段: `id, page, block_id, original, issue_type, suggestion, rule_ref, rule_id`。视觉路去掉 `description`，合并到 `suggestion`（"分析+建议"一体）；视觉仲裁 prompt 同步对齐
- [重构] **rule_ref 字段语义规范** — 有规则时填核心审查要点（不带"规则N："编号前缀）；无规则时固定填"模型自主发现"。通过 zero-shot 示例引导 LLM 格式一致性
- [重构] `_backfill_rule_id` 兜底值从 `"逻辑推测"` 改为 `"模型自主发现"`
- [重构] **前端字号全面提升** — 全站只用 text-lg(18px)/text-xl(20px)/text-2xl(24px) 三级字号，去掉 text-xs/text-sm/text-base
- [修复] **前端规则链接跳空白页** — `ResultPage.tsx` 的 `<a href="/rules?...">` 未带 base path，改用 `import.meta.env.BASE_URL` 拼接
- [重构] **前端规则展示逻辑** — 有规则时显示 `规则 N: {要点}`（灰色文本，无超链接）；无规则时显示 `模型自主发现`（灰色标签）。前端正则兼容老数据（strip "规则 N："前缀、替换"逻辑推测"）
- [重构] **反馈提交提示优化** — 去掉 `_learn_hint()` 和 `FeedbackResponse.learn_hint` 字段，改为 `feedback_count`。前端提交后显示"提交成功，本 PDF 共积累提交了 X 个反馈"
- [修改] `LEARN_AUTO_THRESHOLD` 从 0(关闭) 改为 5（累积 5 个新反馈文件自动触发 learn 分析）
- [修改] 前端 description 兼容：老数据有 description 时拼到 suggestion 蓝色框展示；新数据无此字段不影响
- [部署] 服务器 git pull + .env 补 LEARN_AUTO_THRESHOLD=5 + 重启成功
- [原因] 规则展示不一致（LLM 输出 rule_ref 格式混乱 5 种幻觉）；前端字号过小（text-xs=12px 用户反馈看不清）；反馈提示无用（learn 已自动化不需要提示手动跑）
- [影响] pdf_review/prompts/(text_review/_output_suffix.txt, vision/content.txt, vision/layout.txt, vision/readability.txt, arbitration/vision.txt), pdf_review/pipeline_v4.py, pdf_review/webapi/(app.py, schemas.py), pdf-review-web/src/(全部页面+组件), .env
- [关联] prompt 质量提升；前端体验优化；自动 learn 阈值启用

## 2026-07-16

### 反馈自动触发规则学习分析（可配置阈值）

- [新增] `WebSettings.learn_auto_threshold`（环境变量 `LEARN_AUTO_THRESHOLD`，默认 0=关闭）— 累计新反馈达到该数量时，后台异步触发一次 `learn` 分析，生成 `suggestion_*.json`（pending 状态），**不自动落库**，仍需管理员在 `/admin/suggestions` 审批
- [重构] `pdf_review/scripts/learn_rules.py` 新增 `run_learn_analysis()` 统一入口，供 CLI `learn` 命令与 webapi 自动触发共用
- [新增] `pdf_review/webapi/app.py` 新增 `_count_new_feedback_files()`/`_maybe_trigger_auto_learn()`，`_learn_hint()` 复用同一份新增反馈计数逻辑；模块级 `_AUTO_LEARN_RUNNING` 标记防止并发重复触发
- [修复] **`submit_feedback` 接口 bug**：该接口原为同步函数(`def`)，FastAPI 会将同步接口丢进线程池执行，线程池内无运行中的事件循环，导致新增的 `asyncio.create_task()` 抛出 `RuntimeError: no running event loop`，会使所有反馈提交请求报 500。改为 `async def` 后接口跑在主事件循环，问题解决。此 bug 是本次新功能测试中发现，非用户预先报告
- [原因] 用户提出"反馈积累到阈值能否自动跑 learn"，讨论后确认"生成建议"本身无害可逆，可以自动化，但"落库改规则"仍必须保留人工审批关卡
- [影响] pdf_review/webapi/(app.py, settings.py), pdf_review/scripts/learn_rules.py, .env.example
- [关联] 规则学习闭环的自动化第一步；落库审批环节未变

## 2026-07-15 (第四次更新)

### 任务存储按部门分区 + 老任务迁移 + 路径 bug 全面排查修复

- [新增] `JobRecord` 新增 `owner_username`（稳定外键，对应 users.yaml 的 username）+ `owner_department`（创建时快照部门，不随人员调动变化）
- [重构] 任务物理存储从 `jobs/{job_id}/` 改为按部门分区 `jobs/{department}/{job_id}/`（`storage.job_dir()` 新增 `department` 参数）；`JobRegistry.recover()` 改为两层目录扫描
- [重构] `list_jobs`/`delete_job` 权限过滤改用 `owner_username` 精确外键，不再是 `uploader` 字符串软过滤
- [新增] `pdf_review/scripts/migrate_legacy_jobs.py` — 一次性迁移脚本：识别 `uploader="admin"` 的旧全局 Token 时代任务，回填 owner 字段 + 挪到部门分区路径 + 重写 job.json/checkpoint_*.json 内部硬编码的服务器绝对路径
- [迁移] 服务器同步下来的 6 个老任务已迁移到 `jobs/产品市场/`，全部关联到 `adminAI`；另发现 9 个 my-agent 独立前遗留在 `pdf_review/jobs/` 的更老任务，确认无需保留后已删除
- [修复] **P1 根因 bug**：`WebSettings.jobs_dir`/`rules_backup_dir` 默认值曾用相对路径 `./jobs`/`./rules_backups`，解析结果依赖启动时的 cwd；从 `pdf_review/` 子目录用 `dev_start.*` 启动会导致数据分裂到错误路径（这是 9 个老任务出现在 `pdf_review/jobs/` 的根因）。修复为固定路径：`jobs_dir` 固定相对项目根，`rules_backup_dir` 固定相对 `pdf_review/`（备份对象 `pdf_review/rules/data/` 本身在包内，备份挨着被备份对象更合理）
- [修复] **P2**：`rule_store.py` 的 `root_dir()` 假设存在根级 `my-agent/rules/` 副本（`parents[2]` 推算），该假设在项目独立后已失效（对应目录不存在）。改为单副本设计，删除失效的双副本同步逻辑
- [修复] **P3**：`sync_rules.sh` 引用不存在的根级 `pdf_review/rules_backups/` 路径（原本在项目根，后确认应在 `pdf_review/` 内），已同步修正并纳入规则备份到同步范围
- [修复] **P4**：`.gitignore` 缺失多条规则（曾错误排除 `pdf_review/jobs/` 而非实际使用的项目根 `jobs/`）；补全后确认 `rules_backups/`、`pdf_review/frontend/`（前端构建产物）需要纳入 git 同步（跟规则/反馈走同一套本地↔服务器同步工作流），只排除 `jobs/`（用户任务数据，量大不同步）和前端依赖/中间产物
- [修复] **P5**：`pdf_review/frontend/` 曾被 git 跟踪但内容因本地测试打包而变化，已确认应保留跟踪并更新索引
- [删除] `pdf_review/dev_start.bat`/`.ps1` — 这两个脚本依赖不存在的 `pdf_review/.venv`，且是触发 P1 数据分裂的直接原因；README 改为统一推荐"在项目根运行 `uv run python -m pdf_review.webapi`"
- [删除] 07-06 的规则备份快照 `pre_refine_20260706_111809`（缺 `meta.json`，非标准格式，无法通过现有回滚接口使用，确认无需保留）
- [原因] 用户询问"任务存的哪"引出 cwd 依赖的路径设计缺陷排查；借此机会梳理清楚"运行时任务数据 vs 知识资产(规则/备份/反馈)"的存储边界，为后续多租户/部门隔离规划打基础
- [影响] pdf_review/webapi/(storage.py, jobs.py, runner.py, app.py, settings.py, rule_store.py), pdf_review/scripts/migrate_legacy_jobs.py(新增), pdf_review/dev_start.bat/.ps1(删除), .gitignore, sync_rules.sh, README.md
- [关联] 账号体系第一步的后续加固；为多租户部门隔离规划提供了部门分区的存储基础

## 2026-07-15 (第三次更新)

### 用户账号体系（第一步：用户名+密码 + JWT）

- [新增] `pdf_review/webapi/auth.py` — 用户认证模块：`UserStore`（读写 `users.yaml`）+ 密码哈希（PBKDF2-SHA256，`salt$hash` 格式）+ JWT 签发/验证（PyJWT，HS256，不设过期时间）
- [新增] `users.yaml`（项目根，不提交 git）— 用户账号库，字段：`username`/`password_hash`/`display_name`/`department`/`role`/`created_at`
- [新增] 认证接口：`POST /api/v1/auth/login`（登录换 JWT）、`GET /api/v1/auth/me`、`GET/POST /api/v1/auth/users`（管理员）、`PUT /api/v1/auth/users/{username}/password`（管理员改密）、`DELETE /api/v1/auth/users/{username}`（管理员删除）
- [重构] `pdf_review/webapi/security.py` — 从两级静态 Token 鉴权改为 JWT 鉴权，`require_user`/`require_admin` 返回 `CurrentUser`（含 username/display_name/department/role）
- [重构] `pdf_review/webapi/app.py` — 所有路由的 `role: str` 依赖改为 `user: CurrentUser`；`POST /jobs` 不再需要客户端传 `uploader`，服务端从 JWT 自动填充（取 `display_name` 或 `username`）
- [废弃] 旧全局 Token（`PDF_REVIEW_USER_TOKEN`/`PDF_REVIEW_ADMIN_TOKEN`）不再生效，`settings.py` 保留字段避免旧 `.env` 报错
- [新增] `pyjwt>=2.8.0` 依赖；`.env` 新增 `JWT_SECRET`（签名密钥）
- [新增] 前端登录页改为用户名+密码表单（`LoginPage.tsx`），`api.ts` 新增 `login()`/`getCurrentUser()`/`setCurrentUser()`；`Layout.tsx`/`WorkspacePage.tsx` 改用 JWT 返回的用户信息，移除 uploader 客户端传参
- [修复] `RulesPage.tsx` 顺手修复 `noUncheckedIndexedAccess` 暴露的历史遗留类型错误（正则捕获组/数组下标未判空）
- [新增] 预置 19 个账号：1 个管理员（`adminAI`）+ 12 个产品线账号（`chan-pin-1`~`12`）+ 6 个实名账号（用户名密码均为拼音），全部属于"产品市场"部门；明文密码记录在 `账号密码表.md`（不提交 git，仅供分发）
- [原因] 原全局 Token 方案无法区分具体用户，无法做审计和权限细分；用户提出需要真实账号 + 未来多部门隔离规划
- [影响] pdf_review/webapi/(auth.py 新增, security.py, app.py, settings.py), pyproject.toml, .env.example, users.yaml(新增), pdf-review-web/src/(lib/api.ts, pages/LoginPage.tsx, pages/RulesPage.tsx, components/Layout.tsx, pages/WorkspacePage.tsx), docs/web-api.md, README.md
- [关联] 账号体系第一步（用户名+密码）；后续规划：飞书 OAuth 登录 + 按部门多租户隔离（任务/反馈/规则库）

## 2026-07-15
- [新增] **ProductAgentPlatform 接入** — 注册智能体 `pdf_reviewer` 到内部运营平台，审查完成自动上报 usage 事件（benefit_minutes = 页数×8）+ 首次用户 register 事件
- [新增] `pdf_review/webapi/platform.py` — 平台上报封装模块，失败只打日志不影响业务，不配 PLATFORM_API_KEY 则完全不激活
- [新增] `scripts/register_platform.py` — 一次性注册脚本，兼容平台 HTTP 200 + body code 201 返回方式
- [修改] `pdf_review/webapi/runner.py` — 审查完成回调中埋点上报
- [修改] `.env.example` — 补充 PLATFORM_API_KEY / PLATFORM_AGENT_ID 配置说明
- [新增] `DOCS.md` — 项目文档路由（必读+按需加载+目录约定）
- [原因] 接入内部智能体运营看板，统计用户数/使用次数/节省工时
- [影响] pdf_review/webapi/(runner.py, platform.py), scripts/register_platform.py, .env.example, DOCS.md
- [关联] 运营接入

## 2026-07-09
- [重构] **项目独立化** — pdf_review 从 my-agent 独立为完整项目 `D:\AAAAAAAA\projects\PDF`，前后端 monorepo 结构，force push 到 GitLab pdf-reviewer.git
- [新增] 前端源码 `pdf-review-web/` 纳入项目，与后端统一管理
- [新增] 项目文档体系: `docs/architecture.md`(流水线架构), `docs/rules-system.md`(规则系统), `docs/web-api.md`(API契约), `docs/model-api.md`(模型调用), `docs/paddleocr-api.md`(OCR), `CHANGELOG.md`, `SESSION_SUMMARY.md`
- [新增] `sync_rules.sh` 服务器端规则/反馈变更推回 GitLab（含 pull rebase + rule_stats force add）
- [修复] **子路径访问统一** — 前端 `base=/pdf-reviewer/`，后端 strip prefix 中间件，本地和 nginx 行为一致；加根路径重定向（`/` → `/pdf-reviewer/`）
- [修复] **重定向死循环** — 根路径重定向与 nginx strip prefix 冲突；改为中间件内检测 X-Forwarded-For 头区分代理/直接访问
- [改进] README 更新: 服务器部署（SSH 别名 `git@gitlab-yudaixun`）、本地开发入口 `/pdf-reviewer/`、规则同步流程
- [原因] pdf_review 已零依赖 my-agent，独立项目便于单独版本管理和部署
- [影响] 项目结构(整体), pdf_review/webapi/app.py, pdf-review-web/vite.config.ts, README.md, sync_rules.sh(新增), .gitignore(新增)
- [关联] 项目独立化完成，进入独立维护阶段

## 2026-07-08
- [修复] **rule_id 报告展示自相矛盾** — 旧版当 rule_id=0 时既输出"规则内容"又标"规则编号: 逻辑推测"，读起来矛盾。改为: 有规则(rule_id>0)显示"规则依据"+"规则编号: 规则N"；无规则(rule_id=0)显示"判断依据"+"来源: 大模型推理（无对应规则）"。详情报告内联格式同步为 [规则N: xx] / [模型推理: xx]
- [重构] **rule_id 回填机制升级 (方案A: source_ids 精确回填)** — 根因: 仲裁 LLM 不输出 rule_id，旧版 `_backfill_rule_id` 仅靠 page+original 文本模糊匹配回溯，仲裁改写空格/标点即失配(如尺寸问题 rule_id=34 丢成0)。新方案: 仲裁输入每条打稳定 `_uid`+`[#N]` 编号，prompt 要求输出 `source_ids` 数组，回填优先用 source_ids 精确匹配(合并多条时优先取带明确 rule_id 的来源)，失败降级文本匹配。文本仲裁+视觉仲裁均改造
- [修复] **α-文档结构路系统性误报根治 (方案1+2)** — 根因: 纯文本路承担了"目录页码核对/标题层级覆盖/排版顺序"等空间检查，模型把"文本块线性顺序"误当"页面视觉顺序"臆测，每轮产生9-10条目录页码噪音+标题层级/排版顺序误报。方案1(prompt收窄): document_identity_structure.txt 删除"父级标题覆盖"职责，明确"文本块顺序≠视觉位置，禁止据此判断版面空间问题"，目录页码改为"仅明确矛盾才报，不确定不报"。方案2(反向规则): 新增 avoidance/document_identity_structure.yaml (AV-021 文本块顺序不代表视觉顺序 / AV-022 页码无法确定不臆测)
- [验证] 错误样本A实测: 文档结构路 10条→0条误报，报告总数 19→8，尺寸问题 rule_id=34 正确回填，真实问题(错别字/尺寸格式/DTS声明/中国RoHS)全部保留
- [排查] CCC/中国RoHS rule_id=0 非bug: 规则库对中国市场法规认证覆盖不足(规则12/17均 needs_ref=true 无补充信息时被过滤，规则33是东芝专属)，缺 CCC/中国RoHS/制造商信息等 needs_ref=false 规则。模型推理检出正确但无对应规则(暂不补)
- [改进] **打包脚本纳入学习数据** — pack_module.py 去除 EXCLUDE_DATA_DIRS，data/feedback(12个)+data/suggestions(5个)随zip传输，远程可续跑learn闭环
- [新增] **initialize_from_index() 自动探测规则数** — stats.py 新增函数，从index.yaml自动获取max_rule_id初始化统计（幂等），不再硬编码115
- [修复] **setup.sh webapi 依赖缺失** — `uv sync` 改为 `uv sync --extra webapi`，确保fastapi/uvicorn被安装；setup.sh集成规则统计自动初始化
- [重构] **前端子路径部署改造 (/pdf-reviewer/)** — vite.config.ts base按build/dev区分(`/pdf-reviewer/` vs `/`)；main.tsx BrowserRouter加basename；api.ts BASE改用import.meta.env.BASE_URL拼接；LoginPage+RulesPage裸fetch改走API_BASE。构建产物资源引用正确带/pdf-reviewer/前缀
- [部署] **v4.1.0 远程部署** — 打包(164文件/7.9MB)→上传→uv sync --extra webapi→服务启动成功(0.0.0.0:8000)；nginx反代配置: location /pdf-reviewer/ proxy_pass + client_max_body_size 60m + proxy_read_timeout 300s
- [原因] rule_id 结构化数据在仲裁环节被丢弃再靠NLP匹配抢救(架构脆弱); 文本路能力边界与检查项错配; 前端根路径硬编码不兼容nginx子路径
- [影响] pdf_review/(pipeline_v4/rules/stats/setup.sh), prompts/arbitration/{text,vision}.txt, prompts/text_review/document_identity_structure.txt, rules/avoidance/{index,document_identity_structure}.yaml, scripts/pack_module.py, pdf-review-web/(vite.config.ts/main.tsx/api.ts/LoginPage/RulesPage)
- [关联] 审查精度提升, CCC排查闭环, P3.5部署上线

## 2026-07-07
- [重构] **Prompt 拆分到独立文件** — 新建 `pdf_review/prompts/` 目录(text_review/7 + vision/3 + arbitration/2 + settings.py + meta_detect.txt)，pipeline_v4.py 减少~150行，prompt 改动不碰代码
- [重构] **视觉 prompt 优化(方案A+B)** — blocks_info 角色反转为"误报抑制器"(图中文字看似不全但提取完整=不是裁切); context 从3页全文缩为前后页200字摘要+去掉当前页重复; 效果: 视觉误报 5→0条
- [重构] **元数据 LLM 增强** — enhance_meta_with_llm(快模型独立检测+代码结果合并); OCR完成后串行调用; 合并策略: 代码有确定值信代码(brand/market/language)，doc_type/model_name/summary 信LLM
- [新增] **元数据注入审查 prompt** — user_msg 开头注入[文档信息](brand/market/language/型号/描述); 消除品牌误报(海信当其他品牌5条→0)和市场误报(给国内文档要欧盟合规→0)
- [新增] Prompt 字段长度约束(所有prompt): suggestion≤150字/description≤100字/rule_ref≤80字 + "严禁在字段值中写推理过程"
- [新增] 截断 JSON 修复兜底: _parse_issues_json 策略3(剥离think标签)+策略4(找最后完整issue补全]}); 真实样本从0→2条恢复
- [新增] Trace 全落盘: 所有调用(含ok)均保存原文到 traces/，便于事后排查
- [新增] 依据追溯完整链路: _backfill_rule_id 无条件回填 rule_ref+rule_id; rule_id=0 统一标"逻辑推测"
- [修复] **WebAPI 规则读取适配 market 数组** — list_rules 从恒为空→正常命中; RuleOut 补充4个数组字段; export_rules_excel 新增4列
- [修复] **WebAPI 规则写操作数组化** — RuleWrite 5个字段改数组; rule_store reviewers 由 check_route 自动派生
- [修复] **前端 RulesPage 适配新8分类** — 分类下拉/卡片/详情/编辑表单全面更新; 多行字段解析截断 bug 修复
- [修复] 前端静态文件内置部署(webapi SPA fallback); 报告展示改"提出模型"+"规则内容"(去前缀)+"规则编号"
- [修复] 文档结构 prompt: 加"目录页码是印刷页码非PDF物理页号"避免目录误报
- [改进] 仲裁 prompt: 要求 rule_ref/rule_id 原样保留禁止改写
- [改进] 审查策略: "宁可漏报不误报" + rule_id 标注(有规则填编号，推理填0)
- [改进] 版本升至 v4.1.0; README 更新部署说明(nohup后台运行+前端内置+新打包流程)
- [原因] 规则库重建后 webapi/前端未跟进适配; parse_fail 根因是模型超长推理; 视觉/品牌/市场误报根因是缺少上下文信息
- [影响] pdf_review/(pipeline_v4/document_meta/trace/prompts/webapi/frontend), pdf-review-web/src
- [关联] P3.5 产品化增强, 审查精度提升

## 2026-07-06 (第二次更新)
- [修复] **WebAPI 规则读取适配 market 数组** — `rules.py` 的 `list_rules` market 筛选对数组恒为 True(永远返回 0 条)；改为数组包含匹配(对齐 loader.load_rules)
- [新增] `RuleOut` 补充 doc_type/product_scope/document_area/check_route 四个数组字段，前端可展示/筛选新维度
- [修复] `export_rules_excel` market 数组导出为逗号分隔；新增文档类型/产品范围/文档区域/检查路由四列
- [重构] **WebAPI 规则写操作数组化** — `RuleWrite` market/doc_type/product_scope/document_area/check_route 改为 `list[str]`；`rule_store` 新增 `_derive_reviewers` 从 check_route 自动派生 reviewers；`_render_md_block`/`create_rule`/`modify_rule` 全面支持数组字段+新 yaml 格式
- [修复] **前端规则解析多行字段截断** — `parseRuleText` 正则 lazy+multiline `$` 导致"审查规则"等多行字段只取第一行("错误：..."续行丢失)；改为迭代拆分按字段头位置截取
- [重构] **前端 RulesPage 适配新 8 分类** — 分类下拉/卡片标签/详情弹窗/编辑表单全面更新：8 新分类+7 语言选项+market 等 5 维数组多选标签组件
- [新增] **前端静态文件内置部署** — webapi `app.py` 末尾加 SPA fallback，检测 `pdf_review/frontend/` 存在时自动 serve；打包脚本产出含前端 dist，部署后无需单独跑前端
- [改进] 版本号升至 v4.1.0；README 更新部署说明(nohup 后台运行 + 前端内置 + 新打包流程)
- [原因] 规则库重建后 webapi 读写链路未跟进适配(market 数组/新分类/新维度)；前端正则 bug 导致规则内容展示不完整
- [影响] pdf_review/webapi (schemas/rules/rule_store/app), pdf-review-web/src (api.ts/RulesPage.tsx), pdf_review/pyproject.toml, pdf_review/README.md
- [关联] P3.5 产品化增强, 前端适配新规则体系

## 2026-07-06
- [重构] **规则库整体重建** — 用 ChatGPT 对话式从原始 Excel 重新生成 115 条规则，重新设计分类体系:
  - 8 个新分类替代旧 7 类: language_localization / product_specs_parameters / feature_configuration / legal_certification_compliance / brand_ip_terminology / document_identity_structure / safety_installation_accessory / visual_layout_assets
  - 新增多维数组标签: market / doc_type / product_scope / document_area / check_route（均为数组，一条规则可适用多值）
  - language 从 3 种扩展到 7 种 (all/zh/en/ja/tr/uk/multilingual_compare)
- [重构] **pipeline C-全** — 审查路由从"按检查动作(α4路/β4路)"改为"按规则分类(7文本分类)":
  - α(397b) 和 β(deepseek,可替换) 跑**相同的 7 个分类、用同一套 prompt**，实现全面交叉验证
  - `_run_text_alpha` + `_run_text_beta` 合并为通用 `_run_text_review(model_kind)`
  - 视觉路规则源从 layout_structure 改为 visual_layout_assets
  - `_load_all_rules` 按分类加载（不再按 reviewer），check_route→reviewers 兼容派生
- [修复] loader `load_rules` 支持数组 market 匹配（含 global/all 视为通用）
- [修复] DocumentMeta market/doc_type 检测对齐新代码，language 增加日文假名检测
- [修复] **断句误报** — 语言本地化 prompt 收窄，明确"PDF 提取的跨块换行不是断句错误"
- [修复] **视觉裁切/文字重叠误报** — 根因是喂给视觉模型的 blocks_info 内容被截断到 40 字，模型把"我们的截断"误判为"文档裁切"；改为给完整文本 + prompt 明确"以图片为准，文本块内容完整"
- [修复] 标注定位框只框前几个字 — locate.py 命中后扩展到整个文本块
- [修复] B4 复审 PDF 下载失败 — 改为 FileResponse 直接返回文件流
- [新增] 导入脚本 scripts/import_refined_rules.py + 导出脚本 export_rules_for_refine.py
- [新增] 规则重建指导文档 docs/RULE_REFINE_GUIDE.md
- [新增] 前端 P3.5: F2序号醒目/F3翻页增强/F4暂存统一提交/F5复审PDF/F7评审变色 + B2导出Excel/B3视觉开关/F6开关/F8导出 + 换设备从后端恢复反馈(GET /jobs/{id}/feedback)
- [原因] 原规则库从 Excel 脚本导入丢失大量分类/语言/市场标签，质量参差；旧 pipeline 分路与新分类不匹配
- [影响] pdf_review/ (pipeline_v4/loader/document_meta/rules数据/webapi/scripts), pdf-review-web/ (ResultPage/WorkspacePage/RulesPage/api)
- [关联] P1.5 规则精炼(6/7), P3.5 产品化增强
- [遗留] parse_fail 偶发（超长输出 JSON 截断）待系统定位

## 2026-07-03
- [新增] 用户补充信息注入功能 — context_parser.py + Excel 模板 + CLI --context + WebAPI context_file + 各审查路精准注入
- [重构] feature_matching 26 条规则拆分归入现有 7 类,取消该类别; 视觉路 layout_structure 从 3→6 条
- [新增] needs_ref 条件加载 — 有 context_file 时加载 needs_ref=True 规则(+28%)
- [新增] 回滚自动重置建议状态(approved/rejected→pending) + reopen 端点
- [新增] 前端: 规则编辑 UI(新建/修改/删除) + 登录账号选择 + 结果页筛选 + 规则库分页 + 文件名搜索
- [修复] user token 登录 403 死循环 (403 不再清 token)
- [原因] 补充信息显著提升审查精度(50%问题引用真值表);死规则激活;前端体验补全
- [影响] pdf_review/ (context_parser/pipeline_v4/config/webapi/rules), pdf-review-web/ (多页面)
- [关联] P1.5 审查精度提升, P3 产品化

## 2026-07-01
- [新增] 规则效果统计体系 (rules/stats.py + rule_stats.yaml)，追踪每条规则的触发次数和人工采纳率
- [新增] LLM 审查输出 rule_id 字段，实现问题→规则的精确关联 (prompt 改造 + 仲裁后回填)
- [新增] 独立反向规则库 (rules/avoidance/)，16 条误报防护规则，按 category 标签加载
- [新增] Web API 端点 GET /avoidance，浏览反向规则，支持筛选
- [新增] RuleOut 返回统计字段 (origin/trigger_count/accepted_count/rejected_count/acceptance_rate)
- [新增] IssueOut 返回 rule_id 字段
- [新增] 前端测试项目 `pdf-review-web/` (Vite+React+TS+Tailwind)，覆盖核心链路: 登录/上传/进度/结果/标注图/反馈/规则浏览/统计
- [改进] learn 流程输出 avoidance_rules 类型建议，approve 可落库到反向规则库
- [改进] 反馈提交时自动更新规则统计 (Web + 命令行两端)
- [改进] webapi `__main__.py` 自动加载 `.env` 文件 (`_load_env()`)，消除手动设环境变量依赖
- [改进] 打包脚本 `start.sh` 加入 ROOT_FILES，部署后 zip 根一键启动
- [改进] `.env.example` 补充 Web API 配置模板 (Token + CORS + 任务参数)
- [废弃] exclusions.yaml 内容迁入 avoidance/text_exclusions.yaml，pipeline 不再加载旧文件
- [修复] `get_page` 端点对 annotated_images 稀疏列表的页码匹配 — 改为 `page_{:03d}_annotated` 精确匹配 + fallback 原始 PNG
- [修复] learn 的 f-string 中 `{category}` 未转义导致 NameError
- [改进] 打包脚本排除 rule_stats.yaml (运行时数据，服务器自行初始化)
- [文档] WEB_API.md 补充 avoidance 端点、rule_id、统计字段、get_page 描述更新、启动方式更新
- [文档] README.md 重写 Web 服务章节 (统一 .env + `uv sync --extra webapi` + start.sh + 前端指路)
- [文档] PDF_REVIEW_PIPELINE.md 更新包结构 (avoidance/ + stats.py + webapi/)
- [原因] P1 规则系统闭环 + P3 产品化前端联调
- [影响] pdf_review/webapi/__main__.py, webapi/app.py, scripts/pack_module.py, pdf_review/start.sh(新), pdf-review-web/(新项目)
- [关联] P1 规则系统闭环 + P3 产品化前端

## 2026-06-30

### PDF 审查 Web 服务后端实现 (P1+P2 + 进度回调 + 取消)

- [新增] `pdf_review/webapi/` 子包(FastAPI):settings/security/schemas/storage/jobs/runner/rules/rule_store/suggestions/app/__main__。依赖单向(webapi→核心),FastAPI 作 pdf_review 可选依赖组 `[webapi]`
- [新增] P1 接口:健康检查、任务全流程(上传/列表/状态/结果/页面图/标注PDF/报告/删除)、反馈裁决、规则只读浏览、排除名单、反馈统计
- [新增] P2 接口(管理员):规则建议审批落库、规则在线增删改(单操作入口)、备份/回滚;两级 Token 鉴权 + uploader 软过滤 + 并发信号量 + 重启恢复 + 每次规则写操作前自动全量备份(两副本同步)
- [修改] `pipeline_v4.py` `run_pipeline` 增加可选 `on_event` 进度回调(阶段边界发射事件,verbose 不变、向后兼容)
- [修改] `scripts/learn_rules.py` 额外输出结构化 `suggestion_{ts}.json`(带稳定 sid + pending 状态),供网页审批;命令行行为不变
- [修改] DELETE 支持取消运行中/排队任务(JobRunner 跟踪 task + cancel,正确处理 CancelledError)
- [修复] 上传者中文名改用查询参数(HTTP 头不能带非 ASCII);建议 sid 分隔符 `#`→`_`(URL 路由);规则备份 ID 加微秒(防同秒撞名)
- [文档] `MODULES/WEB_API.md` 更新至与实现一致 + 附录 D 前端对接注意事项;`README.md` 加 Web 服务入口与指路
- [原因] P3 产品化 — 把审查能力做成多人可用的 HTTP 服务
- [影响] pdf_review/webapi/(新增), pdf_review/pipeline_v4.py, pdf_review/scripts/learn_rules.py, pdf_review/pyproject.toml, pyproject.toml, MODULES/WEB_API.md, README.md
- [关联] P3 产品化(后端完成,待前端)

## 2026-06-29

### P0-1 修复视觉仲裁 JSON 裸引号 parse_fail + Web 服务接口契约

- [修复] `pipeline_v4.py` `_parse_issues_json`:新增 `_repair_unescaped_quotes` 上下文扫描器,修复 LLM 输出 JSON 字符串值中未转义的裸双引号(英寸符号 `85"`、中文弯引号后裸引号 `"85”"` 两种真实畸形)。两段式解析:先按原文 `json.loads`,失败才走修复重试,故对正常 JSON 零副作用
- [验证] 单元测试 8/8 + 两个真实捕获样本(视觉仲裁/视觉审查 parse_fail)2/2;端到端重跑同一 PDF,trace 从 `parse_fail=1` 变为 `empty=8|ok=11`,归零
- [新增] `MODULES/WEB_API.md` — PDF 审查 Web 服务完整接口契约(设计定稿,未实现):21 个接口、双 Token 角色、任务异步轮询、问题裁决反馈、规则浏览/建议审批/在线编辑+自动备份;并定义两处核心包待改点(learn 结构化输出、run_pipeline on_event 回调)
- [原因] trace 暴露的视觉 JSON parse_fail 真 bug(P0-1)+ 启动 P3 前端对接的接口设计
- [影响] pdf_review/pipeline_v4.py, MODULES/WEB_API.md(新增)
- [关联] P0-1(完成);P3 产品化(设计阶段)

### 标注定位统一重构 + 序号/红框修复 + 调用级 trace 体系

- [重构] 标注定位抽为独立的、与输出格式无关的统一定位器 `tools/locate.py`；6 层策略归一到 PDF points 坐标系，算一次供 PNG 渲染与 PDF 批注两条路径共用
- [重构] `_annotate_all` 瘦身为纯 PNG 渲染器（只读 `_bbox_pt` 画框）；`pdf_export.py` 两条路径改为直接用 `_bbox_pt`，删除各自的重复搜索与 `_pixel_x/_pixel_y`
- [修复] 审查问题序号改为定位后按阅读顺序（页码→纵向→横向）重排重新编号（新增 `_reorder_issues`）
- [修复] 红框对带前导项目符号的中文句子误匹配成竖条 — locate 搜索剥离前导符号、去掉首词兜底、加窄框防护
- [修复] `_annotate_on_source_pdf` 中 `doc.close()` 后仍读 `len(doc)` 的 use-after-close
- [新增] 调用级链路追踪 `trace.py`（CallTrace/RunTrace/RunTracer + traced_call）；6 类外部调用全接入，记录状态/耗时/原始返回/解析条数
- [新增] trace 三层产物：控制台每调用一行 + `{pdf}_trace_{ts}.json` + 详情报告 trace 表 + `traces/` 失败原文落盘
- [修改] 视觉工具 `analyze_image_async` 增加 `raise_on_error`（v4 传 True 触发重试+trace 分类，v3 默认 False 兼容旧行为）
- [修改] 文本/视觉仲裁兜底收紧：仅 error/timeout/parse_fail 时回退保留全部，合法空结果保持空
- [原因] 两套定位逻辑割裂难维护 + 序号乱序 + 红框偏移 + LLM 调用全程无可观测性（β=0 无法诊断）
- [影响] pdf_review/pipeline_v4.py, pdf_review/trace.py(新增), pdf_review/tools/locate.py(新增), pdf_review/tools/pdf_export.py, pdf_review/tools/vision.py
- [关联] P0 标注定位（完成）；新发现视觉仲裁 JSON 未转义引号导致 parse_fail（待修）

## 2026-06-26

- [优化] 文本仲裁改为纯去重（不再删除任何非重复问题）
- [优化] 视觉仲裁收紧规则（严格删除看错/正常设计/模糊/主观建议的问题）
- [新增] 视觉审查加载 layout_structure 规则
- [修复] 文本仲裁传入文档原文（之前模型无法对照验证）
- [修复] 视觉审查 block 列表来源改为 prefer_pdf 优先
- [修复] 视觉 prompt 要求输出 original + block_id（用于标注定位）
- [修复] 视觉仲裁传参增加 block_id 和 original
- [重构] 标注定位改为 6 层策略（pymupdf→block_id→关键词→图片→最大图→文本中心）
- [新增] Git 同步方案：pdf_deploy + pdf_sync_to_repo → 内网 GitLab
- [新增] 无密码 SSH 密钥配置
- [修改] README 更新为远程部署版本
- [修改] 打包脚本排除运行时数据（feedback/suggestions/batch_results）
- [原因] 文本仲裁过度删除合理问题 + 视觉标注全部落空
- [影响] pdf_review/pipeline_v4.py, scripts/pdf_deploy.py, scripts/pdf_sync_to_repo.py, scripts/pack_module.py
- [关联] P0 视觉标注定位修复（部分完成）

## 2026-06-25

### 架构升级：PDF 审查流水线 v4 — 双模型交叉验证 + 规则学习系统

- [类型] 架构重构 + 新功能
- [原因] v3.1 的 A-global (qwen3-5-397b) 中文卡死 (thinking模式)；规则无差别加载浪费token；C终审负担过重；缺乏持续改进机制
- [影响] 新建 `pipeline_v4.py`、`document_meta.py`、`scripts/parse_feedback.py`、`scripts/learn_rules.py`、`scripts/batch_test.py`、`rules/data/exclusions.yaml`；修改 `__main__.py`、`__init__.py`、`config.py`、`rules/loader.py`、`tools/pdf_parse.py`、`scripts/pack_module.py`
- [关联] MODULES/PDF_REVIEW_PIPELINE.md

#### 具体变更：

1. **DocumentMeta 预处理检测** — 自动识别 language/market/brand/doc_type/is_outlined/model_name/page_scale
   - 规则按元数据精确筛选，中文国内 PDF 规则减少 68%

2. **关闭 thinking 模式** — `chat_template_kwargs: {"enable_thinking": False}`
   - qwen3-5-397b 中文从卡死(>5min) 降到 2.4s

3. **v4 双模型交叉验证架构**
   - 文本审查α (qwen3-397b no_think): 一致性 + 逻辑 + 认证 + 市场 (4路并行)
   - 文本审查β (deepseek-flash): 拼写 + 数值 + 命名 + 一致性复核 (4路并行)
   - 视觉审查 (qwen3-vl): 版面 + 图文 + 内容可读性 (3路并行)
   - 合并分类: α∩β=高置信, 仅α=中, 仅β=低
   - 文本仲裁 (397b no_think): 去重+删误报+纠正分类，不加载规则
   - 视觉仲裁 (VL): 看图确认+去重+补充，不加载规则

4. **输出体系优化**
   - 统一输出文件夹 `{文件名}_{时间戳}/`
   - 切片图存入 `pages/` 子目录
   - 新增审查详情报告 (全过程记录)
   - 新增反馈模板自动生成
   - rule_ref 字段 (每个问题标注依据的规则描述)
   - 命令行输出美化 (序号+计时+分割线)

5. **规则学习系统 (5阶段)**
   - 阶段1: 反馈模板生成 + 解析 (`feedback` 命令)
   - 阶段2: 排除名单 `exclusions.yaml` (8条标准授权声明)
   - 阶段3: 经验积累统计 (`learn --stats`)
   - 阶段4: LLM 辅助规则提炼 (`learn`)
   - 阶段5: 自动学习提醒 (≥5条反馈或≥3条误报时提示)

6. **批量测试** — `batch` 命令，逐文件跑+汇总统计+保存结果JSON

7. **打包脚本重写** — 正确的zip结构 (根/配置 + pdf_review/代码)，不打包.env密钥

8. **性能优化**
   - 并发 Semaphore 5→15, OCR 3→5
   - 性能诊断: 瓶颈是文本模型(非视觉), VL实际0.3s/次很快

9. **保留 v3.1** — `--v3` 参数可切换旧版

---

## 2026-06-24 (第二次更新)

### 新功能：模块打包工具 + PDF 审查包移植优化

- [类型] 工具 + 适配
- [原因] 需要将 pdf_review 等子模块打包移植到其他机器独立运行
- [影响] 新建 `scripts/pack_module.py`；修改 `pdf_review/models.py`、新增 `pdf_review/__main__.py`、`pdf_review/setup.sh`、`pdf_review/pyproject.toml`
- [关联] MODULES/PDF_REVIEW_PIPELINE.md

#### 具体变更：

1. **pack_module.py** — 通用模块打包脚本
   - 自动读版本号、排除 `__pycache__`/`.pyc`
   - 无参数时列出所有可打包模块
   - 输出 `{模块名}_v{版本}.zip` 到项目根

2. **pdf_review/models.py** — 兼容 pydantic-ai v2
   - `OpenAIModel` → try `OpenAIChatModel` (v2) except fallback `OpenAIModel` (v1)

3. **pdf_review/__main__.py** — 简化运行入口
   - `python -m pdf_review "文件.pdf"` 替代原来的长命令
   - 支持 `--checkpoint` 参数走检查点模式

4. **pdf_review/setup.sh** — 目标端一键初始化
   - 自动整理散文件为标准项目布局
   - 自动 `uv sync` 安装依赖

5. **pdf_review/pyproject.toml** — 独立项目配置
   - scripts 入口指向 `__main__:_main`
   - 清华镜像源

---

## 2026-06-22 (第二次更新)

### 重构：PDF 审查流水线 v3.1 — 五路拆分 + 多项质量提升

- [类型] 架构优化 + 质量提升
- [原因] 提高审查准确度、减少漏报和误报、改善标注 PDF 质量
- [影响] `pdf_review/pipeline.py`, `pdf_review/tools/pdf_export.py`, `pdf_review/tools/vision.py`
- [关联] MODULES/PDF_REVIEW_PIPELINE.md

#### 具体变更：

1. **五路并行拆分** (原三路)
   - A-local 拆为 A-local-spelling(拼写/语法) + A-local-numeric(数值/格式)
   - B 拆为 B-layout(版面结构) + B-content(图文内容)
   - 各自有独立 prompt 和规则，模型注意力更聚焦

2. **滑动窗口** — A-local 窗口5页步长4(重叠1页)，捕获批次边界错误

3. **规则完整传入** — 去掉 4000/6000/1500/2000 字符截断，所有规则完整送入模型

4. **标注 PDF 方案 C** — 未转曲直接在原 PDF 加批注(红框+序号+高亮)，转曲用图片标注

5. **C 终审增强** — 新增纠正分类/建议的职责 + review_note 终审意见字段

6. **temperature=0** — 所有 LLM 调用统一设置，提高结果一致性

7. **统一重试逻辑** — `_retry_async` 最多2次指数退避，覆盖所有调用

8. **代码级去重** — `_dedup_issues` 过滤自我否定 + 原文包含关系去重

9. **标注 fallback** — 5层降级定位，最后兜底页面左上角序号

---

## 2026-06-22

### 重构：PDF 审查抽为独立包 `pdf_review/`

- [类型] 架构重构
- [原因] PDF 审查代码散落 7 个目录，需要移植到其他项目独立使用
- [影响] 新建 `pdf_review/` 独立包（config/models/pipeline/tools/rules/scripts）
- [关联] MODULES/PDF_REVIEW_PIPELINE.md

### Bug修复：视觉 API SSE 解析 + 标注 PDF 页码错位

- [类型] Bug修复
- [原因] 聚好看视觉 API 返回 `data:{...}` (无空格)，代码检查 `data: ` (有空格) 导致 B/C 返回空；标注 PDF 用 img_idx 当页码导致错位
- [影响] `pdf_review/tools/vision.py`, `pdf_review/tools/pdf_export.py`, `pdf_review/tools/annotate.py`
- [关联] MODULES/PDF_REVIEW_PIPELINE.md

### 新功能：decryptor 加密文件解密下载工具

- [类型] 新功能
- [原因] 需要上传加密文件到内网解密服务并下载明文
- [影响] 新建 `decryptor/` 目录 (decrypt.py + README.md)
- [关联] 独立工具，无 Phase 关联

### 性能优化：PDF 流水线异步并发化 + v3 架构

- [类型] 性能优化 + 架构重构
- [原因] 2页 PDF 需 3-4 分钟，瓶颈在逐页串行调 LLM API
- [影响] 全部节点 async 化，OCR/B/C 并发(Semaphore=5)，v3 三初审+一终审架构
- [关联] MODULES/PDF_REVIEW_PIPELINE.md

### 接口适配：切换到聚好看内部模型 API

- [类型] 接口适配
- [原因] 从 DashScope 切换到聚好看内部服务器
- [影响] `src/config.py`, `src/agents/model.py`, `src/tools/vision.py`, `.env`
- [关联] docs/聚好看api调用.md

---

## 2026-06-17

### PDF 流水线 v2 完整重构 + 规则系统

- [类型] 架构重构
- [原因] 不同类型错误由不同专长模型负责，规则库需分类分块
- [影响] `src/graph/pdf_review_pipeline.py` 7节点重写，`rules/` 目录新建 8 个规则文件 + index.yaml，`scripts/` 导入导出脚本
- [关联] MODULES/RULES_SYSTEM.md, MODULES/PDF_REVIEW_PIPELINE.md

---

## 2026-06-16

### 改进：OCR 结构化数据 + annotate 精确定位

- [类型] 改进
- [原因] annotate 命中率只有 80%（全角/半角差异），OCR API 实际有 bbox 坐标数据未利用
- [影响] `src/tools/ocr.py` 新增结构化接口，annotate 改为优先用 bbox 定位
- [关联] MODULES/PDF_REVIEW_PIPELINE.md

### 流水线 v2 重构 + 精细标注

- [类型] 架构重构
- [原因] 需要半并行架构 (A‖B → C) + 两层标注 (红框+黄色高亮)
- [影响] `src/graph/pdf_review_pipeline.py`, `src/tools/annotate.py`
- [关联] MODULES/PDF_REVIEW_PIPELINE.md

---

## 2026-06-15

### 新功能：Skill 技能机制 (Phase 6)

- [类型] 新功能
- [原因] 加新能力只有写工具/写SubAgent两条路，都要改代码；Skill 是丢文件即可
- [影响] 新建 `src/skills/`, `skills/weekly-report/`, 注册 `load_skill` 工具
- [关联] PHASES/PHASE6.md

### 新功能：pdf_reviewer 引入多模态 + OCR

- [类型] 新功能
- [原因] pdf_reviewer 需要看图能力和 OCR 能力
- [影响] `src/tools/vision.py`, `src/tools/ocr.py`, `src/tools/pdf.py`
- [关联] MODULES/PDF_REVIEW_PIPELINE.md

### 新功能：pdf_reviewer 子 Agent 框架搭建

- [类型] 新功能
- [原因] 需要审阅多语言带图 PDF 文档的子 Agent
- [影响] `src/tools/pdf.py` (parse_pdf), `src/agents/sub_registry.py`
- [关联] PHASES/PHASE6.md
