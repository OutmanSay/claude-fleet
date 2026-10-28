---
name: fleet
description: '舰队模式：Rex 终端里把窗格当 worker 用。旗舰=主会话（左格），僚舰=split 出来的独立 claude 进程（右格）。
  触发：/fleet、开舰队、起个僚舰、派个工人、右侧开一格跑 claude、让我能看见它干活、可视委派。
  核心卖点：worker 全程可视 + 人可随时接管 + 跨 session SendMessage 回执。
  不触发：纯后台跑完就散（→ codex-delegate 看板）；本机外机器（Rex 不管远程）。'
metadata:
  openclaw:
    emoji: 🛳
allowed-tools:
- Bash
- SendMessage
- ListAgents
---

# 舰队模式（Fleet Mode）

在 Rex 终端里把**窗格当 worker 用**的模式：主会话（旗舰）在左，split 出来的独立 claude 进程（僚舰）在右。人能看到僚舰全程、随时点进去接管；旗舰用 `rex send/capture` 派活、用 `SendMessage` 收回执。

## 跟官方 teammate mode 的区别

| | 官方 teammate | Fleet |
|---|---|---|
| 进程 | 同 session 内嵌 | 独立 claude 进程 |
| 可视 | 主会话看输出 | 独立窗格全程直播 |
| 接管 | 只能等 | 人随时点进去敲 |
| 通讯 | session 内 | 跨 session SendMessage |
| 生命周期 | 任务完就散 | 长驻，可反复派活 |

## 编制约定

- **旗舰** = 主会话自己（`danco-ef` 之类）
- **僚舰** = 右侧 split 出来的 claude 进程，编号 1/2/3，label 写「僚舰-N」或职能名（哨兵/苦力/侦察）
- 每个僚舰 = 一个 `block:` + 一个 `danco-XX` session ID，**两张身份证都要记**

## 命令

### `/fleet` 或 `/fleet up [N]`

开 N 个僚舰（默认 1），纵向 split 在右侧（已有僚舰就 stacked 在下）：

```bash
# 第一个僚舰（锚主块，split=right）
rex split -s "$REX_SESSION" -b "$REX_BLOCK" --split=right --ratio 40 \
  --label "僚舰-1" --focus=false --json -- claude

# 第二个起（锚上一个僚舰的 block，split=below）
rex split -s "$REX_SESSION" -b <上一个僚舰的block> --split=below --ratio 50 \
  --label "僚舰-2" --focus=false --json -- claude
```

返回 `.block_ids[0]` 是僚舰的 block ID，存起来。

**起来后必做两件事**：
1. **改名**：`rex block rename -s "$REX_SESSION" <block_id> "僚舰-N"` 或职能名
2. **对号 session**：`ListAgents` 看新出现的 `danco-XX`（启动时间就是刚才），记下「僚舰-N ↔ block ↔ danco-XX」映射

### `/fleet send <僚舰> <任务>`

派活。两步：

```bash
rex send -s "$REX_SESSION" -b <block_id> '<任务文本>' && \
rex send-key -s "$REX_SESSION" -b <block_id> enter
```

任务文本里**必须带这两条指令**（否则它不回报、可能乱动 rex）：

```
干完用 SendMessage 告诉 danco-ef 结果（一句话+关键产物路径）。
别动 rex session/window/block/rename。
```

### `/fleet ls`

列编队状态：`rex block ls -s "$REX_SESSION"` + `ListAgents` 对号。

### `/fleet read <僚舰>` 或 `/fleet capture <僚舰>`

读僚舰屏幕：`rex capture -s "$REX_SESSION" -b <block_id> --trim | tail -50`

### `/fleet down <僚舰>` 或 `/fleet down all`

收编：`rex block close -s "$REX_SESSION" <block_id>`（会杀掉那个 claude 进程）

### `/fleet radar`

挂雷达：主会话定期 `rex capture` 各僚舰 + 检查 `/tmp/w*-*.log` 有新行，**只有异常/完成/新产物才推给用户**，把人从盯梢里解放出来（抄 muxboard 的 "what needs me?" 思路）。实现：`run_in_background` 起一个 bash while 循环 + `ScheduleWakeup`，或在主会话里每次新消息顺手 capture 一遍。

## 通讯协议（已验证 2026-10-28）

- **我 → 僚舰**：`rex send` 打字（推荐，你能看到）或 `SendMessage to danco-XX`
- **僚舰 → 我**：`SendMessage to danco-ef`（在 prompt 里点我的名字）
- **僚舰 → 僚舰**：也走 `SendMessage`，先告诉它们对方 session 名

`ListAgents` 每行格式：`danco-XX [ref] · interactive · busy|idle · started N ago`。**新起的僚舰 = started 最短的**。

## 踩过的坑

- ⚠️ **僚舰会改 rex session 名**：claude 启动会触发 SessionStart hook 叫它 rename，而它和我同 session——会改掉我这个 session 的 label。**派活 prompt 里必带「别动 rex session」**。
- ⚠️ **session 名 ≠ session ID**：僚舰自己 `rex session rename` 后，label 变了但 `danco-XX` ID 不变，SendMessage 仍可达。记映射要记 **ID 不是 label**。
- ⚠️ **bypass 权限模式才有用**：起僚舰用 `claude`（继承 ~/.claude/settings.json 的 bypassPermissions），普通模式会每步都弹权限框废掉。
- ⚠️ **新 pane 会抢焦点**：`--focus=false` 必带，否则用户的输入焦点被切走。
- ⚠️ **跨 session 消息有延迟**：僚舰 idle 时收件快，busy 时会延迟到下个 tool round 才看。急事还是 `rex capture` 直接看屏。
- ⚠️ **ListAgents 看不到自己**：`danco-ef`（本 session）不在列表里，发消息给它=发给自己。

## 收队仪式感（2026-10-28 沉淀）

任务收尾时让僚舰自报一句「我是 danco-XX，block 是 XXX，任务已完成」，主会话核完 SendMessage 一句「收队」——它能听到，然后 `rex block close` 收编。
