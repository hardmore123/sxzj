# 海外爬取项目 · 完整代码清单与功能说明

> 本文档由代码扫描自动汇总，覆盖以下五个目录的代码文件（>1KB），并按功能分类组织：
> 1. `海外/overseas/` — 核心爬虫代码包
> 2. `海外数据爬取_部署包/overseas/` — 部署代码（含框架副本 + 部署包专属模块）
> 3. `竞品对阵分析_部署包/overseas/` — 对阵分析服务
> 4. `compintel-deploy/overseas/compintel/` — 认证智能体
> 5. `海外/.kiro/skills/overseas-schema-gen/` — Skill 定义
>
> **统计口径**：仅列文件大小 >1KB 的代码文件；排除 `__pycache__`、`.git`、`crawl4ai`、`browser_profile` 等生成/缓存目录。
> **架构要点**：本项目采用「站点适配插件 + 声明式 schema」架构——每个站点一个 `sites/<code>/adapter.py`（继承 `SiteAdapter`）配一份 `schema.json`（选择器写这里），核心引擎自动发现注册。新增站点只需建目录，不改核心代码。
> **规模概览**：核心框架（海外/overseas）约 100+ 模块；站点适配器共 361 个、站点 schema 296 份；覆盖 871 个 >1KB 代码文件。

---

## 目录

- [一、爬虫核心代码](#一爬虫核心代码)
  - [1.1 框架基础设施层](#11-框架基础设施层)
  - [1.2 场景编排与 CLI 层](#12-场景编排与-cli-层)
  - [1.3 规格采集（SPEC）模块](#13-规格采集spec模块)
  - [1.4 零售线（价格+网评）模块](#14-零售线价格网评模块)
  - [1.5 翻译与评论导出模块](#15-翻译与评论导出模块)
  - [1.6 API 服务与前端](#16-api-服务与前端)
  - [1.7 墨西哥线（MX）模块](#17-墨西哥线mx模块)
  - [1.8 站点适配器层（sites/）](#18-站点适配器层sites)
  - [1.9 部署包专属模块](#19-部署包专属模块)
  - [1.10 LangGraph 全自动流程](#110-langgraph-全自动流程)
- [二、对阵分析服务代码](#二对阵分析服务代码)
- [三、认证智能体代码](#三认证智能体代码)
- [四、Skill 定义文件](#四skill-定义文件)
- [五、运维脚本](#五运维脚本)
- [六、前端文件](#六前端文件)
- [七、配置文件](#七配置文件)
- [附录：架构说明与统计](#附录架构说明与统计)

---

## 一、爬虫核心代码

### 1.1 框架基础设施层

| 文件路径 | 大小 | 功能说明 | 核心函数/类 |
|---|---|---|---|
| `海外/overseas/__init__.py` | 0.5KB | 框架包入口，说明分层架构（scenarios/sites/extract/fetchers/infra/db），核心只用标准库，Playwright 为可选依赖 | `__version__` |
| `海外/overseas/models.py` | 6.3KB | 领域数据模型，用 dataclass 定义 Product/SpecItem/PriceSnapshot/Review/ReviewSummary/RankingItem/ProductPayload/SearchHit 及日本 SPEC 的 SpecRow/SpecSheet；站点适配器统一产出这些结构 | `Product`, `PriceSnapshot`, `Review`, `ReviewSummary`, `ProductPayload`, `SpecSheet`, `SpecRow` |
| `海外/overseas/extract.py` | 20KB | 声明式抽取引擎，按站点 schema.json 的选择器链、键值表、正则兜底、JSON-LD 等规则从 DOM/HTML 抽取标题、价格、货币、评分、库存、品牌、型号、规格表与评价列表；内置归一化 transform | `class Extractor`, `apply_transform()`, `TRANSFORMS`, `jsonld_find_product()` |
| `海外/overseas/fetchers.py` | 46KB | 抓取层，统一暴露 Dom 接口：HttpFetcher 用标准库 urllib 抓静态/JSON 页，BrowserFetcher 用 Playwright 抓 JS 渲染/反爬页（复用 browser+context、持久化 profile、登录态、scroll/click 懒加载、网络响应捕获） | `class HttpFetcher`, `class BrowserFetcher`, `class PlaywrightDom`, `class LxmlDom`, `wait_for_login()` |
| `海外/overseas/infra.py` | 15KB | 抓取基础设施：按域名限速（最小间隔+随机抖动+429/503 指数退避）、反爬/登录墙识别、URL→响应正文磁盘缓存（带 TTL）、代理解析与 TCP 探活 | `class RateLimiter`, `class BlockDetector`, `class DiskCache`, `resolve_proxy()`, `login_wall_hit()`, `ProxyUnavailableError` |
| `海外/overseas/db.py` | 57KB | SQLite 存储层，对外只暴露 Database 一个类，全部 SQL 收于此；价格/评价摘要/排名均为只 INSERT 的时序快照，评价用 (product_id, review_key) 唯一键做增量去重；内置 schema 迁移 | `class Database`, `upsert_product()`, `add_reviews()`, `save_spec_sheet()`, `record_kakaku_status()`, `init_schema()` |
| `海外/overseas/config.py` | 13.7KB | 全局配置模块，所有参数以 OVERSEAS_ 前缀环境变量覆盖、默认值可直接本地跑通：路径、抓取节流、浏览器、代理、缓存、SQLite、翻译、AI、LLM 配置 | `llm_settings()`, `session_path()`, `profile_dir()`, `summary()`, `ensure_dirs()` |
| `海外/overseas/regions_config.py` | 17.5KB | 区域线集中配置，以纯数据 dataclass（Region/Brand/SpecSite/RetailSite）声明每条国家线的品牌、SPEC 官网站点与价格+网评零售站，替代散落的硬编码 | `dataclass Region`, `dataclass Brand`, `dataclass SpecSite`, `dataclass RetailSite`, `REGIONS`, `get_region()` |
| `海外/overseas/schema.sql` | 21.5KB | SQLite 建表 DDL，定义 product/price_snapshot/review/review_summary/spec_series/spec_row/retail_task_status/kakaku_status 等全部表结构与索引 | 无（声明式 DDL） |
| `海外/overseas/dict/__init__.py` | 8.8KB | 翻译词典加载入口，加载 spec_ja_zh.json / spec_es_zh.json 等离线词典供 SPEC 项目名翻译 | `load_dict()`, `translate()` |
| `海外/overseas/dict/spec_ja_zh.json` | 33KB | 日文→中文 SPEC 项目名离线词典（如 型番→型号、パネル→面板） | 无（数据文件） |
| `海外/overseas/dict/spec_es_zh.json` | 1.8KB | 西文→中文 SPEC 项目名离线词典（南美线用） | 无（数据文件） |

### 1.2 场景编排与 CLI 层

| 文件路径 | 大小 | 功能说明 | 核心函数/类 |
|---|---|---|---|
| `海外/overseas/scenarios.py` | 64KB | 采集场景编排模块，定义 S1 关键词搜索、S2 型号定向监控、S3 详情直采、S4 评价增量追踪四大零售场景，以及 SPEC 品牌官网规格周度采集（含多级系列抓取与品牌熔断）；统一处理浏览器/HTTP 双路径、详情抽取、payload 落库与审计 | `class RunStats`, `make_fetcher()`, `open_dom()`, `collect_detail()`, `save_payload()`, `s1_keyword_search()`, `s2_model_monitor()`, `s3_detail()`, `s4_review_incremental()`, `spec_crawl()` |
| `海外/overseas/cli.py` | 58KB | 命令行入口模块，用 argparse 暴露 init-db/sites/search/detail/monitor/reviews/probe/login/spec-crawl/spec-onboard/spec-export/retail-health 等子命令，并把竞品情报（FCC/EPREL/ENERGY STAR/ANATEL/IC-ISED/TÜV）抓取与预警分析也接到同一 CLI | `build_parser()`, `cmd_search()`, `cmd_monitor()`, `cmd_spec_onboard()`, `cmd_spec_crawl()`, `cmd_login()`, `cmd_compintel_fcc_full()`, `cmd_compintel_run()` |

### 1.3 规格采集（SPEC）模块

| 文件路径 | 大小 | 功能说明 | 核心函数/类 |
|---|---|---|---|
| `海外/overseas/generic_spec.py` | 36KB | 配置驱动的通用 SPEC 适配器（阶段二 POC），由一份 AdapterSpec JSON 驱动 GenericSpecAdapter 执行入口发现与规格解析，支持 dom_anchor 单层、dom_anchor_two_level 两级、xhr_api 接口三种发现模式 | `class GenericSpecAdapter`, `build_generic_adapter()`, `series_entries()`, `_discover_two_level()`, `_discover_xhr_api()`, `parse_spec_model()` |
| `海外/overseas/spec_parser.py` | 35KB | SPEC 表解析器，把各品牌官网规格表（海信/索尼/松下/夏普/TCL/REGZA 及北美英文站）解析成统一的 SpecSheet，处理 rowspan/colspan、机型列对齐、三元组与平铺键值对等多种结构，并调用词典翻译填入 item_zh | `parse_hisense_table()`, `parse_vertical_spec_table()`, `parse_panasonic_table()`, `parse_sharp_tables()`, `parse_en_kv_pairs()`, `merge_single_model_sheets()`, `GENERIC_SPEC_JS` |
| `海外/overseas/spec_generator.py` | 35.5KB | 阶段三模块，由探测报告经 LLM 生成 AdapterSpec 并做静态校验（字段/正则/域名同源/白名单）与真实入口 HTML 上实测打分（命中率+语义+找全率），不达标则回喂问题修复最多 max_repair 轮 | `generate_spec()`, `validate_spec()`, `score_spec_on_html()`, `prune_html()`, `build_generate_messages()`, `build_repair_messages()`, `class SpecValidationError` |
| `海外/overseas/spec_probe.py` | 14.5KB | 陌生网站结构探测模块，给定入口 URL 自动摸清电视总览页的候选产品链接选择器、URL 模式、分页/加载方式，输出结构化探测报告供 LLM 总结成 AdapterSpec；支持在线（fetcher 打开页面）与离线（传已渲染 HTML）两种方式 | `build_probe_report()`, `probe_site()`, `_rank_link_selectors()`, `_detect_pager()`, `pruned_entry_html()` |
| `海外/overseas/spec_parser_us.py` | 20KB | 美国线 SPEC 解析辅助模块，把 TCL/Sony/Samsung/LG/Hisense 美国官网的 div 键值块、内嵌 JSON、Next.js 数据、Wix 组件等异构结构规整成统一三元组再横排合并成单机型 SpecSheet | `build_single_model_sheet()`, `parse_tcl_us_html()`, `parse_sony_us_html()`, `parse_lg_us_html()`, `parse_samsung_us_html()`, `extract_balanced()` |
| `海外/overseas/spec_export.py` | 23KB | SPEC 规格导出模块，将各品牌系列规格数据按「区分｜项目｜项目(中文)｜机型1｜机型2…」横排格式导出为 CSV 和 Excel；按归一化日文项目键跨系列合并去重 | `export_csv()`, `export_xlsx()`, `_collect()`, `_norm_item_ja()`, `load_release_map()` |
| `海外/overseas/spec_items.py` | 4.4KB | SPEC 项目主行名规范化工具，处理已确认不改变规格语义的页面异写与脏数据，统一空白/斜杠/括号写法、移除脚注标记 | `canonical_item_ja()`, `_normalize_surface()` |
| `海外/overseas/spec_identity.py` | 1.2KB | SPEC 内部身份辅助函数，为存在 family_id 的系列生成稳定内部键（series + [family_id=...]），用于 spec_series / spec_series_status 的入库与续跑判定 | `family_series_key()`, `display_series_name()` |

### 1.4 零售线（价格+网评）模块

| 文件路径 | 大小 | 功能说明 | 核心函数/类 |
|---|---|---|---|
| `海外/overseas/retail_spec.py` | 52KB | 零售线契约模块，负责 RetailSpec（纯 JSON 配置）的加载与 L0~L3 静态校验（结构/安全同源/语义/红线 on_no_match=skip），并提供配置驱动的型号三段兜底匹配、多店铺价格抽取与列映射、评价翻页 URL 生成及 spec 驱动的评价分页收集 | `validate_retail_spec()`, `match_product()`, `explain_no_match()`, `extract_shop_prices()`, `review_page_url()`, `collect_reviews_paged()`, `payload_matches_model()` |
| `海外/overseas/retail_common.py` | 9KB | 零售站通用抽取工具，基于商品页内嵌 JSON-LD 同步抽取商品信息/价格/评分聚合/评价列表，返回统一 ProductPayload；提供搜索结果链接候选提取、价格/浮点/整数解析辅助函数 | `jsonld_product_payload()`, `search_link_hits()`, `jsonld_blocks()`, `find_product_node()`, `to_price()` |
| `海外/overseas/retail_probe.py` | 15KB | 零售线探测器，用已知型号搜索目标站、按含型号纯度排序候选容器选择器，并打开 top 商品页探价格/评分/评价选择器命中情况；判定评价翻页方式与多店铺报价表 | `probe_retail_site()`, `detect_review_pagination_ex()`, `detect_multi_shop()`, `probe_product_page()`, `build_retail_report()` |
| `海外/overseas/retail_generator.py` | 13KB | 零售线 schema 生成器，将探测报告经 LLM 生成 RetailSpec JSON 配置（含 search/match/price/reviews/paginate/shops 六段契约），配合静态校验与回喂修复闭环 | `generate_spec()`, `build_generate_messages()`, `build_repair_messages()`, `score_spec_on_pages()`, `spec_from_probe_report()` |
| `海外/overseas/retail_health.py` | 9.6KB | 零售线定期健康探测模块，对已配置 RetailSpec 站点抽 N 个型号走「搜索→匹配→详情」小样验证，统计价格填充率与评价数；填充率或匹配数骤降时告警 | `class HealthResult`, `run_health()`, `check_repo()`, `main()`, `_sample_models()` |
| `海外/overseas/retail_compliance.py` | 6.9KB | 零售线合规白名单准入模块，提供三层合规校验：内置默认域名白名单 + 外部配置合并、抓取前读 robots.txt 判断 Disallow 路径、ensure_allowed 统一入口 | `ensure_allowed()`, `load_allowlist()`, `fetch_robots()`, `robots_disallows()`, `class ComplianceError` |

### 1.5 翻译与评论导出模块

| 文件路径 | 大小 | 功能说明 | 核心函数/类 |
|---|---|---|---|
| `海外/overseas/translator.py` | 5.9KB | 在线翻译回退客户端，用于离线词典未命中时把日文项目名译成中文；基于标准库 urllib，支持 DeepL/Google/自定义三种提供商，结果缓存到 JSON 文件 | `translate_online()`, `_call_deepl()`, `_call_google()`, `_call_custom()`, `flush_cache()` |
| `海外/overseas/xinghai_translation.py` | 19KB | HiGPT/xinghai-ultra 网评 AI 后端，通过 OpenAI 兼容的 Chat Completions 接口把日语网评翻译成简体中文并提炼优缺点；凭据只从环境变量读取，未配置时回退本地 NLLB/Argos | `class XinghaiUltraJapaneseToChinese`, `build_xinghai_hooks()`, `translate()`, `translate_many()`, `extract_pros()`, `extract_cons()` |
| `海外/overseas/review_translation.py` | 26.8KB | 网评翻译核心模块，提供本地 Argos（日→英→中两段）和 NLLB-200（日→中直连）两种离线引擎，并通过 build_translation_hooks 统一暴露给导出层；引擎和远程客户端均延迟加载 | `class ArgosJapaneseToChinese`, `class NllbJapaneseToChinese`, `build_translation_hooks()`, `build_argos_hooks()`, `build_nllb_hooks()`, `class TranslationUnavailable` |
| `海外/overseas/review_export.py` | 6.3KB | 通用网评汇总模板，定义 20 列宽表（含翻译/AI 优缺点/溯源/价格列）供 Amazon、価格.com 等多站点多地区共用；提供可插拔的 AiHooks 钩子集合 | `class ReviewRecord`, `class AiHooks`, `write_csv()`, `write_xlsx()`, `COLUMNS` |
| `海外/overseas/bv_reviews.py` | 12.8KB | BazaarVoice（BV）评价抓取器，直连 BV 的 bfd JSON 接口取评价明细，绕过浏览器渲染；对每条评价保留真实 ProductId/OriginalProductName 溯源 | `fetch_bv_reviews()`, `bv_config()`, `is_bv_spec()`, `bfd_token()`, `_to_review()` |
| `海外/overseas/price_export.py` | 11.6KB | 价格监控汇总宽表，一行一机型，店铺段可按 RetailSpec 动态生成（multi_shop 读 spec.shops.columns，single_shop 简化为渠道/现价两列，无 spec 回落日本 kakaku 8 店铺常量） | `class PriceRecord`, `build_columns()`, `shop_column_names()`, `group_by_brand()`, `write_csv()`, `write_xlsx()` |
| `海外/overseas/price_export_ca.py` | 5KB | 加拿大线价格监控汇总模板，一行一机型，每个零售站一列价格；零售站列从 regions_config 的 ca.retail_sites 动态读取，自动计算最低价及对应零售站 | `class CaPriceRecord`, `retail_columns()`, `build_columns()`, `write_csv()`, `write_xlsx()` |
| `海外/overseas/export.py` | 5.3KB | 从数据库导出网评宽表，复用 review_export 的 20 列通用模板；逐条评价带商品与站点信息，价格取该商品最近一次 price_snapshot | `_rows()`, `export_csv()`, `export_xlsx()` |
| `海外/overseas/llm_client.py` | 11.2KB | 通用 LLM Chat 客户端，支持 xinghai（内网 HiGPT）、deepseek（公网官方 API）、openai_compatible（任意兼容端点）三类 provider；优先用 openai SDK，未安装时回退 urllib | `class LLMClient`, `class LLMUnavailable`, `chat()`, `_chat_openai_sdk()`, `_chat_urllib()`, `_parse_response()` |

### 1.6 API 服务与前端

| 文件路径 | 大小 | 功能说明 | 核心函数/类 |
|---|---|---|---|
| `海外/overseas/api/service.py` | 46KB | 查询服务层，把 SQLite 数据库行整理成面向前端的结构化 JSON；封装地区/品牌/产品边界校验，暴露 health/brands/models/spec/prices/price_history/weekly_price_trend/reviews/status 等只读查询方法 | `class Service`, `health()`, `models()`, `spec()`, `prices()`, `price_history()`, `weekly_price_trend()`, `reviews()`, `status()` |
| `海外/overseas/api/server.py` | 19.2KB | HTTP 服务入口，基于标准库 http.server 实现，零第三方依赖；负责路由分发（/api/health、/api/brands、/api/models、/api/spec、/api/prices、/api/jobs、/api/export/*、/api/probe 等）、静态前端托管、CORS 与命令行启动 | `class Handler`, `_route_get()`, `_route_post()`, `_serve_static()`, `build_parser()`, `main()` |
| `海外/overseas/api/jobs.py` | 32KB | 任务管理与定时调度层：异步触发采集任务，单工作线程 + FIFO 队列串行执行（避免 SQLite 写并发与域名限速），用子进程跑采集脚本并按任务写增量日志；Scheduler 把"任务+参数+触发规则"持久化到 schedules.json | `class Job`, `class JobManager`, `class Scheduler`, `_resolve_python()`, `_task_timeout()` |
| `海外/overseas/api/exporter.py` | 10.6KB | 导出层：把库里的 SPEC / 网评 / 价格数据生成 CSV 或 XLSX 字节流供 HTTP 下载；日本线与 CA 等零售线分两套 SQL 与记录类型，网评导出可选现场翻译 | `export_spec()`, `export_reviews()`, `export_prices()`, `_review_records()`, `_price_records()`, `_emit()` |
| `海外/overseas/api/regions.py` | 6.5KB | 地区→站点→任务能力的元数据模块，供前端地区切换与站点通道验证使用；声明日本线（已启用）与加拿大线（从 regions_config 构造）、欧美占位 | `REGIONS`, `_build_ca_region()`, `all_probe_targets()`, `REGION_BY_CODE`, `BRAND_DISPLAY` |
| `海外/overseas/api/probe.py` | 3.2KB | 站点可访问性验证（通道验证）模块，抓取前探测目标站点是否可访问、是否被拦；默认用轻量 urllib HTTP 探测，use_browser=True 时用 Playwright 浏览器探测 | `probe_url()`, `_probe_http()`, `_probe_browser()`, `_result()` |
| `海外/overseas/api/static/index.html` | 13KB | 海外电视数据采集台前端单页（零依赖原生 HTML），含抓取 SPEC/评价/价格、通道验证、定时任务、运行记录六大 Tab | 无（静态页面） |
| `海外/overseas/api/static/app.js` | 34.7KB | 海外电视数据采集台前端逻辑（零依赖原生 JS），实现 API 调用、Tab 切换、表单提交、任务列表轮询、导出下载 | `api()`, `apiGet()`, `apiPost()`, `toast()`, 各 Tab 渲染函数 |
| `海外/overseas/api/static/style.css` | 9KB | 海外电视数据采集台前端样式表 | 无（静态样式） |

### 1.7 墨西哥线（MX）模块

| 文件路径 | 大小 | 功能说明 | 核心函数/类 |
|---|---|---|---|
| `海外/overseas/mx_retail.py` | 31KB | 墨西哥线零售抓取编排：按"型号 × 启用零售站"在同一商品页同步抓价格 + 网评；复用日本线 scenarios 的型号匹配、collect_detail 同步抽取与评价翻页补齐，强制 MXN 币种，含站级熔断 | `class RetailRow`, `crawl_model_on_site()`, `crawl_region_retail()`, `resolve_region_currency()`, `rows_to_price_records()`, `rows_to_review_records()`, `write_mx_price_csv()` |

### 1.8 站点适配器层（sites/）

**架构**：`sites/<code>/adapter.py` 继承 `SiteAdapter` 基类（定义于 `海外/overseas/sites/__init__.py`，12.5KB），配一份 `sites/<code>/schema.json` 声明式选择器。`SiteRegistry` 自动发现并注册所有适配器。适配器只负责「URL 怎么拼」和「列表页怎么取商品」，字段抽取全部交给 schema.json + Extractor，保持站点代码极薄。

**统一核心文件**：
| 文件路径 | 大小 | 功能说明 | 核心函数/类 |
|---|---|---|---|
| `海外/overseas/sites/__init__.py` | 12.5KB | 站点适配插件层基类 + 自动发现注册中心；定义 SiteAdapter 基类（URL 规则、抽取器、SPEC 能力、零售能力等钩子）与 SiteRegistry（动态 import sites/<code>/adapter.py 并登记 SiteAdapter 子类） | `class SiteAdapter`, `class SiteRegistry`, `SiteRegistry.discover()`, `SiteRegistry.get()`, `SiteAdapter.parse_product()`, `SiteAdapter.find_product()`, `SiteAdapter.parse_search()` |
| `海外/overseas/sites/amazon_common.py` | 8.5KB | Amazon 各站（ca/us/mx）共用逻辑：价格区校验——防止把推荐位/配件轮播的价格当成电视价（源于 Hisense 32A4NV 配件价事故） | `price_in_region()`, `parse_amazon_product()`, `PRICE_GOOD`, `PRICE_BAD`, `PRICE_REGION_JS` |
| `海外/overseas/sites/walmart_common.py` | 8.4KB | Walmart 共用逻辑：取价护栏（沿祖先链判断价格元素是否落在可信容器内）+ 权威型号抽取（JSON-LD 的 model/mpn 与规格表 Model 行） | `WALMART_PRICE_GOOD`, `WALMART_PRICE_BAD`, `PRICE_REGION_JS`, `extract_models()` |
| `海外/overseas/sites/costco_common.py` | 4.9KB | Costco 共用的商品页增强解析：因 Costco 标题不含完整型号（写"55 Class - U7SG Series"），从页头 item-number 与规格表 Model 行两处抽取权威型号码 | `extract_models()`, `_clean_model()`, `_HEADER_MODEL_RE`, `_SPEC_MODEL_RE`, `_BV_PID_RE`, `_SIZE_RE` |
| `海外/overseas/sites/ca_spec_common.py` | 3.7KB | 加拿大官网 SPEC 入口页的通用链接抽取与电视范围过滤工具；过滤掉显示器/音响/冰箱等非电视品类 | `clean_url()`, `anchor_records()`, `_NON_TV_RE` |
| `海外数据爬取_部署包/overseas/sites/eu_spec_common.py` | 7KB | 欧洲官网 SPEC 入口页的通用链接抽取与电视范围过滤工具（欧洲线扩展） | `clean_url()`, `anchor_records()`, `_NON_TV_RE` |
| `海外/overseas/sites/generic_jsonld/adapter.py` | 1.4KB | 通用 JSON-LD 适配器，适用于页面有稳定 JSON-LD 的站点，免写站点解析代码 | `class GenericJsonLdAdapter` |

**手写适配器（>4KB，含站点专有解析逻辑的重要适配器）**：

| 文件路径 | 大小 | 站点/品牌 | 功能说明 | 核心类 |
|---|---|---|---|---|
| `海外/overseas/sites/kakaku_jp/adapter.py` | 14.6KB | 価格.com（日本比价站） | 日本比价站多店铺价格+评价采集，8 店铺逐字对齐 | `class KakakuJpAdapter` |
| `海外/overseas/sites/samsung_us/adapter.py` | 20KB | Samsung 美国 | Samsung 美国官网 SPEC 规格表解析 | `class SamsungUsAdapter` |
| `海外/overseas/sites/samsung_mx/adapter.py` | 24.6KB | Samsung 墨西哥 | Samsung 墨西哥官网 SPEC 解析 | `class SamsungMxAdapter` |
| `海外/overseas/sites/samsung_ca/adapter.py` | 10.6KB | Samsung 加拿大 | Samsung 加拿大官网 SPEC 解析 | `class SamsungCaAdapter` |
| `海外/overseas/sites/tcl_us/adapter.py` | 20.2KB | TCL 美国 | TCL 美国官网 SPEC 解析 | `class TclUsAdapter` |
| `海外/overseas/sites/lg_us/adapter.py` | 13.2KB | LG 美国 | LG 美国官网 SPEC 解析 | `class LgUsAdapter` |
| `海外/overseas/sites/lg_ca/adapter.py` | 14.1KB | LG 加拿大 | LG 加拿大官网 SPEC 解析 | `class LgCaAdapter` |
| `海外/overseas/sites/lg_mx/adapter.py` | 12.9KB | LG 墨西哥 | LG 墨西哥官网 SPEC 解析 | `class LgMxAdapter` |
| `海外/overseas/sites/hisense_us/adapter.py` | 12.9KB | Hisense 美国 | Hisense 美国官网 SPEC 解析 | `class HisenseUsAdapter` |
| `海外/overseas/sites/hisense_mx/adapter.py` | 14.4KB | Hisense 墨西哥 | Hisense 墨西哥官网 SPEC 解析 | `class HisenseMxAdapter` |
| `海外/overseas/sites/hisense_ca/adapter.py` | 8.2KB | Hisense 加拿大 | Hisense 加拿大官网 SPEC 解析 | `class HisenseCaAdapter` |
| `海外/overseas/sites/sony_us/adapter.py` | 9.3KB | Sony 美国 | Sony 美国官网 SPEC 解析 | `class SonyUsAdapter` |
| `海外/overseas/sites/sony_ca/adapter.py` | 10.9KB | Sony 加拿大 | Sony 加拿大官网 SPEC 解析 | `class SonyCaAdapter` |
| `海外/overseas/sites/philips_ca/adapter.py` | 10.6KB | Philips 加拿大 | Philips 加拿大官网 SPEC 解析 | `class PhilipsCaAdapter` |
| `海外/overseas/sites/tcl_ca/adapter.py` | 10KB | TCL 加拿大 | TCL 加拿大官网 SPEC 解析 | `class TclCaAdapter` |
| `海外/overseas/sites/tcl_jp/adapter.py` | 5.9KB | TCL 日本 | TCL 日本官网 SPEC 解析 | `class TclJpAdapter` |
| `海外/overseas/sites/tcl_mx/adapter.py` | 8.4KB | TCL 墨西哥 | TCL 墨西哥官网 SPEC 解析 | `class TclMxAdapter` |
| `海外/overseas/sites/panasonic_jp/adapter.py` | 5.1KB | Panasonic 日本 | Panasonic 日本官网多级 SPEC（入口→系列→型号页）解析 | `class PanasonicJpAdapter` |
| `海外/overseas/sites/sharp_jp/adapter.py` | 3.1KB | Sharp 日本 | Sharp 日本官网 SPEC 解析 | `class SharpJpAdapter` |
| `海外/overseas/sites/regza_jp/adapter.py` | 3.3KB | REGZA 日本 | REGZA（东芝）日本官网 SPEC 解析 | `class RegzaJpAdapter` |
| `海外/overseas/sites/sony_jp/adapter.py` | 3.3KB | Sony 日本 | Sony BRAVIA 日本官网 SPEC 解析 | `class SonyJpAdapter` |
| `海外/overseas/sites/hisense_jp/adapter.py` | 2.9KB | Hisense 日本 | Hisense 日本官网 SPEC 解析 | `class HisenseJpAdapter` |
| `海外/overseas/sites/bestbuy_us/adapter.py` | 6.4KB | Best Buy 美国 | Best Buy 美国站 sitemap 确定性抓取 | `class BestBuyUsAdapter` |
| `海外/overseas/sites/bestbuy_ca/adapter.py` | 4.4KB | Best Buy 加拿大 | Best Buy 加拿大站采集 | `class BestBuyCaAdapter` |
| `海外/overseas/sites/amazon_ca/adapter.py` | 4KB | Amazon 加拿大 | Amazon 加拿大站适配器，复用 amazon_common 价格区校验 | `class AmazonCaAdapter` |
| `海外/overseas/sites/amazon_us/adapter.py` | 4.6KB | Amazon 美国 | Amazon 美国站适配器 | `class AmazonUsAdapter` |
| `海外/overseas/sites/amazon_mx/adapter.py` | 4.7KB | Amazon 墨西哥 | Amazon 墨西哥站适配器 | `class AmazonMxAdapter` |
| `海外/overseas/sites/walmart_us/adapter.py` | 4.8KB | Walmart 美国 | Walmart 美国站适配器，复用 walmart_common | `class WalmartUsAdapter` |
| `海外/overseas/sites/costco_ca/adapter.py` | 3.8KB | Costco 加拿大 | Costco 加拿大站适配器，复用 costco_common 抽权威型号 | `class CostcoCaAdapter` |
| `海外/overseas/sites/costco_us/adapter.py` | 2.9KB | Costco 美国 | Costco 美国站适配器 | `class CostcoUsAdapter` |
| `海外/overseas/sites/palacio_mx/adapter.py` | 4.4KB | El Palacio de Hierro 墨西哥 | 墨西哥高端百货零售站适配器 | `class PalacioMxAdapter` |

**模板适配器（1-2KB，纯配置驱动）**：以下站点的 adapter.py 仅声明 code/name/base_url/country/channel 等元数据与 search_url/product_url 规则，字段抽取完全由同目录 schema.json 驱动，逻辑高度同质。**部署包（`海外数据爬取_部署包/overseas/sites/`）含全球 200+ 站点**，核心包（`海外/overseas/sites/`）含北美/日本/南美约 90 站点。代表性站点包括：

- **Amazon 系（8 站）**：amazon_ca/us/mx/de/fr/it/es/uk/be/nl/pl/se/tr
- **南美/中美系（30+ 站）**：falabella_cl/co/pe、ripley_cl/pe、lider_cl、paris_cl、mercadolibre_cl/pe、alkosto_co、exito_co、jumbo_co、olimpica_co、hiraoka_pe、carsa_pe、metro_pe、oechsle_pe、plazavea_pe、estilos_pe、credivargas_pe、lacuracao_pe、efe_pe、artefacta_ec、comandato_ec、cresa_ec、jaher_ec、japon_ec、marcimex_ec、orvehogar_ec、sukasa_ec、todohogar_ec、ferrisariato_ec、coral_ec、megamaxi_ec、multicenter_bo、dismac_bo、alkostoplus_bo、samsung_bo、tcl_bo、hisense_bo/pe/co/ec、lg_ec、sodimac_pe、tottus_pe、falabella_pe、abc_cl、hites_cl、multimax_pa、panafoto_pa、raenco_pa、rodelag_pa、evision_pa、tecnofacil_gt、unicomer_gt、max_gt、elektra_gt
- **欧洲系（60+ 站）**：mediamarkt_de/at/be/ch/es/hu/it/nl/pl/tr、saturn_de、conrad_de、alternate_de、cyberport_de/at、connect_de、expert_de、computerbild_de、chip_de、hifitest_de、ebay_de/es/fr/it/uk、amazon_de/fr/it/es/uk、cdiscount_fr、darty_fr、fnac_fr/pt、boulanger_fr、conforama_fr、auchan_fr、carrefour_fr/es、leclerc_fr、ldlc_fr、rueducommerce_fr、sonvideo_fr、electrodepot_fr、elcorteingles_es、pccomponentes_es、worten_es/pt、radiopopular_pt、coolblue_be/nl、bol_be/nl、vandenborre_be、allegro_pl、mediaexpert_pl、morele_pl、euro_agd_pl、alza_cz/hu/sk、czc_cz、datart_cz/sk、mall_cz、proshop_dk、elgiganten_dk/se、komplett_no/se、power_dk/fi/no/se、elkjop_no、gigantti_fi、verkkokauppa_fi、brack_ch、digitec_ch、galaxus_ch、interdiscount_ch、fust_ch、euronics_de/ee/it/lv、otto_de、skroutz_gr、public_gr、plaisio_gr、kotsovolos_gr、220_lv、unieuro_it、trony_it、eprice_it、hwupgrade_it、techzilla_it、argos_uk、currys_uk、ao_uk、jd_uk、jlp_uk、very_uk、which_uk、richersounds_uk、markselectrical_uk、hughes_uk、samsung_uk、sony_uk、tcl_uk、emag_bg/hu/ro、altex_ro、flanco_ro、hepsiburada_tr、trendyol_tr、vatan_tr、teknosa_tr、medimax（各站）、mvideo_ru、dns_ru、citilink_ru、eldorado_ru、ozon_ru、wildberries_ru、yandex_market_ru、technopark_ru、allo_ua、comfy_ua、foxtrot_ua、rozetka_ua、gigatron_rs、tehnomanija_rs、ananas_rs、tehnomag_ba、domod_ba、bigbang_si、mimovrste_si、harveynorman_si、kaup24_ee、klick_ee、nay_sk、rdveikals_lv、pigu_lt、varle_lt、topocentras_lt、tehnomanija_rs、sancta_domenica_hr、ekupi_hr、elipso_hr、links_hr

**对应 schema.json**：每个有零售能力的站点配一份 `schema.json`（2-7KB），声明 search/price/reviews/pagination 段的选择器、JSON-LD 字段、评价来源（pdp_embedded/bazaarvoice_bfd/vtex_reviews_graphql/powerreviews_api/reviews_io_api 等）。共 296 份 schema.json。

### 1.9 部署包专属模块

> 以下模块仅存在于 `海外数据爬取_部署包/overseas/`，是部署包相对于核心包的扩展。

| 文件路径 | 大小 | 功能说明 | 核心函数/类 |
|---|---|---|---|
| `海外数据爬取_部署包/overseas/backfill.py` | 14KB | 待维护参数回溯补全引擎，对型号缺失的 18 个规格参数做三层递进回溯（数据库精确关键词搜索→官网重抓→AI 查找官网 URL）；AI 仅用于查找信息源不产出规格值，找不到则诚实标记"待维护" | `backfill_model_params()`, `_backfill_from_db()`, `_backfill_from_site()`, `_backfill_from_ai()`, `trigger_site_crawl()`, `BACKFILL_KEYWORDS` |
| `海外数据爬取_部署包/overseas/benchmark_server.py` | 108KB | 竞品对阵分析后端服务，提供国家选择→AI 对阵推荐→SPEC 对比→雷达图→AI 结论的 HTTP 接口；内置 20+ 国家的价格带阈值与货币配置，含海量日文/英文/西文规格值归一化显示规则 | `COUNTRIES`, `_DISPLAY_NORMALIZE`, `_normalize_display()`, `class Handler`, `api_ai_recommend()`, `api_spec_compare()`, `api_radar()`, `api_ai_analysis()`, `api_backfill()`, `main()` |
| `海外数据爬取_部署包/overseas/mms_reviews.py` | 14.6KB | MediaMarkt/Saturn/MediaWorld（MMS 集团）商品评价采集模块；第 1 页取 SSR 内联的 apolloState 评价实体，后续页通过点真实分页按钮截获站方自身 GraphQL 响应（避免拼 URL 重放被 WAF 拦） | `fetch_mms_reviews()`, `ssr_reviews()`, `_to_review()`, `_decode_at()`, `product_id_from_url()` |
| `海外数据爬取_部署包/overseas/price_sanity.py` | 2KB | 电视价格合理性检查工具，按币种给出电视最低售价下限，用于拦截分期月供/运费/评分/配件价等明显错价；遵循"缺价>错价"原则 | `MIN_TV_PRICE`, `check_prices()` |
| `海外数据爬取_部署包/overseas/regions_auto.py` | 32KB | 自动生成的欧洲各国+危地马拉+巴拿马零售区域配置表（由 gen_regions_from_specs.py 从 retail spec 生成，勿手改）；每区域定义品牌列表与零售站点及 enabled 状态 | `AUTO_REGIONS`, `EXTRA_RETAIL_SITES` |
| `海外数据爬取_部署包/overseas/report/generate_report.py` | 35.8KB | DeepResearch 海外电视市场调研报告生成系统，基于 6 个 Agent 协作（规划/采集/分析/对手/撰写/审核）生成 Markdown 报告；所有结论锚定 DB 真实数据并标注信源 ID 防幻觉 | `class ChiefArchitect`, `class DataScout`, `class DataAnalyst`, `class CompetitorAnalyst`, `class ReportWriter`, `class CriticMaster`, `generate_report()` |
| `海外数据爬取_部署包/overseas/selling_point_export.py` | 4KB | 卖点 Excel/CSV 导出模块，按品牌导出每条卖点一行（品牌/系列/机型/类型/标题/描述/图片视频 URL/LLM 精炼摘要） | `export_xlsx()`, `export_csv()`, `_collect()` |
| `海外数据爬取_部署包/overseas/selling_point_md_export.py` | 10KB | 卖点 Markdown 导出模块，按品牌生成一个 MD 文档，每机型一节含精炼卖点编号列表与结合 Spec 的 200 字 LLM 摘要（缓存到独立 summary 表可重入） | `export_md()`, `collect()`, `ensure_summaries()`, `render_md()`, `_clean_refined()`, `SUMMARY_PROMPT` |

### 1.10 LangGraph 全自动流程

> 部署包独有的 LangGraph 编排引擎，把 250 站爬取全流程改造为状态图。

| 文件路径 | 大小 | 功能说明 | 核心函数/类 |
|---|---|---|---|
| `海外数据爬取_部署包/overseas/flow/langgraph_crawl.py` | 6.2KB | LangGraph 全自动海外爬取流程引擎，构建 init→spec→retail→review→verify→report→retry 的状态图，支持 5 线（日/北美/南美/欧/亚）串行执行与 JSON 断点续跑 | `build_graph()`, `run_line()`, `run_all()`, `_save_checkpoint()`, `_load_checkpoint()`, `main()` |
| `海外数据爬取_部署包/overseas/flow/state.py` | 2.1KB | LangGraph 全局状态 TypedDict 定义，声明爬取流程所需的全部字段：全局配置、站点列表、各阶段结果、失败/封锁/超时站、重试控制、流程阶段、增量统计、断点续跑标记等 | `class CrawlState (TypedDict)` |
| `海外数据爬取_部署包/overseas/flow/nodes/init.py` | 5.6KB | 初始化节点，负责磁盘空间检查、备份 DB、按线路加载对应的 SPEC 与 RETAIL 站点列表，记录基线计数供后续增量对比 | `LINE_COUNTRIES`, `_get_sites_for_line()`, `_check_disk_space()`, `init_node()` |
| `海外数据爬取_部署包/overseas/flow/nodes/spec_crawl.py` | 4.8KB | SPEC 规格抓取节点，通过 ThreadPoolExecutor 并行（默认 2 站）在独立子进程中调用 spec_crawl 抓取品牌官网规格，每站超时 600 秒 | `_run_spec()`, `_run_one_spec()`, `spec_node()` |
| `海外数据爬取_部署包/overseas/flow/nodes/retail_monitor.py` | 5.8KB | RETAIL 价格监控节点，通过 ThreadPoolExecutor 并行（默认 3 站）在独立子进程中调用 s2_monitor_known 抓取价格，每站 90 秒超时、每站最多 10 个 SKU | `_run_monitor()`, `_run_one()`, `retail_node()` |
| `海外数据爬取_部署包/overseas/flow/nodes/review_incremental.py` | 3.7KB | 网评增量抓取节点，查询已有 review 的零售站（最多 10 站、每站前 20 个 SKU），在子进程中调用 s4_review_incremental（pages=3，超时 120s）抓取新网评 | `review_node()`, `REVIEW_SKU_LIMIT` |
| `海外数据爬取_部署包/overseas/flow/nodes/verify.py` | 1.6KB | 验证节点，连接 DB 读取抓取后的 spec_row/price_snapshot/review 计数，与 init 节点记录的基线对比算出增量；增量为 0 时追加警告（不阻塞流程） | `verify_node()` |
| `海外数据爬取_部署包/overseas/flow/nodes/retry.py` | 1.9KB | 失败站重试节点，对 failed_sites 中未达最大重试次数（默认 2 次）的站点重新调用 _run_monitor，成功则移出失败列表，重试耗尽则标记为 blocked_sites | `retry_node()` |
| `海外数据爬取_部署包/overseas/flow/nodes/report.py` | 3.6KB | 报告生成节点，汇总 SPEC/RETAIL/REVIEW 各阶段结果与前后增量数据，渲染成 Markdown 表格报告（含成功/失败/超时/封锁统计） | `LINE_NAMES`, `report_node()` |

---

## 二、对阵分析服务代码

> 位于 `竞品对阵分析_部署包/overseas/`，是海外爬取框架的独立部署形态，专注竞品对阵分析（国家选择→AI 对阵推荐→SPEC 对比→雷达图→AI 结论）。

| 文件路径 | 大小 | 功能说明 | 核心函数/类 |
|---|---|---|---|
| `竞品对阵分析_部署包/overseas/benchmark_server.py` | 108KB | 竞品对阵分析后端 HTTP 服务，基于 SQLite 库提供国家选择、AI 对阵推荐、规格对比、雷达图、AI 结论及参数回溯/自动溯源等 REST API，并托管前端静态页面；内置 20+ 国家的价格带阈值配置和海量规格值归一化正则规则；启动时注入本地 xinghai-ultra 代理，默认监听 127.0.0.1:8090 | `COUNTRIES`, `_DISPLAY_NORMALIZE`, `_normalize_display()`, `class Handler`, `api_ai_recommend()`, `api_spec_compare()`, `api_radar()`, `api_ai_analysis()`, `api_backfill()`, `api_auto_trace()`, `_setup_xinghai()`, `main()` |
| `竞品对阵分析_部署包/overseas/db.py` | 58KB | 存储层 SQLite 实现，对外只暴露 Database 类，所有 SQL 收于此；价格/评价汇总/排名只做 INSERT 不 UPDATE 保留历史快照；评价用 INSERT OR IGNORE + UNIQUE 增量去重；含 schema 初始化、旧库迁移、SPEC 系列按周快照、kakaku/零售型号级任务状态记录与聚合查询 | `class Database`, `init_schema()`, `_migrate_columns()`, `upsert_product()`, `add_reviews()`, `save_spec_sheet()`, `record_kakaku_status()`, `weekly_price_series()`, `stats()` |
| `竞品对阵分析_部署包/overseas/config.py` | 14.8KB | 全局配置模块，所有参数可通过 OVERSEAS_ 前缀环境变量覆盖；涵盖路径、抓取节流与熔断阈值、浏览器、代理探活、翻译接口、网评 AI、LLM 配置 | `DB_PATH`, `CACHE_DIR`, `PROFILE_DIR`, `llm_settings()`, `ensure_dirs()`, `session_path()`, `profile_dir()`, `summary()` |
| `竞品对阵分析_部署包/overseas/llm_client.py` | 11.2KB | 通用 LLM Chat Completions 客户端，支持 xinghai/deepseek/openai_compatible 三类 provider；优先用 openai SDK，未安装时回退 urllib（含 SSE 流式解析与可重试状态码退避） | `class LLMUnavailable`, `class LLMClient`, `LLMClient.chat()`, `_chat_openai_sdk()`, `_chat_urllib()` |
| `竞品对阵分析_部署包/overseas/backfill.py` | 14KB | 待维护参数回溯补全引擎，对指定型号缺失参数做三层递进回溯（DB 精确搜索→官网重抓→AI 查找官网 URL）；内置 18 个参数的精确搜索关键词表和日本线品牌→适配器映射 | `BACKFILL_KEYWORDS`, `BRAND_TO_ADAPTER`, `backfill_model_params()`, `_backfill_from_db()`, `_backfill_from_site()`, `_backfill_from_ai()`, `trigger_site_crawl()` |
| `竞品对阵分析_部署包/run_benchmark_server.py` | — | 启动脚本，拉起 benchmark_server | `main()` |
| `竞品对阵分析_部署包/启动竞品对阵分析.bat` | — | Windows 一键启动批处理 | 无（脚本） |

**前端**：
| 文件路径 | 大小 | 功能说明 | 核心函数/类 |
|---|---|---|---|
| `竞品对阵分析_部署包/overseas/benchmark_static/index.html` | 39.8KB | 竞品对阵分析前端单页，提供国家选择、AI 对阵推荐、SPEC 对比表、雷达图、AI 结论展示 | 无（静态页面） |
| `竞品对阵分析_部署包/overseas/benchmark_static/echarts.min.js` | 1029KB | ECharts 图表库（用于雷达图渲染） | 无（第三方库） |

---

## 三、认证智能体代码

> 位于 `compintel-deploy/overseas/compintel/`，是竞品认证情报（compintel）抓取与预警系统，抓取全球 6 类认证数据库（FCC/EPREL/ENERGY STAR/ANATEL/IC-ISED/TÜV+VDE）并识别竞品上市预警信号。

### 3.1 编排与 API 层

| 文件路径 | 大小 | 功能说明 | 核心函数/类 |
|---|---|---|---|
| `compintel-deploy/overseas/compintel/orchestrator.py` | 44.6KB | 竞品情报抓取的薄编排层，负责建 run、调用各数据源爬虫、落库并触发预警信号识别；内置品牌关键词/授权码别名表，支持 FCC 全量分页、EPREL 列表/扫描/供应商搜索、ENERGY STAR/ANATEL/IC-ISED/TÜV+VDE 等数据源的一键串行抓取 | `discover_brand_grantees()`, `crawl_fcc_full()`, `crawl_fcc()`, `crawl_eprel()`, `crawl_eprel_scan()`, `crawl_eprel_supplier()`, `crawl_all_extended()`, `_normalize_brand()` |
| `compintel-deploy/overseas/compintel/web_api.py` | 11.5KB | 竞品情报可视化看板的后端 API，基于 Python 标准库 http.server 零依赖实现；暴露 /api/stats、/api/alerts、/api/records、/api/history、/api/run、/api/analyze 等 JSON 路由，前端单页由同目录 dashboard.html 提供 | `class DashboardHandler`, `serve()`, `_row_to_serializable()` |
| `compintel-deploy/compintel_cli.py` | 10.5KB | 竞品情报系统的独立 CLI 入口，不依赖海外采集框架的 sites/extract 模块；提供 compintel-web、compintel-fcc、compintel-eprel(-scan/-supplier)、compintel-energystar、compintel-anatel、compintel-ic-ised、compintel-certs、compintel-run-extended、compintel-alert 等子命令 | `main()`, `cmd_compintel_web()`, `cmd_compintel_fcc()`, `cmd_compintel_eprel_supplier()`, `cmd_compintel_run_extended()`, `cmd_compintel_alert()` |

### 3.2 各认证源爬虫

| 文件路径 | 大小 | 功能说明 | 核心函数/类 |
|---|---|---|---|
| `compintel-deploy/overseas/compintel/fcc_crawler.py` | 14.6KB | FCC 认证记录爬虫，数据源为 fccid.io 聚合站（官方 apps.fcc.gov 被 Akamai 403）；用正则解析搜索页/授权码分页/详情页 HTML，支持"品牌关键词发现授权码→遍历所有分页"的全量抓取与增量早停，及保密文件清单提取 | `class FccCrawler`, `class FccRecord`, `parse_search_results()`, `parse_company_results()`, `parse_grantee_page()`, `discover_grantees()` |
| `compintel-deploy/overseas/compintel/eprel_crawler.py` | 11.5KB | EPREL 欧盟能效注册记录爬虫（电子显示器品类），调用 EPREL 官方 JSON API；因列表接口固定 25 条且不可翻页，提供按供应商名搜索、按注册号区间扫描等多种抓取策略 | `class EprelCrawler`, `class EprelRecord`, `hit_to_record()`, `list_recent()`, `search_by_supplier()`, `scan_range()` |
| `compintel-deploy/overseas/compintel/energystar_crawler.py` | 8.3KB | ENERGY STAR 能效认证爬虫，数据源为 Socrata SODA API（免 API key）；按 brand_name + date_certified 用 SoQL 分页拉取，支持品类过滤和增量 since 参数 | `class EnergyStarCrawler`, `_api_get()`, `fetch_by_brand()`, `fetch_all_brands()`, `_normalize()` |
| `compintel-deploy/overseas/compintel/anatel_crawler.py` | 9.2KB | ANATEL 巴西电信设备认证爬虫，下载 ANATEL 开放数据每日更新的 ZIP-CSV（19 万+条）；按制造商名关键词过滤竞品品牌，支持按 Homologação 号去重和 since 增量 | `class AnatelCrawler`, `fetch_all()`, `_download_zip()`, `_parse_csv()`, `_parse_date_br()`, `BRAND_FABRICANTE_MAP` |
| `compintel-deploy/overseas/compintel/ic_ised_crawler.py` | 17.9KB | IC/ISED 加拿大无线电设备认证爬虫，数据源为 ISED SMS 数据库的 Spring WebFlow 表单；通过提取 execution token + 维护 cookie jar 完成"搜公司名→取公司号→按公司号查 REL"三步流程，内置六大竞品公司号缓存 | `class IcedCrawler`, `search_companies()`, `search_rel()`, `fetch_all_brands()`, `ISED_COMPANY_NUMBERS` |
| `compintel-deploy/overseas/compintel/cert_crawler.py` | 10.7KB | TÜV Rheinland（Certipedia）与 VDE/CB（Solr JSON）安全认证爬虫适配层，复用 crawl4ai-main/cert_verify 包的 Provider 并改为按公司名搜索；把 cert_verify 的 CertRecord 转为 compintel_record 格式 | `class TuvCertCrawler`, `class VdeCertCrawler`, `_normalize_cert_record()`, `search_by_company()`, `fetch_all_brands()` |

### 3.3 预警与分析层

| 文件路径 | 大小 | 功能说明 | 核心函数/类 |
|---|---|---|---|
| `compintel-deploy/overseas/compintel/alert.py` | 22.5KB | 竞品上市预警信号识别模块，定义 7 类基础信号（new_fcc_id/confidential_expired/eprel_new_reg/energy_class_jump/eprel_delisted/supplier_change/field_change）及通用 new_cert 信号；launch_warning 在基础信号上叠加产品类型过滤、在售比对和提前量计算，只产出真正的"新产品即将上市"预警 | `detect_new_records()`, `detect_confidential_expired()`, `detect_supplier_change()`, `detect_field_changes()`, `detect_energy_class_jump()`, `detect_delisted()`, `detect_launch_warning()` |
| `compintel-deploy/overseas/compintel/llm_analyze.py` | 16KB | LLM 分析模块，调用海信内网聚好看模型（deepseek-v4-flash 等，OpenAI 兼容接口）解读竞品上市预警信号，生成中文简报；支持单条认证信号解读、上市预警解读和按品牌汇总趋势分析，LLM 不可用时优雅降级为规则式摘要 | `call_llm()`, `analyze_cert_signal()`, `analyze_launch_warning()`, `analyze_competitor_trend()`, `batch_analyze()`, `MODEL_CONFIGS` |
| `compintel-deploy/overseas/compintel/compintel_db.py` | 14.4KB | compintel 表族的读写封装，复用主 Database 的 crawl_run 审计与 SQLite 连接；提供记录快照增改、文档清单、信号告警、抓取状态（增量早停）与时序变化检测的专用方法 | `class CompIntelDB`, `add_record()`, `add_doc()`, `seen_source_id()`, `last_record()`, `diff_records()`, `add_alert()`, `alerts_since()`, `stats()` |

### 3.4 复用的框架层（compintel-deploy/overseas/）

> compintel-deploy 复用了海外爬取框架的基础设施层，文件与核心包同名同构。

| 文件路径 | 大小 | 功能说明 | 核心函数/类 |
|---|---|---|---|
| `compintel-deploy/overseas/models.py` | 6.3KB | 领域数据模型（同核心包 models.py） | `Product`, `PriceSnapshot`, `Review`, `SpecSheet` |
| `compintel-deploy/overseas/fetchers.py` | 46KB | 抓取层（同核心包 fetchers.py） | `HttpFetcher`, `BrowserFetcher`, `PlaywrightDom`, `LxmlDom` |
| `compintel-deploy/overseas/infra.py` | 15KB | 基础设施层（同核心包 infra.py） | `RateLimiter`, `BlockDetector`, `DiskCache`, `resolve_proxy()` |
| `compintel-deploy/overseas/db.py` | 57KB | 存储层（同核心包 db.py，含 compintel source CHECK 迁移） | `class Database`, `init_schema()`, `_migrate_compintel_source_check()` |
| `compintel-deploy/overseas/config.py` | 13.9KB | 全局配置（同核心包 config.py） | `llm_settings()`, `ensure_dirs()`, `summary()` |
| `compintel-deploy/overseas/schema.sql` | 21.5KB | SQLite 建表 DDL（含 compintel 相关表） | 无（声明式 DDL） |

---

## 四、Skill 定义文件

> 位于 `海外/.kiro/skills/overseas-schema-gen/`，是项目唯一的 skill（统一入口），三分支自动分派：SPEC（品牌官网→发现型号→规格）、RETAIL（电商零售站→给定型号→抓价+网评）、OPS（已有适配器全量运行/监控/修复）。

### 4.1 主入口与分支定义

| 文件路径 | 大小 | 功能说明 | 核心内容 |
|---|---|---|---|
| `海外/.kiro/skills/overseas-schema-gen/SKILL.md` | 20.5KB | 海外零售站接入的统一 skill 入口文档，定义 SPEC/RETAIL/OPS/FLOW/REPORT 五分支自动分派规则与铁律；描述经验驱动的自动探索闭环：查 cases.jsonl 找相似站→复用模板→探测→校验→转正→沉淀 | 核心架构图、分派表、10 条通用铁律、6 步自动探索流程、工具边界表、经验驱动闭环 |
| `海外/.kiro/skills/overseas-schema-gen/spec/SKILL.md` | 14KB | SPEC 分支完整流程文档，定义给陌生电视品牌官网 URL 产出 AdapterSpec 的 6 步主流程：查经验→探入口→探型号页→写 schema→验证→沉淀；包含卖点采集扩展 | 4 条铁律、Step0-6 主流程、auto_schema 快速路径、卖点采集扩展、DB 落库与导出 |
| `海外/.kiro/skills/overseas-schema-gen/retail/SKILL.md` | 9KB | RETAIL 分支完整流程文档，定义给陌生电商站用型号驱动探测产出 RetailSpec 的 6 步主流程，强制先读 robots.txt 再写选择器；含失败诊断对照表 | 6 条铁律（含 robots 优先）、Step0-5、失败诊断表、判定"该人工写适配器"的信号 |

### 4.2 探测与生成脚本

| 文件路径 | 大小 | 功能说明 | 核心函数/类 |
|---|---|---|---|
| `海外/.kiro/skills/overseas-schema-gen/spec/probe.py` | 15.7KB | SPEC 分支强探测脚本，复用项目 BrowserFetcher/spec_probe/generic_spec 能力；提供 --entry（探入口页产品链接选择器/分页信号）、--model（探型号页规格载体）、--verify（校验 schema）三种模式 | `class _StubAdapter`, `_make_browser_fetcher()`, `cmd_entry()`, `cmd_model()`, `cmd_verify()`, `main()` |
| `海外/.kiro/skills/overseas-schema-gen/spec/probe_sp.py` | 42KB | SPEC 分支自包含探测脚本（无项目依赖），直接用 Playwright API 渲染页面执行 JS；支持 --entry/--model/--sp（卖点页）三种独立探测模式，含可达性预检 | `check_reachable()`, `probe_model()`, `probe_sp()`, `probe_entry()`, `_detect_pager()`, `_rank_link_selectors_from_hrefs()` |
| `海外/.kiro/skills/overseas-schema-gen/spec/auto_schema.py` | 9.3KB | SPEC 分支经验优先自动 Schema 生成器；先查 cases.jsonl 按 4 级优先级匹配相似站（精确 code/同 host/同品牌不同 region/同 host 父域），命中则直接把 solution 转成 schema JSON 不跑浏览器秒出 | `load_cases()`, `extract_host()`, `match_cases()`, `case_to_schema()`, `main()` |
| `海外/.kiro/skills/overseas-schema-gen/spec/spec_keywords.py` | 6.4KB | SPEC 分支多语言规格关键词表，供 probe 判断页面结构是否含规格内容；覆盖 ja/en/es/de/pt 五语种及语言无关单位符号，另含卖点关键词表 | `SPEC_KEYWORDS_BY_LANG`, `SP_KEYWORDS_BY_LANG`, `all_keywords()`, `ALL_KEYWORDS` |
| `海外/.kiro/skills/overseas-schema-gen/spec/selfcheck.py` | 8.6KB | SPEC 分支 skill 骨架健康自检脚本，纯静态核对不联网；检查文件完整性、SKILL.md frontmatter、cases.jsonl 可解析性、文档死链、引擎白名单同步、规范示例、已转正 schema 合法性 | `check_files()`, `check_cases()`, `check_deadlinks()`, `check_whitelist_sync()`, `check_spec_examples()`, `check_real_specs()` |
| `海外/.kiro/skills/overseas-schema-gen/retail/probe.py` | 7.7KB | RETAIL 分支探测入口脚本，包装 overseas.retail_probe 探测器与 retail_generator 生成器；提供在线探测、静态校验 RetailSpec 草稿、从探测信号生成离线模板 spec 三种模式 | `_make_fetcher()`, `cmd_probe()`, `cmd_verify()`, `cmd_template()`, `main()` |
| `海外/.kiro/skills/overseas-schema-gen/retail/selfcheck.py` | 6.3KB | RETAIL 分支 skill 骨架健康自检脚本，纯静态核对不联网；检查文件完整性、cases.jsonl 可解析性、文档死链、零售引擎白名单同步、已转正 retail spec 能过 validate | `check_files()`, `check_cases()`, `check_deadlinks()`, `check_whitelist_sync()`, `check_retail_specs_validate()` |

### 4.3 经验库与运维手册

| 文件路径 | 大小 | 功能说明 | 核心内容 |
|---|---|---|---|
| `海外/.kiro/skills/overseas-schema-gen/ops/crawl-ops.md` | 39.9KB | 海外零售站长时全量爬取运维手册，覆盖启动/监控/自愈/僵尸检测/DB 污染清理/交付核对；含南美欧洲亚洲线平台分类抓取路线、Stealth 增强、LangGraph 并行编排、DeepResearch 报告 | 触发条件、supervisor/watchdog/rebuild 架构、平台分类路线、Stealth 实测、五步流程、benchmark 污染修复 |
| `海外/.kiro/skills/overseas-schema-gen/spec/reference.md` | 42.7KB | SPEC 分支判断经验库，每条来自真实站点实测教训；覆盖产品链接选择器纯度、分页信号可靠性、JS 片段转义坑、规格页解析选档、Samsung captcha 误判修复、SPEC 导出空白诊断、compintel 竞品情报等 | 选产品链接选择器、分页与动态加载、规格页解析、该手写适配器信号、引擎缺口、SPEC 导出诊断、Samsung captcha、compintel 分支 |
| `海外/.kiro/skills/overseas-schema-gen/retail/reference.md` | 77.7KB | RETAIL 分支判断经验库，聚焦匹配准确率（防张冠李戴）；覆盖搜索页探测、短号型词边界匹配红线、Costco 标题拆尺寸守卫、BestBuy robots 合规、VTEX/Magento/Falabella 平台路线、BV 评价接口、价格差异≠错价判定 | 搜索页探测、短号型误命中、Costco 实测、BestBuy 实测、VTEX/Magento/Falabella 平台、kakaku 评价、小样本检验方法论 |
| `海外/.kiro/skills/overseas-schema-gen/spec/cases.jsonl` | — | SPEC 分支接入案例库（每站一条 JSONL），skill 越用越强的核心机制 | 无（JSONL 数据） |
| `海外/.kiro/skills/overseas-schema-gen/retail/cases.jsonl` | — | RETAIL 分支接入案例库 | 无（JSONL 数据） |

### 4.4 剧本（Playbook）

> 结构型套路索引，按探测信号分流到对应套路文档。

**SPEC 分支剧本**（`spec/playbook/`）：

| 文件路径 | 大小 | 功能说明 |
|---|---|---|
| `spec/playbook/README.md` | 2.3KB | SPEC 分支结构型套路索引，按 probe.py --entry 给出的 load_more_hint 信号分流到 7 类套路，并维护已验证站点清单 |
| `spec/playbook/static_lineup.md` | 2.7KB | 静态 lineup 一次列全的发现套路（已验证秘鲁 Hisense/REGZA JP/Sony BRAVIA JP） |
| `spec/playbook/lazy_scroll.md` | 2.4KB | 滚动懒加载追加的发现套路（scroll_until_stable） |
| `spec/playbook/numbered_paginate.md` | 5.6KB | 编号页码分页（含替换式与追加式渲染）发现套路（已验证 Philips CA 100%） |
| `spec/playbook/embedded_json.md` | 3.3KB | 规格藏在内嵌 JSON 里的规格页解析套路（已验证 Philips CA） |
| `spec/playbook/multilevel.md` | 4.3KB | 总览→系列→型号的多级发现套路（目标松下/夏普） |
| `spec/playbook/xhr_api.md` | 5.1KB | JSON 接口出数据的发现套路（目标 TCL/Samsung/LG） |
| `spec/playbook/needs_handwritten.md` | 2.9KB | 止损指南，判定某站超出通用引擎边界应转手写适配器 |
| `spec/playbook/compintel_api_http.md` | 31KB | 第三方合规数据库（FCC/EPREL/ENERGY STAR/ANATEL/IC-ISED/TÜV/VDE）固定程序抓取套路 |

**RETAIL 分支剧本**（`retail/playbook/`）：

| 文件路径 | 大小 | 功能说明 |
|---|---|---|
| `retail/playbook/README.md` | 4.5KB | RETAIL 分支结构型 playbook 索引，按 shops.mode/搜索形态/评价来源分流到 11 类套路 |
| `retail/playbook/single_shop.md` | 3.2KB | 自营站套路（Amazon/Liverpool/Coppel 等），一型号一价、评价在同商品页 |
| `retail/playbook/multi_shop.md` | 1.8KB | 聚合站（比价站，如 kakaku.com）套路，一商品页列多家店铺报价 |
| `retail/playbook/landed_pdp.md` | 1.6KB | 搜索直达详情页套路（裸型号搜索被 302 到 PDP） |
| `retail/playbook/series_only_listing.md` | 4.2KB | 列表页只有系列名（无完整型号）的套路（代表 Costco CA/US） |
| `retail/playbook/sitemap_discovery.md` | 6.3KB | sitemap 确定性发现套路（适用于禁抓搜索但有公开 sitemap 的站，如 bestbuy.ca） |
| `retail/playbook/reviews_page.md` | 1.8KB | 评价独立页+翻页套路 |
| `retail/playbook/next_data_inline.md` | 4.5KB | 评价内嵌在 Next.js __NEXT_DATA__ 的套路（代表 lider.cl） |
| `retail/playbook/vtex_catalog_api.md` | 6.2KB | VTEX 平台 Catalog API 直采套路（南美 15 站） |
| `retail/playbook/vtex_reviews.md` | 12KB | VTEX 站 reviews-and-ratings GraphQL 评价探测与抓取套路 |
| `retail/playbook/yotpo_vtex.md` | 3.6KB | Yotpo 挂件评价直采套路（VTEX 站评价走 Yotpo，已验证 elektra_gt） |
| `retail/playbook/thirdparty_review_api.md` | 14.2KB | 评价在第三方挂件接口（BazaarVoice/Feefo/LFL Group）的套路 |
| `retail/playbook/needs_human.md` | 1.4KB | 需要人工介入的止损套路（验证码/登录墙/加购墙等） |

**任务模板与其他**：

| 文件路径 | 大小 | 功能说明 |
|---|---|---|
| `spec/task_template.md` | 5.4KB | SPEC 分支新站接入任务卡模板（六步标准化流程） |
| `retail/task_template.md` | 3KB | RETAIL 分支新站接入任务卡模板（六步流程） |
| `spec/sony_jp_selling_points_sample.md` | 20KB | Sony 日本官网卖点采集样本（含 Extract 方法示例） |
| `flow/README.md` | 3KB | LangGraph 全自动流程化海外爬取引擎说明文档 |
| `report/README.md` | 3.6KB | DeepResearch 海外电视市场调研报告生成系统说明 |

---

## 五、运维脚本

| 文件路径 | 大小 | 功能说明 | 核心内容 |
|---|---|---|---|
| `海外数据爬取_部署包/deploy/setup_linux.sh` | 3.8KB | Linux 环境准备与自检脚本（幂等可重复执行）：venv→依赖→离线 Chrome→数据库初始化→站点注册自检→网络连通性自检 | 7 步自检（venv/pip/playwright/离线 Chrome/浏览器启动/DB/站点注册/连通性） |
| `海外数据爬取_部署包/scripts/refresh_spec.ps1` | 1.2KB | Windows PowerShell 刷新 SPEC 脚本封装，调用 scripts/refresh_spec.py 重抓/导出规格 | 参数解析（Python 路径/项目根/DB/输出目录/--site/--export-only） |

---

## 六、前端文件

| 文件路径 | 大小 | 功能说明 | 核心内容 |
|---|---|---|---|
| `海外/overseas/api/static/index.html` | 13KB | 海外电视数据采集台前端单页（零依赖原生 HTML），含抓取 SPEC/评价/价格、通道验证、定时任务、运行记录六大 Tab | header/nav/六大 panel |
| `海外/overseas/api/static/app.js` | 34.7KB | 海外电视数据采集台前端逻辑（零依赖原生 JS），实现 API 调用、Tab 切换、表单提交、任务列表轮询、导出下载 | `api()`, `apiGet()`, `apiPost()`, `toast()`, 各 Tab 渲染函数 |
| `海外/overseas/api/static/style.css` | 9KB | 海外电视数据采集台前端样式表 | 无（静态样式） |
| `竞品对阵分析_部署包/overseas/benchmark_static/index.html` | 39.8KB | 竞品对阵分析前端单页，提供国家选择、AI 对阵推荐、SPEC 对比表、雷达图、AI 结论展示 | 无（静态页面） |
| `竞品对阵分析_部署包/overseas/benchmark_static/echarts.min.js` | 1029KB | ECharts 图表库（用于雷达图渲染） | 无（第三方库） |
| `compintel-deploy/overseas/compintel/dashboard.html` | 18.4KB | 竞品上市预警看板前端单页（内联 CSS），展示认证记录统计、告警列表、AI 解读面板 | stats 卡片、toolbar、ai-panel、records 表格 |

---

## 七、配置文件

| 文件路径 | 大小 | 功能说明 | 核心内容 |
|---|---|---|---|
| `海外数据爬取_部署包/CRAWL_CONFIG.json` | — | 全局抓取配置汇总（v2.1），声明 74 个站点的元数据（code/name/country/base_url/currency/protection/capabilities/search/price/reviews/pagination），供前端与脚本读取 | platforms 分布表、sites 数组（每站 search/price/reviews/pagination 段） |
| `海外数据爬取_部署包/requirements.txt` | 1.2KB | Python 依赖清单：核心零第三方依赖；playwright（可选抓取）、argostranslate（可选离线翻译）、openpyxl（可选 xlsx 导出） | 4 个可选依赖及安装说明 |
| `海外数据爬取_部署包/pyproject.toml` | 1.9KB | Python 项目元数据（overseas-crawler v0.3.0），声明可选依赖组（browser/xlsx/translate/translate-nllb/http）与打包的 sites 子包列表 | `[project]`, `[project.optional-dependencies]`, `[project.scripts]`（overseas = overseas.cli:main） |
| `compintel-deploy/pyproject.toml` | 1.9KB | compintel 部署包项目元数据（同上结构） | 同上 |
| `竞品对阵分析_部署包/pyproject.toml` | 1.9KB | 对阵分析部署包项目元数据（同上结构） | 同上 |
| `竞品对阵分析_部署包/requirements.txt` | 1.2KB | 对阵分析依赖清单 | 同上 |
| `海外数据爬取_部署包/data/retail_allowlist.json` | — | 零售站合规白名单（外部配置，供 retail_compliance 加载合并） | 域名白名单数组 |
| `海外数据爬取_部署包/schemas/declarative/*.schema.json` | — | 声明式 AdapterSpec 规范文件（每站一份，如 amazon_ca.schema.json、kakaku_jp.schema.json 等），与 sites/<code>/schema.json 对应 | 无（声明式配置） |
| `海外数据爬取_部署包/schemas/AdapterSpec_schema规范.md` | — | AdapterSpec 声明式 schema 规范文档 | 无（规范说明） |
| `海外数据爬取_部署包/schemas/RetailSpec_schema规范.md` | — | RetailSpec 声明式 schema 规范文档 | 无（规范说明） |

**关键文档（>1KB，部署/交接/说明类）**：
| 文件路径 | 大小 | 功能说明 |
|---|---|---|
| `海外数据爬取_部署包/DEPLOY.md` | 6.1KB | 部署与执行指南：环境准备、环境变量、执行完整抓取流程 |
| `海外数据爬取_部署包/MASTER_INVENTORY.md` | — | 项目资产清单（自动生成）：74 站点、91 RetailSpec、116 Adapter，含平台分布与数据量统计 |
| `海外数据爬取_部署包/CHANGELOG.md` | — | 变更日志 |
| `海外数据爬取_部署包/docs/项目总说明.md` | — | 项目总说明 |
| `海外数据爬取_部署包/docs/部署说明.md` | — | 部署说明 |
| `海外数据爬取_部署包/docs/使用说明.md` | — | 使用说明 |
| `海外数据爬取_部署包/docs/接口说明.md` | — | API 接口说明 |
| `海外数据爬取_部署包/docs/跨国家线复用与开发指南.md` | — | 跨国家线复用与开发指南 |
| `海外数据爬取_部署包/docs/LLM自动适配器技术方案.md` | — | LLM 自动适配器技术方案 |
| `海外数据爬取_部署包/docs/零售线多国自动适配技术方案.md` | — | 零售线多国自动适配技术方案 |
| `compintel-deploy/DEPLOY.md` | 6.1KB | compintel 部署指南 |
| `compintel-deploy/FUNCTIONALITY.md` | 8.7KB | compintel 功能说明 |
| `compintel-deploy/REPORT.md` | 8.3KB | compintel 报告 |
| `compintel-deploy/WEEKLY_REPORT.md` | 8.4KB | compintel 周报 |
| `compintel-deploy/TODO.md` | 7.4KB | compintel 待办 |
| `竞品对阵分析_部署包/README.md` | 17.2KB | 对阵分析服务说明文档 |

---

## 附录：架构说明与统计

### 分层架构

```
海外/overseas/
├── scenarios.py        场景编排（S1搜索/S2监控/S3直采/S4评价增量/SPEC周采）
├── sites/              站点适配插件（自动发现，含声明式抽取 schema）
│   ├── __init__.py     SiteAdapter 基类 + SiteRegistry 注册中心
│   ├── <code>/adapter.py   继承 SiteAdapter，给 URL 规则
│   ├── <code>/schema.json  声明式抽取规则（选择器写这里）
│   └── *_common.py     跨站共用逻辑（amazon/walmart/costco 价格区校验）
├── extract.py          声明式抽取引擎（多选择器兜底 + transforms）
├── fetchers.py         抓取层（HttpFetcher / BrowserFetcher + 浏览器复用）
├── infra.py            基础设施（域名限速 / 反爬识别 / 缓存 / 代理）
├── db.py               存储层（SQLite，价格与评分按时序快照只增不改）
├── config.py           全局配置（OVERSEAS_ 环境变量覆盖）
├── api/                HTTP 服务（service/server/jobs/exporter + 静态前端）
├── compintel/          认证智能体（FCC/EPREL/ENERGY STAR/ANATEL/IC-ISED/TÜV）
└── spec_*/retail_*     SPEC 与零售线工具（探测/生成/解析/导出/健康/合规）
```

### 三大部署形态

1. **海外数据爬取_部署包**：完整框架 + 全球 200+ 站点适配器 + LangGraph 全流程 + DeepResearch 报告 + benchmark 对阵服务。覆盖日/北美/南美/欧洲/亚洲五线，74 站点、116 适配器、9373 商品、143169 价格快照、146189 评价。
2. **竞品对阵分析_部署包**：独立部署的竞品对阵分析服务，专注国家选择→AI 对阵推荐→SPEC 对比→雷达图→AI 结论，默认监听 127.0.0.1:8090。
3. **compintel-deploy**：竞品认证情报系统，抓取全球 6 类认证数据库并识别竞品上市预警信号，含 LLM 解读与可视化看板。

### 数据规模（来自 MASTER_INVENTORY.md）

| 指标 | 数值 |
|---|---|
| 站点总数（DB） | 74 |
| RetailSpec 文件数 | 91 |
| Adapter 文件数 | 116 |
| 商品总数（product） | 9373 |
| 价格快照总数 | 143169 |
| 评价总数 | 146189 |
| 平台分布 | VTEX 17、Amazon 8、Magento 11 等 |

### 工程红线（来自 CLAUDE.md）

- 长时爬取**必须** `--save-db`、**串行**（单出口 IP）、先备份 DB、用 supervisor 断点续抓。
- "缺价 > 错价"：宁留空不造假价。
- 失败型号（failed/no_item/no_price/no_reviews）直接跳，不无限重试。
- 环境变量：`OVERSEAS_PROXY`（出口代理，不设直连失败）、`OVERSEAS_USE_PROFILE=1`（持久化浏览器 profile）、`PYTHONIOENCODING=utf-8`。

---

> 本文档生成于代码扫描，如需更新某个文件的详细说明，可针对该文件重新读取并补充。文件大小单位为字节（除特别标注 KB/MB 外），实际字节数见各表格"大小"列。
