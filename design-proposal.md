# OpenClaw AI Bot 小队 · 人格 + 群名 + 工作区 全命名体系

> 状态：**设计稿 v4（2026-08-12 落地完稿）** — 三件套全部落地：人格（IDENTITY）+ 群名 + 工作区目录。
> 覆盖：人格名（IDENTITY / 显示名）+ 飞书群名 + 工作区目录名，三件同构、一套体系。
> 配套：落地流程、验证与坑见 `runbook-2026-08-12.md`（两者互补）。

---

## 一、现状总表（v3 已按实测更新）

| bot | 工作区(目录) | memory量 | 飞书群名(落地) | 群chat_id | 现人格名 |
|---|---|---|---|---|---|
| **main** | `workspace/` | 345M（memory 子目录 76K，跨 2025-06→2026-07；"10天"存疑，待明确口径） | **OpenClaw·阿岩** | `oc_35c6...a76d` | ❌无（IDENTITY.md 空模板） |
| **newbot** | `workspace-newbot/` | 328K | **OpenClaw·小七** | `oc_83ff...32c3` | 老七（盘上，待改 小七） |
| **bot2** | `workspace-bot2/` | 304K | **OpenClaw·老六** | `oc_ebec...20d5` | 老六 |
| **bot3** | `workspace-bot3/` | 464K | **OpenClaw·阿叁** | `oc_ffe9...4031` | 007（盘上，待改 阿叁） |

> 群名 4 个已用 PUT 改名落地（v2 稿的「OpenClaw Agent 群/OpenClaw-newbot/…」为改名前快照，已过时）。
> 人格名与群名/账号显示名**已统一**（2026-08-12 人格文件落地：老七→小七、007→阿叁、main 空模板→阿岩），详见 runbook。

---

## 二、命名体系 · 三件套统一

### 命名哲学
- 人格名 = **单字+昵称**（阿岩/小七/老六/阿叁），一声呼出、成组好记。
- 群名 = **「OpenClaw · 名」**：一眼看出是 OpenClaw 的团队群，且对到每个 bot。
- 工作区 = **`workspace-<rolename>`**：技术名不参杂中文，且尽量和人格名/职责对应，方便脚本与 grep。

### 完整对照表（v4：群名列、人格名列、工作区列全部已落地）

| bot | 人格名 | 群名 | **工作区（已落地）** | 职责 |
|---|---|---|---|---|
| **main** | 阿岩 ⛰️ | OpenClaw · 阿岩 | `workspace-main`（原 `workspace/`） | 总指挥/主助理 |
| **newbot** | 小七 🔧 | OpenClaw · 小七 | `workspace-ops`（原 `workspace-newbot/`） | 运维工 |
| **bot2** | 老六 🐍 | OpenClaw · 老六 | `workspace-lab`（原 `workspace-bot2/`） | 代码实验室 |
| **bot3** | 阿叁 📚 | OpenClaw · 阿叁 | `workspace-lib`（原 `workspace-bot3/`） | 资料库 |

> ✅ **已选定并落地 A 方案**（技术名解耦）：`workspace-main/ops/lab/lib`。目录名与中文人格解耦，grep/脚本/权限不易错。

---

## 三、为什么工作区改名有风险（先讲清楚）

- 目录名被 `openclaw.json` 的 `agents.*.workspace`（第 130/135/141/147 行）**硬编码引用**。
- `workspace/` 是 **Main Agent 默认工作区**（345MB；memory 子目录仅 76K、文件时间跨 2025-06→2026-07，v2 稿"10 天记忆"口径存疑待明确），改名需格外小心。
- bot2/bot3/newbot 虽然才几百 KB，但 **`memory/` 和 `MEMORY.md` 若一起 mv** 会丢失或绑定错乱。

**做法（安全，已执行）：** `mv 旧目录 新目录` 整体搬移 + 同步改 `openclaw.json` 的 workspace 行 → `openclaw doctor` 校验 → 重启 gateway。agentId/bindings 未动（`newbot/bot2/bot3` 路由稳定）。⚠️ 落地核对：main 原本**无** workspace 行（走 `resolveDefaultAgentWorkspaceDir` = `$HOME/.openclaw/workspace`），故给 main **新增显式 workspace 行**后共改 4 行（第 130/135/141/147 行：3 个子 bot 改路径 + main 新增 1 行）；config 备份 `openclaw.json.bak.20260812-rename`。

---

## 四、落地清单

1. **人格文件**（每工作区）：✅ **已全部落地**（v4）
   - `SOUL.md` → 该角色专属完整人格（追加"小队定位"小节）
   - `IDENTITY.md` → 填 Name/Creature/Vibe/Emoji/团队定位（4 份全更新：阿岩/小七/老六/阿叁）
   - `USER.md` → 更新"你/老板"与团队上下文（main 由空模板填写，3 子 bot 补充小队行）
   - `AGENTS.md` → 补充"你是小队一员"（新增 Team 小节）
2. **显示名**（`openclaw.json`，仅人格名部分）：✅ **已落地**（v3 实测）
   - `accounts.newbot.name` → **小七（运维工）**
   - `accounts.bot2.name` → **老六（代码实验室）**
   - `accounts.bot3.name` → **阿叁（资料库）**
   - （default 由群卡片呈现，无需 account name）
3. **群名**（API 改 4 个群的 name）：✅ **已落地**（PUT 改名 OpenClaw·阿岩/小七/老六/阿叁，详见 §五.3 与 runbook §3.3）
4. **工作区**（A 方案）✅ **已落地**：mv + 改 4 行配置（详情见 §三）。

---

## 五、拍板结果（4 项全部定案）

1. **工作区命名** ✅ **已拍板并执行 A 方案**：`workspace-main / workspace-ops / workspace-lab / workspace-lib`（技术名，与人格解耦）。
2. **main 工作区改名** ✅ **已执行**：345M 目录整体 mv；查明 main 默认解析机制（`resolveDefaultAgentWorkspaceDir` = `~/.openclaw/workspace`）后，给 main **新增显式 workspace 行**规避默认路径，再重启验证（`openclaw agents` 确认解析新路径）。
3. **群名** ✅ **已执行落地**：4 群已 PUT 改名 `OpenClaw·阿岩/小七/老六/阿叁`（回滚=API 再改回，流程见 runbook §3.3）。
4. **人格名**（阿岩/小七/老六/阿叁）✅ **全量落地**：群名、账号名、IDENTITY.md 三层一致（老七→小七、007→阿叁 已替换）。
