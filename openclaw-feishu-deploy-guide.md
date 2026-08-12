# OpenClaw + 飞书 部署实战指南

> 版本适用：**OpenClaw 2026.6.34 (extended-stable)** + `@openclaw/feishu 2026.6.34`（2026-08-12 实测回归）
> 本文件是**照做手册**：从零搭建到多 bot 运营，命令可直接复制；所有「坑」均来自实际踩过并验证的案例。
> 配套：`runbook-2026-08-12.md`（过程记录/决策史）、`design-proposal.md`（命名体系）。三者互补。

---

## 目录

1. [部署架构总览](#1-部署架构总览)
2. [前置准备](#2-前置准备)
3. [安装 OpenClaw（版本纪律）](#3-安装-openclaw版本纪律)
4. [飞书开放平台侧准备](#4-飞书开放平台侧准备)
5. [配置骨架（openclaw.json）](#5-配置骨架openclawjson)
6. [多 Bot / 多 Agent / 多工作区](#6-多-bot--多-agent--多工作区)
7. [systemd 常驻服务](#7-systemd-常驻服务)
8. [升级路径（extended-stable 纪律）](#8-升级路径extended-stable-纪律)
9. [机制真相（决定一切行为的底层规则）](#9-机制真相决定一切行为的底层规则)
10. [飞书 API 自助操作（建群/改名/发消息）](#10-飞书-api-自助操作建群改名发消息)
11. [安全基线（共享主机）](#11-安全基线共享主机)
12. [排错速查表（现象 → 根因 → 解法）](#12-排错速查表现象--根因--解法)
13. [验证清单（改造后必跑）](#13-验证清单改造后必跑)

---

## 1. 部署架构总览

```
[飞书开放平台] 4 个自建应用(websocket 长连接)
      │  im.message.receive_v1 事件（长连接推送）
      ▼
[OpenClaw Gateway]  systemd user 服务 · 127.0.0.1:18789 · token 认证
      │  每个飞书应用 = 一个 account（default/newbot/bot2/bot3）
      │  每个 account → 路由到独立 agent（工作区/人格/记忆隔离）
      ▼
[iagent API]  https://iagent.iauto.com/v1 · openai-completions 兼容
      （DeepSeek-V4 系：iagent/standard、DeepSeek-V4-Flash）
```

- **四层**：飞书应用 → Gateway → Agent（工作区+人格）→ 模型 Provider。
- **数据目录**：`$HOME/.openclaw/`（config、agents、workspaces、npm 插件、日志）。
- **进程**：systemd user 服务，`ExecStart` 直指全局 node_modules 的 `dist/index.js gateway --port 18789`。
- **连接模式**：全部 bot 用 **websocket 长连接**（飞书事件订阅方式 = 「使用 长连接 接收事件/回调」），**不需要公网 webhook**。

---

## 2. 前置准备

| 项 | 要求 | 说明 |
|---|---|---|
| OS | Linux（共享 / 独立均可） | 实测 Ubuntu 22.04 系 |
| Node | v24 LTS（nvm 亦可） | `nvm install 24` |
| 网络 | 可达 registry.npmjs.org；可达 iagent API | |
| 内存 | ≥ 1GB 空闲（Gateway 常驻 600MB 级） | 4 bot + 模型调用实测 640MB |
| 飞书开放平台 | 开发者账号 + 自建应用 | 每个 bot 一个应用 |

```bash
# 推荐 nvm（本机已核实的安装方式）
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
nvm install 24 && nvm use 24
node -v   # v24.17.0
```

---

## 3. 安装 OpenClaw（版本纪律）

> ⚠️ **核心纪律：插件与核心必须同版本锁步（peerDependencies 要求 `openclaw >= 同版本`）**。
> 单升插件不受支持；升级 = 核心 + 插件一起。

```bash
# 查看全部版本轨（dist-tags）
curl -s https://registry.npmjs.org/openclaw | python3 -c "
import json,sys; d=json.load(sys.stdin); print(json.dumps(d['dist-tags'], indent=1))"
# 实测输出：
#   alpha: 2026.5.19-alpha.1
#   latest: 2026.7.1-2
#   extended-stable: 2026.6.34   ← 生产选它（6.x 线内最稳）
#   beta: 2026.8.1-beta.1

# 生产建议：装 extended-stable（有 6.x 维护版）
npm install -g openclaw@extended-stable
openclaw --version   # 期望 2026.6.34 (hash)

# 插件同步升级（会自动选与运行时兼容的最新版本，无需指定版本）
openclaw plugins update --all
# 日志会显示：Resolved @openclaw/feishu to 2026.7.1, but incompatible → using 2026.6.34
```

**版本路线决策**：
- `extended-stable`（2026.6.34）：**生产首选** —— 6.x 线、含 7 月全部 feishu 流式修复（防重复内容/flush 计时器/卡片 API 30s 超时/blockStreaming 独立消息）。
- `latest`（2026.7.1-2）：有新功能（feishu live preview 2026-08-10 合入），但 7.1 流式慢渲染 issue 未决。
- `beta`（2026.8.1-beta.1）：追新冒险，生产勿用。

**升级后必做**（见 §8 完整流程）。

---

## 4. 飞书开放平台侧准备

每个 bot = 一个**独立自建应用**（推荐，错误/权限互不牵连；也可多账号共享一应用）。

| 项 | 值 | 位置（开放平台） |
|---|---|---|
| `appId` | `cli_xxxxxxxx` | 凭证与基础信息 |
| `appSecret` | 保密 | 同上 |
| 长连接 | 开启 | 事件与回调 → 订阅方式 → **使用长连接接收事件/回调** |
| 事件订阅 | `im.message.receive_v1` | 事件与回调 → 事件订阅 |

**权限（重要）**：
- 收发消息读写权限（如 `im:message`、`im:message:send_as_bot`）。
- **bot 加入群需要 `im:chat.members:write_only`**（错误码 99991672 时在权限页搜索该 scope 申请）。
- 想用 API 读回群消息做真值验证：补 `im:message.group_msg` 读权限（**默认 4 应用都没有，读群消息 API 返回 code 230027**）。

下线后把 `appId/appSecret` 填进 openclaw.json（§5）。

---

## 5. 配置骨架（openclaw.json）

路径：`$HOME/.openclaw/openclaw.json`。以下是 4 bot 实战配置的骨架（节选）。

### 5.1 全局：模型 provider（iagent 为例）

```jsonc
{
  "models": {
    "providers": {
      "iagent": {
        "baseUrl": "https://iagent.iauto.com/v1",
        "apiKey": "igw_…",              // 注意权限收敛（§11）
        "api": "openai-completions",
        "timeoutSeconds": 3600,          // 单次模型请求超时（见 §9.4）
        "models": [
          {
            "id": "iagent/standard",
            "name": "IAgent Standard",
            "contextWindow": 131072,     // ⚠️ 必须<=实际上限，见 §9.5
            "maxTokens": 16384
          }
        ]
      }
    }
  }
}
```

### 5.2 Agent 默认与超时

```jsonc
{
  "agents": {
    "defaults": {
      "model": {                            // ⭐ 形态是对象：primary + fallbacks
        "primary": "iagent/standard",
        "fallbacks": [
          "iagent/qwen3.5-b300/Qwen3.5-122B",
          "iagent/deepseek.v.4-b300/DeepSeek-V4-Flash"
        ]
      },
      "timeoutSeconds": 10800,        // agent 运行总超时（天花板），见 §9.4
      "toolProgressDetail": "raw",    // 工具进度详情模式
      "verboseDefault": "on"          // ⚠️⭐ 工具行可见的关键开关！见 §9.2
    }
  }
}
```

### 5.3 飞书 channel（顶层 + 账户级双保险）

```jsonc
{
  "channels": {
    "feishu": {
      "enabled": true,
      "appId": "cli_aab920536efa1cc8",   // default 账号的 app
      "appSecret": "…",
      // 连接模式：长连接(websocket) 在飞书开发者后台「订阅方式」设置，
      // openclaw.json 无 connectionMode 键（实测 6.34 config 无此字段）
      "blockStreaming": true,            // 顶层=default 账号
      "blockStreamingCoalesce": { "enabled": true, "minDelayMs": 400, "maxDelayMs": 1200 },  // 完成块合并冲卡
      "chunkMode": "newline",            // 块切分方式
      "streaming": true,                 // ⭐ 流式开关（与 blockStreaming 配合）
      "renderMode": "card",
      "typingIndicator": true,
      "replyInThread": "enabled",
      "groupSessionScope": "group_topic", // 一话题一个 session（最强隔离）
      "resolveSenderNames": true,
      "groupSenderAllowFrom": ["ou_…"],  // 群内允许的发送人（默认拥有者）
      "accounts": {
        "default": {
          "dmPolicy": "allowlist",
          "allowFrom": ["ou_own…"],      // ⚠️ 账户策略读取在 accounts.*
          "groupPolicy": "allowlist",
          "groupAllowFrom": ["oc_main群…"]
        },
        "newbot": {
          "appId": "cli_newbot…",
          "appSecret": "…",
          "name": "小七（运维工）",
          "blockStreaming": true,        // ⭐ 账户级配置！见 §9.3
          "renderMode": "card",
          "dmPolicy": "allowlist", …     // （策略和 default 一致铺开）
        },
        "bot2": { … }, "bot3": { … }
      }
    }
  }
}
```

### 5.4 Tools（网络抓取余量）

```jsonc
{
  "tools": {
    "web": { "fetch": { "timeoutSeconds": 60 }, "search": { "timeoutSeconds": 60 } },
    "byProvider": { "iagent/qwen3.5-b300/Qwen3.5-122B": { "deny": ["group:web", "browser"] } }
  }
}
```

### 5.5 配置验证命令

```bash
openclaw config validate    # 全量 schema 校验
openclaw doctor             # 健康检查（Plugins Errors 应为 0）
```

---

## 6. 多 Bot / 多 Agent / 多工作区

### 6.1 路由设计（推荐）

| account | agent | 群名 | 工作区 | 人格 IDENTITY |
|---|---|---|---|---|
| default | main（阿岩⛰️ 总指挥） | OpenClaw·阿岩 | `workspace-main` | 阿岩 |
| newbot | newbot（小七🔧 运维） | OpenClaw·小七 | `workspace-ops` | 小七 |
| bot2 | bot2（老六🐍 代码） | OpenClaw·老六 | `workspace-lab` | 老六 |
| bot3 | bot3（阿叁📚 资料） | OpenClaw·阿叁 | `workspace-lib` | 阿叁 |

- 每个 agent 独立 `agents/<id>/`（AGENTS/SOUL/IDENTITY/USER/记忆/会话）。
- **DM 映射**：accountId（app）自动路由到同名 agent；可显式 `bindings` 覆盖。

### 6.2 工作区改名机制（重要坑）

- `main` 不写 `workspace` 行时，解析到默认 `$HOME/.openclaw/workspace`；
- **显式加 `workspace` 行后走配置路径**。改工作区 = `mv 目录` + config 写 4 行 + doctor + 重启。
- 改后 `openclaw agents` 应显示新路径（验证）。

### 6.3 隔离保障

- 群内 `groupSessionScope: group_topic` + `replyInThread: enabled` → 一话题一个 session，最强隔离。
- `groupPolicy/groupAllowFrom` 白名单防误入群。

---

## 7. systemd 常驻服务

```bash
# 生成服务（安装时自动）或手工字段
systemctl --user daemon-reload
systemctl --user enable --now openclaw-gateway.service
systemctl --user status openclaw-gateway.service
```

**ExecStart 形态（保持原样）**：
```
ExecStart=<nvm>/bin/node <nvm>/lib/node_modules/openclaw/dist/index.js gateway --port 18789
```

**必须项的 unit 内容**（不要整组 hardening！）：
```
[Service]
Environment=HOME=/workspace/data/xieming
UMask=0077        # ⭐ 权限持久化关键（见 §11 坑2）
TimeoutStopSec=…  # 给长文档发送留下 SIGTERM 宽限（官方实践，见升级文档）
```

⚠️ 排错期临时 `--log-level debug` 加在 `gateway` 前（`dist/index.js --log-level debug gateway --port 18789`），**结束必须改回**（否则日志爆炸）。备份 unit 后改，验完还原。

**日志**：
- systemd：`journalctl --user -u openclaw-gateway.service -f`
- 文件写：`/tmp/openclaw/openclaw-YYYY-MM-DD.log`（debug 级别最细）

---

## 8. 升级路径（extended-stable 纪律）

**完整流程（已实战验证 6.8 → 6.34）**：

```bash
# 1. 备份（三件套：config + unit + 插件目录）
BK=~/.openclaw/backups/upgrade-$(date +%Y%m%d-%H%M%S)
mkdir -p $BK
cp ~/.openclaw/openclaw.json $BK/
cp ~/.config/systemd/user/openclaw-gateway.service $BK/
tar czf $BK/feishu-plugin.tgz -C ~/.openclaw/npm .
sha256sum $BK/openclaw.json   # 记下

# 2. 升级核心（不要直接 npm update，用 dist-tag）
npm install -g openclaw@extended-stable

# 3. 插件同步升级
openclaw plugins update --all

# 4. 迁移配置 + 修 schema
openclaw doctor --fix          # 自动：账户策略下移 accounts.default、清 stale 插件条目

# 5. 同步 unit 版本标记（可选但避免 status 提示；键名是 OPENCLAW_SERVICE_VERSION）
sed -i 's/OpenClaw Gateway (v2026.6.8)/OpenClaw Gateway (v2026.6.34)/; s/OPENCLAW_SERVICE_VERSION=2026.6.8/OPENCLAW_SERVICE_VERSION=2026.6.34/' \
    ~/.config/systemd/user/openclaw-gateway.service
systemctl --user daemon-reload

# 6. 重启 & 验证
systemctl --user restart openclaw-gateway.service
openclaw gateway status --deep   # CLI/Gateway 双版本一致 + Runtime running + probe ok
openclaw --version
```

**升级后检查清单**：4 账号 ws `started` 日志、`openclaw agents list`、`config validate`、一次真实 DM 工具行验证（§13）。

---

## 9. 机制真相（决定一切行为的底层规则）

> 这些是**源码级验证过的硬事实**（dist 直读）。不搞清它们，配置再对也白搭。

### 9.1 `room_event` 硬规则（群聊默认不可见）
- 群内消息被分类为 `InboundEventKind = "room_event"`，`source-reply-delivery-mode.ts` 无条件返回 `message_tool_only`：
  - **群内最终回复默认私有**，必须由模型显式调用 `message(action=send)` 工具才可见；
  - 工具进度行在群内同样被抑制（`message_tool_only && !shouldEmitToolResultProgress() → return`）。
- 这是**刻意设计**（官方确认），不是 bug；改 `messages.groupChat.visibleReplies` 无法绕过入站群消息。
- DM（p2p）**不受此限制**——最终文本自动可见。
- 实践中：模型可以学会用 `message` 工具（07:23 实测群内 198 字 committed），但工具行在群内仍不可见。

### 9.2 ⭐ 工具行的开关链（DM 可用！）
升级到 6.34 后工具行链路**已接通**（6.8 事件根本没接，不要倒退）：

```
agents.defaults.verboseDefault = "on"   // ⭐ KEEP：off=全透（默认）
  → forwardFollowupProgressEvent → onToolStart 事件
  → feishu 插件 formatChannelProgressDraftLineForEntry
  → updateStreamingStatusLine → 卡片 status 行 "🛠️ exec uname -a"
```

- 合法值：`off | on | full`；配合 `toolProgressDetail`（`raw`/`explain`）。
- 位置在正文**末尾**是设计（拼接顺序 thinking + '---' + 正文 + statusLine）。
- 6.34 变更：事件改名 `onToolProgress → onToolStart`（按旧名 grep 得 0 是假象）。
- **群内同样**：被 §9.1 room_event 硬规则（`message_tool_only`）压着；上游暂无 per-channel toolProgress 开关（相关 issue 仍 open），群内工具行暂不可行。

### 9.3 账户级 vs 顶层配置作用域（最坑）
- feishu 插件运行时读取 `account.config.*`（**账户级**）。顶层只对 `default` 账号生效。
- **新账号必须补**：`blockStreaming: true` + `renderMode: "card"`（否则块流式/卡片渲染不生效）。
- 6.34 升级后 `dmPolicy/allowFrom` 由顶层自动迁移到 `accounts.default`。

### 9.4 超时体系（三层，必须联动）
```text
① agents.defaults.timeoutSeconds  整个 run 的上限（默认 48h）
② models.providers.*.timeoutSeconds  单请求上限
③ CLI --timeout（0=永不）
```
- **①是天花板**：②再大也不能超过①；两者都设才算数。
- 实战推荐：① 10800（3h）、② 3600（1h）。文档原话："provider timeouts cannot extend the whole run … raise that ceiling too"。

### 9.5 模型限制错配（卡死/⚠️failed 必查）
- 现象：长会话 → `contextWindow 524288` 与供应商实际 131072 不符 → compaction 400 → 卡到 timeout → 群里投「工具链失败摘要」。
- 解法：`models[].contextWindow` 必须 ≤ 真实上限（131072）；`maxTokens ≤ 16384`；+ ① 超时拉高。
- 排查：`openclaw agent --json -m "…"` 会给出 `executionTrace` 和 HTTP 错误提示。

### 9.6 块流式 / 卡片行为
- `blockStreaming: true` → 完成块尽早冲卡；`renderMode: "card"` / `"auto"` 控制最终卡。
- `renderMode: "raw"` 时**不产生**流式卡（禁用进度渲染）。
- DM 长任务：正文逐段滚出 + 最终总结进卡；文本回复 `queuedFinal=true, replies=1`；纯工具任务 `queuedFinal=false, replies=0` 也正常（最终文本走卡片 finalize，无独立消息）。

---

## 10. 飞书 API 自助操作（建群/改名/发消息）

用 Python + tenant_access_token（`app_id/app_secret`），**勿外泄 token**。

```python
# 鉴权
POST /open-apis/auth/v3/tenant_access_token/internal
{"app_id","app_secret"} → tenant_access_token
```

### 建群
```python
POST /open-apis/im/v1/chats?user_id_type=open_id
{"name":"OpenClaw·阿岩","user_id_list":[owner_ou],"owner_id":owner_ou}
# → 成功返回 chat_id: "oc_…"（chat_mode: group）
```

### 改群名（⚠️ 必须 PUT，PATCH 404）
```python
# ❌ PATCH  → 404（飞书端点不支持 PATCH）
PUT /open-apis/im/v1/chats/{chat_id}  body: {"name":"OpenClaw·新名"}
# ✅ → {"code":0,"msg":"success"}
```

### 发消息验证
```python
POST /open-apis/im/v1/messages?receive_id_type=chat_id
{"receive_id":chat_id,"msg_type":"text","content":{"text":"hi"}}
```

---

## 11. 安全基线（共享主机）

| 对象 | 权限 | 做法 |
|---|---|---|
| `agents/*/sessions/` | **700** | unit `UMask=0077`（持久化，别只 chmod）|
| `.jsonl.deleted.*` | 600 | 批量收敛 |
| `.bashrc` / opencode.jsonc（含 key） | 600 | |
| `.claude/transcripts/` | 700 | |
| `iagent` API key | — | 收敛权限、**移出 git 索引**（`git rm --cached`，但历史仍含——加 remote 前必须 filter-repo） |
| systemd hardening | **只加 UMask** | 进程限制类选项（NoNewPrivileges/Protect*）会炸 gateway（实测 exit 218），禁止整组注入 |

---

## 12. 排错速查表（现象 → 根因 → 解法）

| 现象 | 根因 | 解法 |
|---|---|---|
| 飞书无任何回复 | 事件订阅非 websocket / 白名单没放开 | 检查开发者后台「长连接」，scope 补 `im:message`；`allowFrom` |
| 卡片出现但空、无内容 | `renderMode:raw` 或流式被禁用 | 看 §9.3：账户级 `renderMode:card` + `blockStreaming:true` |
| 没有 🛠️ 工具行 | `verboseDefault` 没设 or 6.8 老版本 | 设 `agents.defaults.verboseDefault:"on"` + 6.34；群内=规则限制（§9.1） |
| 工具行最后才出现 | 正常设计 | `statusLine` 追加在正文末尾（§9.2），非 bug |
| 长任务卡死 + ⚠️ failed 摘要 | contextWindow 超上限 / 超时天花板太低 | §9.5 模型限制 + §9.4 超时联动 |
| `config-health.json` conflict 警告 | SQLite 迁移遗留 | 无害，等上游或手动归档 |
| doctor 报 TaskFlow 孤儿 | 历史任务引用缺失 | `openclaw tasks flow cancel <id>` |
| 权限改了又回弹 775 | 忘 UMask | unit 加 `UMask=0077` 后重启验证 |

---

## 13. 验证清单（改造后必跑）

```bash
# 全链路冒烟（无需飞书；⚠️ CLI 走 embedded runner，不经 feishu monitor，
# 只验证 模型/provider/agent 路由；卡片渲染必须靠下方飞书 DM 实测）
openclaw gateway status --deep        # 双版本 + Runtime + probe ok
openclaw agent --agent main --channel feishu -m "回复'正常'" --json
# 期望: "text":"正常" stopReason="stop"，executionTrace.winnerProvider=iagent

# 飞书真实验证（必须有飞书账号）
#   1) 任一 bot DM 发：「请用 shell 执行 uname -a，把结果发给我」
#   2) 期望：卡片上出现 正文 + 🛠️ uname -a（在末尾）
#   3) 查日志：Started streaming → tool start → Closed streaming

# 权限复查（UMask 生效）
stat -c '%a' ~/.openclaw/agents/*/sessions   # 应为 700

# 安全
openclaw doctor   # Plugins Errors:0
```

---

> 附录 A：本机现状快照 → 见 `runbook-2026-08-12.md` §1 环境快照。
> 附录 B：命名体系设计与落地 → `design-proposal.md`。
> 最后维护：2026-08-12（实测回归 6.34 + verboseDefault 工具行 + 超时体系）。