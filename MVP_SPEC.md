# TrendRadar AI 日报 MVP 需求评审稿

> 状态：待用户确认（to-spec 阶段，未进入实施）
> 基线版本：TrendRadar 6.10.0（fork: lfb511/TrendRadar）
> 编写日期：2026-07-30
> 适用范围：本评审稿仅约束 MVP 第一、二阶段，不涉及长期产品化。

---

## 0. 评审稿用途

本文档是把 GPT 给出的搭建建议，对照 TrendRadar 6.10.0 真实代码逐项核对后，固化的**可执行规格**。GPT 原文里有若干与真实代码不符或自相矛盾的地方（见第 6 节"差异与纠偏"），一律以本评审稿为准。

**确认本稿后，才会进入实施；实施只做本稿列出的改动，不扩展。**

---

## 1. 背景与目标

### 1.1 业务背景
Furman_li（浙江国坤智能科技 项目经理/售前顾问）希望搭建一个最小可用的 AI 日报系统，用于每日跟踪 AI 模型、AI 开发工具、制造业数字化、政策与产业四个方向的资讯，辅助售前和项目跟进。

### 1.2 MVP 目标
验证完整链路：**资讯抓取 → 关键词筛选 → HTML 日报生成 → GitHub Pages 展示 → 飞书机器人推送 → （阶段B）AI 生成中文日报**。

### 1.3 不做的事（边界）
- 不部署 Docker、不启用 MCP Server、不接远程存储（R2/OSS/COS/S3）、不接 Cloudflare。
- 不修改 trendradar 核心 Python 代码。
- 不做用户登录、管理后台、页面视觉改造。
- 不同时接多个推送渠道（只飞书）。
- 不开 AI 智能筛选、AI 全文阅读、AI 翻译。
- 不重构项目目录、不改页面视觉风格。

---

## 2. 现场事实清单（grilling 已确认的决策）

| # | 决策点 | 确认结果 |
|---|---|---|
| F1 | 代码来源 | 用户已 Fork 到 `lfb511/TrendRadar`，本地 clone 进来工作 |
| F2 | 密钥安全等级 | **严格**：零密钥进仓库，提交前本地 secret 扫描，全部走 GitHub Secrets |
| F3 | 关键词语法风险 | **先读源码验证语法再写**（已完成，见第 4.2 节） |
| F4 | 调度策略 | **保留官方默认每小时跑**（用户接受代价）；阶段A 强制 AI 关闭做金钱护栏 |
| F5 | 推送渠道 | **飞书机器人（唯一）**，Secret 名 `FEISHU_WEBHOOK_URL` |
| F6 | 条数限制 | 用 `@N` 限制每组（8/8/8/5），全局 `max_news_per_keyword` 保持 0 |
| F7 | 试用续期机制 | 保留官方机制，文档提示用户每 ≤6 天点一次 `Check In` |
| F8 | 工作分支 | `mvp/trendradar-ai-daily` |

---

## 3. 现场事实核对结果（我替你查的，来自真实代码）

### 3.1 Secrets 真实名称（来自 `.github/workflows/crawler.yml:138-162` + `trendradar/core/loader.py`）

| 用途 | 真实 Secret 名 | 代码依据 | GPT 是否给出 |
|---|---|---|---|
| AI 分析开关 | `AI_ANALYSIS_ENABLED` | loader.py:281，**布尔型**，未设时回退 config.yaml | ✅ |
| AI 模型 | `AI_MODEL` | loader.py:261 | ✅ |
| AI Key | `AI_API_KEY` | loader.py:262 | ✅ |
| AI Base URL | `AI_API_BASE` | loader.py:263 | ✅ |
| **飞书 webhook** | `FEISHU_WEBHOOK_URL` | crawler.yml:139，loader.py:413 | ❌ GPT 漏了，本稿补上 |

### 3.2 关键护身符（loader.py:284）
```python
"ENABLED": enabled_env if enabled_env is not None else ai_config.get("enabled", False)
```
含义：`AI_ANALYSIS_ENABLED` 这个 Secret **不设时，回退 config.yaml 的 `ai_analysis.enabled`**。所以只要 config.yaml 设 false，即使 Secret 被误填，AI 也不会开启——这正好支撑 F4（每小时跑但不烧钱）。

### 3.3 config.yaml 现状 vs GPT 阶段A 要求

| GPT 要求 | 当前默认值 | 是否需改 |
|---|---|---|
| 时区 Asia/Shanghai | Asia/Shanghai | ✅ 已满足 |
| 关闭调度系统 | `schedule.enabled: false` | ✅ 已满足 |
| 报告模式 current | `current` | ✅ 已满足 |
| 筛选方式 keyword | `keyword` | ✅ 已满足 |
| 关闭 AI 分析 | `ai_analysis.enabled: true` | ❌ **必须改 false** |
| 关闭 AI 翻译 | `ai_translation.enabled: true` | ❌ **必须改 false** |
| 保留 HTML 报告 | `storage.formats.html: true` | ✅ 已满足 |
| 本地 SQLite 存储 | `storage.backend: auto`（GHA 下→local） | ✅ 已满足 |
| 不配远程存储 | remote 段全空 | ✅ 已满足 |

**结论：阶段A config.yaml 只需改 2 行。**

---

## 4. 阶段A 功能规格（实施清单）

### 4.1 配置改动（config/config.yaml）

**只改 2 行，局部编辑，不重写：**

| 行 | 字段 | 原 → 新 |
|---|---|---|
| 414 | `ai_analysis.enabled` | `true` → `false` |
| 443 | `ai_translation.enabled` | `true` → `false` |

其余字段全部保持官方默认（含 11 个热榜平台、3 个 RSS、storage local、report.mode=current、filter.method=keyword）。

**显式保留不动**（符合 GPT "关闭调度" 精神 + 用户 F4 选择）：
- workflow cron 保留官方 `33 * * * *`（每小时第33分钟）
- `schedule.enabled` 保持 false

### 4.2 关键词配置（config/frequency_words.txt）

**语法已用源码核对**（`frequency_words.txt` 头部说明 + `trendradar/core/frequency.py` 解析逻辑）：
- `[GLOBAL_FILTER]` / `[WORD_GROUPS]` 分区标记 ✓
- `/正则/` ✓（自动忽略大小写）
- `=> 别名` ✓（GPT 原稿漏了别名，会显示丑陋长正则，本稿补上）
- `@数字` 单独成行 ✓
- `[组别名]` ✓

**重写为以下内容**（替换现有词组，保留文件头部说明）：

```text
[GLOBAL_FILTER]
震惊

[WORD_GROUPS]

[AI模型]
/\bOpenAI\b|\bChatGPT\b|\bClaude\b|\bAnthropic\b|\bGemini\b|\bDeepSeek\b|\bGLM\b|智谱|通义千问|\bQwen\b|\bKimi\b|\bMiniMax\b|大模型/ => AI模型动态
@8

[AI开发工具]
/\bCodex\b|Claude Code|\bCursor\b|\bWindsurf\b|GitHub Copilot|\bMCP\b|AI Agent|智能体|多智能体|\bRAG\b/ => AI开发工具
@8

[制造业数字化]
/智能制造|工业软件|数字化工厂|\bMES\b|\bWMS\b|\bAPS\b|\bPMC\b|\bQMS\b|\bSPC\b|\bIoT\b|生产计划|供应链数字化/ => 制造业数字化
@8

[政策与产业]
/工信部|工业和信息化|人工智能政策|智能制造政策|工业互联网|新型工业化|浙江制造|温州制造/ => 政策与产业
@5
```

> 注：`@N` 与全局 `max_news_per_keyword: 0` 的关系——全局 0=不限制，由各组 `@N` 分别控制。这符合官方示例，也解决 GPT "5 vs 8" 的自相矛盾。

### 4.3 GitHub Secrets（阶段A 仅 1 个）

| Secret 名 | 值 | 来源 |
|---|---|---|
| `FEISHU_WEBHOOK_URL` | 飞书自定义机器人 webhook URL | 用户在飞书群创建机器人后获得，**不写入任何文件** |

### 4.4 阶段A 验收清单（7 项）

手动运行 `Get Hot News` workflow 后，必须全部通过：

- [ ] V1. workflow 成功完成（绿色）
- [ ] V2. 无 Python 异常
- [ ] V3. 仓库根目录生成 `index.html`
- [ ] V4. GitHub Pages 能访问（Settings → Pages 开启，source 选主分支/index.html）
- [ ] V5. 页面显示筛选后的新闻（四个关键词组各有命中）
- [ ] V6. 飞书群收到推送消息
- [ ] V7. 提交记录 grep 无 webhook/key/password（本地 secret 扫描通过）
- [ ] V8. 输出实际匹配数量（Actions 日志里看各关键词组命中条数）

> V1-V2 未过 → 不进入阶段B。

---

## 5. 阶段B 功能规格（阶段A 通过后）

### 5.1 前置：GLM API 本地验证（GPT 5.2 强制）
**在写任何 GitHub Secret 前**，本地跑最小 Python 脚本，用 OpenAI 兼容格式请求一次 GLM，确认：
- 智谱账号实际可用模型名（不猜测，不沿用旧名）
- API Key 能用于普通 API 调用
- OpenAI 兼容 Base URL（智谱为 `https://open.bigmodel.cn/api/paas/v4`）

验证通过后才配 TrendRadar。

### 5.2 阶段B 配置改动

**config/config.yaml**：
- `ai_analysis.enabled: false` → `true`
- `ai_analysis.mode: follow_report` → `daily`
- `ai_analysis.language: Chinese`（已是）
- `ai_analysis.max_news_for_analysis: 150` → `40`
- `ai_analysis.include_rss: false` → `true`
- `ai_analysis.include_standalone: true` → `false`
- `ai_analysis.include_rank_timeline: true` → `false`
- `ai_translation.enabled` 保持 `false`（不翻译）
- `filter.method` 保持 `keyword`（不启用 AI 智能筛选）
- `report.mode: current` → `daily`

**config/ai_analysis_prompt.txt**：保持官方结构，做轻量调整（不重写），让日报包含：
1. 今日最重要的 3–5 条 AI 动态
2. AI 模型与开发工具更新
3. 对制造业数字化（MES/WMS/APS/PMC）的潜在影响
4. 值得进一步研究的 GitHub 项目或技术方向
5. 对项目经理的行动建议
6. 明确区分事实 / 分析判断 / 建议，不输出空泛口号

### 5.3 阶段B GitHub Secrets（新增 4 个）

| Secret 名 | 值 | 说明 |
|---|---|---|
| `AI_ANALYSIS_ENABLED` | `true` | 开启 AI 分析 |
| `AI_MODEL` | `openai/<实际模型名>` | 智谱走 OpenAI 兼容，前缀加 `openai/`（见 config.yaml:382-388 说明） |
| `AI_API_KEY` | 智谱 API Key | 不写入文件 |
| `AI_API_BASE` | `https://open.bigmodel.cn/api/paas/v4` | 智谱 OpenAI 兼容地址 |

> 飞书 webhook 沿用阶段A 的 `FEISHU_WEBHOOK_URL`，不新增。

### 5.4 阶段B 验收清单（8 项）

- [ ] B1. AI 接口调用成功（Actions 日志无 4xx/5xx）
- [ ] B2. AI 日报使用中文
- [ ] B3. 日报不是简单罗列标题（有分析、有结构）
- [ ] B4. AI 分析内容能引用本次抓取到的新闻
- [ ] B5. 单次分析不超过 40 条新闻（日志确认输入条数）
- [ ] B6. 飞书推送无明显截断或乱码
- [ ] B7. Actions 日志不输出完整 API Key
- [ ] B8. 记录本次模型名、输入数量、输出结果、异常情况

---

## 6. 差异与纠偏（本稿 vs GPT 原稿）

| # | GPT 原稿说法 | 真实情况 / 本稿处理 |
|---|---|---|
| D1 | 关键词正则不加别名 | 语法能跑但推送显示丑陋长正则；本稿给每组加 `=> 别名` |
| D2 | "每组最多5条" + 关键词写 `@8` + 全局 `max_news_per_keyword` | 三套规则冲突；本稿用 `@N` 分组控制，全局保持 0 |
| D3 | 推送优先企业微信，列出 `WEWORK_*` Secrets | 用户改用飞书；本稿改 `FEISHU_WEBHOOK_URL`，企业微信 Secrets 清单作废 |
| D4 | 飞书 Secret 名未给 | 本稿补 `FEISHU_WEBHOOK_URL`（crawler.yml:139 实证） |
| D5 | "关闭调度系统" | 官方 `schedule.enabled` 默认已 false；但 workflow cron 默认每小时跑；用户选保留每小时（F4），阶段A AI 关闭做护栏 |
| D6 | 未提 7 天试用续期机制 | crawler.yml:62-94 有自动 disable 机制；本稿列入已知问题，提示每 ≤6 天点 `Check In` |
| D7 | 阶段A 列了一堆 `AI_*` Secrets | 阶段A 不开 AI，本稿阶段A 只配飞书 1 个 Secret；AI Secrets 全部移到阶段B |
| D8 | 未提 `AI_MODEL` 需加 `openai/` 前缀 | config.yaml:382-388 明确：用 api_base 时 model 须加 `openai/` 前缀；本稿阶段B 已纳入 |

---

## 7. 已知问题与风险

| # | 问题 | 影响 | 缓解 |
|---|---|---|---|
| R1 | 7 天试用续期 | 到期不点 Check In，抓取静默停止 | 文档提示 + 建议设日历提醒（每 6 天） |
| R2 | 每小时跑 + 阶段B 开 AI | 每小时烧一次 GLM 额度（一天 24 次） | 阶段B 通过后，建议改 cron 为 `10 0 * * *`（每天北京 08:10） |
| R3 | 关键词真实命中量未知 | 阶段A 首跑可能某组 0 命中 | 首跑后看日志，按需扩词 |
| R4 | GitHub Pages 访问慢（国内） | 页面打开慢 | MVP 不上 Cloudflare，后续可加 |
| R5 | fork 改 workflow 后难 pull 上游 | 官方更新难合并 | MVP 不改 workflow 核心，只动 config |

---

## 8. 回滚方式

| 改动 | 回滚动作 |
|---|---|
| config.yaml 2 行 | `git checkout main -- config/config.yaml` 或删分支回到 main |
| frequency_words.txt | `git checkout main -- config/frequency_words.txt` |
| GitHub Secrets | 仓库 Settings → Secrets → 删除对应项 |
| 飞书机器人 | 飞书群设置 → 群机器人 → 移除 |
| GitHub Pages | Settings → Pages → Source 改 Disable |

整分支回滚：`git checkout main && git branch -D mvp/trendradar-ai-daily`（远程同步删除）。

---

## 9. 最终交付物（阶段B 通过后输出）

1. 修改文件清单
2. 每个文件的修改原因
3. GitHub Secrets 清单（不输出值）
4. 手动运行步骤
5. GitHub Pages 访问方式
6. Actions 运行结果
7. 日报截图或输出样例
8. 当前已知问题
9. 回滚方式
10. 下一阶段建议

提交前做一次安全检查：`git log -p | grep -iE "webhook|api[_-]?key|password|token|secret"` 确认无密钥泄露。

---

## 10. 待用户确认

> **请确认以下三点后，我进入实施（创建分支 → 改 config → 写 frequency_words → 推送）：**

- [ ] C1. 第 4 节阶段A 规格（config 改 2 行 + 关键词四组 + 飞书 1 个 Secret）OK
- [ ] C2. 第 5 节阶段B 规格（含 GLM 本地验证前置 + 4 个 AI Secrets）OK
- [ ] C3. 第 6 节对 GPT 原稿的 8 处纠偏（尤其 D2 条数、D3 渠道、D6 续期）你认可

有异议的条款请直接指出，我改稿不改代码。
