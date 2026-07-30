# TrendRadar AI 日报 MVP 部署手册

> 给项目经理（Furman_li）的部署文档，语言尽量不涉及技术黑话，必要时加括号解释。
> 基线版本：TrendRadar 6.10.0（fork: lfb511/TrendRadar）
> 工作分支：`mvp/trendradar-ai-daily`
> 配套评审稿：`MVP_SPEC.md`（需求评审稿，讲"为什么这么做"）

---

## 1. 概述

MVP 的目标是验证一条完整的链路：

**资讯抓取 → 关键词筛选 → HTML 日报生成 → GitHub Pages 展示 → 飞书机器人推送 →（阶段B）AI 生成中文日报**

- 代码基于 TrendRadar 6.10.0 的 fork（仓库 `lfb511/TrendRadar`）。
- 所有改动都在 `mvp/trendradar-ai-daily` 这条分支上，不污染主分支 `main`。
- 分两个阶段：阶段A 只做"抓取 + 筛选 + HTML + Pages + 飞书推送"，不花 AI 的钱；阶段A 全部通过后，才开阶段B 接入智谱 GLM 做中文日报。
- 部署方式：GitHub Actions（GitHub 提供的免费定时任务，每小时自动跑一次），不需要自建服务器。

---

## 2. 阶段A 部署步骤（按顺序，可照做）

> 前提：代码已经提交到分支 `mvp/trendradar-ai-daily`（本仓库已经是这个状态）。

### 步骤 1：确认代码在正确分支
- 在 GitHub 仓库页面左上角的分支下拉框里，确认当前是 `mvp/trendradar-ai-daily`。
- 不需要额外操作。

### 步骤 2：配置飞书推送 Secret（阶段A 只配这一个）
1. 进入 GitHub 仓库页面，点 **Settings**（仓库设置，不是账号设置）。
2. 左侧菜单点 **Secrets and variables** → **Actions**。
3. 点右上角 **New repository secret**（新建仓库密钥）。
4. 填写：
   - **Name**（名称）：`FEISHU_WEBHOOK_URL`
   - **Value**（值）：你的飞书机器人 webhook URL，形如 `https://open.feishu.cn/open-apis/bot/v2/hook/xxxxxxxx`
5. 点 **Add secret** 保存。
- 阶段A 只需要配这一个 Secret，AI 相关 Secret 留到阶段B 再配。

> 飞书机器人怎么创建见下方第 3 节，先创建拿到 URL 再回来填这里。

### 步骤 3：开启 GitHub Pages
1. 进入仓库 **Settings** → 左侧 **Pages**。
2. **Source**（来源）选 **Deploy from a branch**（从分支部署）。
3. **Branch**（分支）选 `mvp/trendradar-ai-daily`，右边文件夹选 **Root**（根目录）。
4. 点 **Save**。
- 注意：Pages 第一次会显示"没有内容"，这是正常的，需要先跑一次下面的 workflow 生成 `index.html`，Pages 才有东西可展示。

### 步骤 4：手动跑一次 workflow
1. 进入仓库 **Actions**（顶部标签页）。
2. 左侧 workflow 列表里选 **Get Hot News**。
3. 点右上角 **Run workflow**（运行工作流）→ 确认分支是 `mvp/trendradar-ai-daily` → 点绿色 **Run workflow** 按钮。
4. 等待运行完成（大约 2–5 分钟），看到绿色对勾（✓）即成功。
- 成功后，仓库根目录会多出一个 `index.html`，GitHub Pages 几分钟内就能访问了。

---

## 3. 创建飞书机器人（详细）

1. 打开飞书，进入你要接收推送的群（单人群也行，自己拉自己进一个空群）。
2. 点群右上角的设置图标（⚙️）→ 找到 **群机器人** → 点 **添加机器人** → 选 **自定义机器人**。
3. **名字**：填"AI日报"（随意，自己认得就行）。
4. **描述**：随意填。
5. **安全设置**：选 **自定义关键词**，关键词填 `日报`。
   - 为什么：飞书机器人会校验消息内容里必须含这个关键词才放行，我们的日报内容一定含"日报"二字；不设这个飞书会拒收。
6. 点 **完成/创建** → 会显示一段 **webhook URL**，形如 `https://open.feishu.cn/open-apis/bot/v2/hook/xxxxxxxx`。
7. **复制这个 URL** → 回到第 2 节步骤 2 填进 GitHub Secret。
- 重要：这个 URL 只能填进 GitHub Secrets，**不要**贴进任何代码文件、**不要**发给别人、**不要**截图发群里。它相当于机器人的密码，谁拿到就能往你群里发消息。

---

## 4. GitHub Secrets 清单

GitHub Secrets 是 GitHub 提供的"加密保管箱"，存进去的值代码看不到明文，只能在 workflow 运行时引用，不会泄露。

| 阶段 | Secret 名 | 用途 | 值 | 何时配 |
|---|---|---|---|---|
| A | `FEISHU_WEBHOOK_URL` | 飞书推送地址 | 飞书机器人 webhook | 阶段A 部署时 |
| B | `AI_ANALYSIS_ENABLED` | AI 分析开关 | `true` | 阶段B |
| B | `AI_MODEL` | AI 模型名 | `openai/glm-5.2`（⚠️ 必须带 `openai/` 前缀，因用了自定义端点） | 阶段B |
| B | `AI_API_KEY` | 智谱 API Key | 你的智谱 Key | 阶段B |
| B | `AI_API_BASE` | 智谱 API 地址 | `https://open.bigmodel.cn/api/coding/paas/v4`（⚠️ Coding Plan 用此端点，非 `/paas/v4`） | 阶段B |

> **端点说明**：智谱普通用户用 `https://open.bigmodel.cn/api/paas/v4`；**Coding Plan（编码计划）订阅用户必须用 `/api/coding/paas/v4`**，用错会报"余额不足或无可用资源包"。本次实测 Coding Plan Key 在普通端点报余额不足，在 coding 端点正常。

> **红线**：Secret 的值（webhook URL、API Key 等）**绝不写入仓库任何文件**，不进 git commit。如果写进文件并提交，相当于公开泄露，必须立即吊销重置。

---

## 5. 阶段A 验收清单（8 项，V1–V8）

手动跑完第 2 节的 workflow 后，对照下表逐项检查，全部打勾才算阶段A 通过：

- **V1**：workflow 成功完成（Actions 页面显示绿色对勾 ✓）。
- **V2**：点进 Actions 运行日志，没有 Python 异常或 Traceback（一大段红色报错堆栈）。
- **V3**：仓库根目录生成了 `index.html`（在 workflow 的 artifacts 里能下载，或直接在分支文件列表里看到）。
- **V4**：GitHub Pages 能访问，地址是 `https://<你的GitHub用户名>.github.io/TrendRadar/`。
- **V5**：打开 Pages 页面，能看到筛选后的新闻；四个关键词组（AI模型 / AI开发工具 / 制造业数字化 / 政策与产业）每组都有命中条目。
- **V6**：飞书群收到了推送消息。
- **V7**：git log（提交记录）里搜不到 webhook / api_key / password / token 明文。
  - 检查命令：`git log -p | grep -iE "webhook|api[_-]?key|password|token|secret"`，应该没有输出。
- **V8**：Actions 运行日志里能看到各关键词组实际命中了多少条。

> V1、V2 未通过，说明链路没跑通，**不要进入阶段B**。

---

## 6. 阶段B 简述（阶段A 全部通过后才做）

> **状态：本地端到端验证已通过**（2026-07-30）。GLM-5.2 调用成功，AI 日报质量达标（含制造业视角、售前建议），飞书推送成功。剩 GitHub Secrets 配置 + Actions 触发。

阶段B 是把抓到的新闻交给智谱 GLM 大模型，生成一份中文的、有分析的日报。步骤大致是：

1. **先本地验证 GLM API 能通**（重要前置）：
   - 用一个最小的 Python 脚本，按 OpenAI 兼容格式请求一次智谱 GLM，确认三件事能对上：模型名、API Key、Base URL（`https://open.bigmodel.cn/api/paas/v4`）。
   - 本地能跑通，再去配 GitHub Secrets，避免在 GitHub Actions 里反复试错。
2. **改 config.yaml 开启 AI 分析**（需要改若干行，比如 `ai_analysis.enabled` 改 true、模式改 `daily`、语言保持中文、分析条数上限改 40、包含 RSS 源等）。
3. **轻调 prompt（提示词）文件** `config/ai_analysis_prompt.txt`：
   - 保持官方结构，不重写，只在里面加上"制造业视角"和"给项目经理的行动建议"。
4. **配 4 个 AI Secrets**（见第 4 节表格的 B 段）。
5. **跑 workflow 验收 B1–B8**（8 项验收，包含 AI 调用成功、中文输出、引用了本次新闻、Key 不泄露到日志等）。

> 阶段B 详细规格见 `MVP_SPEC.md` 第 5 节。

---

## 7. 已知问题与风险（重点看清楚）

- **R1：7 天试用续期**。
  - GitHub Actions 的免费时长有个机制：连续 7 天不点一次 `Check In`（签到）workflow，会自动把 `Get Hot News`（抓取）workflow 关掉，抓取会静默停止。
  - 代码依据：`.github/workflows/crawler.yml` 第 62–94 行。
  - 缓解：**每 ≤6 天去仓库 Actions 页手动点一次 `Check In` workflow 的 Run workflow**，建议在手机日历设一个每 6 天提醒一次的循环提醒，别忘。
- **R2：每小时运行（阶段B 已开 AI，此风险已激活⚠️）**。
  - 当前定时任务设置是 `33 * * * *`，即每小时的第 33 分钟跑一次。
  - 阶段B 已开启 AI 分析，每小时会调一次智谱 GLM，**一天调 24 次**，会持续消耗 Coding Plan 额度。
  - **强烈建议**：阶段B 通过后，把 cron 改成 `10 0 * * *`（每天 UTC 00:10，即北京时间 08:10），一天只跑一次。改法：编辑 `.github/workflows/crawler.yml` 第 38 行。
  - 注意：Coding Plan 额度通常有上限，24 次/天的 daily 分析可能较快耗尽。
- **R3：关键词命中量未知**。
  - 首次运行时，可能某个关键词组一条都抓不到（命中 0 条）。
  - 缓解：看 Actions 日志里各组的实际命中数，按需要扩充关键词。
- **R4：GitHub Pages 国内访问慢**。
  - 国内打开 Pages 页面可能比较慢。
  - 缓解：MVP 阶段不上 Cloudflare（云加速服务），后续可以考虑加。
- **R5：fork 改了 config 后难 pull 上游**。
  - 我们改了配置文件，官方仓库后续更新时手动合并会比较麻烦。
  - 缓解：MVP 不改核心 Python 代码，只动配置文件，影响较小。

---

## 8. 回滚方式

每类改动都有对应的"撤销"动作，按需使用：

| 改动 | 回滚动作 |
|---|---|
| config.yaml 的 2 行改动 | `git checkout main -- config/config.yaml`（从主分支恢复这个文件） |
| frequency_words.txt | `git checkout main -- config/frequency_words.txt` |
| 整个分支 | 先切回主分支：`git checkout main`；删本地分支：`git branch -D mvp/trendradar-ai-daily`；删远程分支：`git push origin --delete mvp/trendradar-ai-daily` |
| GitHub Secrets | 仓库 **Settings → Secrets →** 找到对应项，点 **Remove / 删除** |
| 飞书机器人 | 飞书群设置 → 群机器人 → 找到"AI日报"机器人 → **移除** |
| GitHub Pages | 仓库 **Settings → Pages → Source** 改为 **Disable** |

---

## 9. 下一阶段建议

阶段B 全部通过后，可以考虑以下优化：

- **降频**：把 cron 从每小时改为每天 08:10（修改 `.github/workflows/crawler.yml` 第 38 行，改成 `10 0 * * *`），避免每小时烧 GLM 额度。
- **接制造业垂直 RSS 源**：目前抓的是 Hacker News、雅虎财经等通用资讯源，对"制造业数字化"覆盖偏少，后续可接一些制造业垂直媒体或行业 RSS，提升命中率。
- **加 Cloudflare Pages 加速**：用 Cloudflare（云加速服务）代理 GitHub Pages，让国内打开更快。
- **长期使用考虑 Docker 部署**：摆脱 GitHub Actions 的 7 天试用续期限制，自建服务器跑，稳定可控（但这超出了 MVP 范围，是后续产品化的事）。

---

## 附：文档维护说明

- 本文档只描述"怎么部署、怎么验收、出问题怎么回滚"，需求背景见 `MVP_SPEC.md`。
- 若部署过程中发现某步骤和实际不符，以 GitHub / 飞书 / 智谱官方当前界面为准，并回来更新本文档。
- 所有截图位置（如 Settings、Actions 页面布局）以 GitHub 当前版本为准，GitHub 偶尔会改版。
