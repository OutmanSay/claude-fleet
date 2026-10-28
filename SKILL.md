---
name: fleet
description: '舰队模式：Rex 终端里把窗格当 worker 用。旗舰=主会话（左格），助理=split 出来的独立 claude 进程（右格）。
  触发：/fleet、开舰队、起个助理、派个工人、右侧开一格跑 claude、让我能看见它干活、可视委派。
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

在 Rex 终端里把**窗格当 worker 用**的模式：主会话（旗舰）在左，split 出来的独立 claude 进程（助理）在右。人能看到助理全程、随时点进去接管；旗舰用 `rex send/capture` 派活、用 `SendMessage` 收回执。

## 跟官方 teammate mode 的区别

| | 官方 teammate | Fleet |
|---|---|---|
| 进程 | 同 session 内嵌 | 独立 claude 进程 |
| 可视 | 主会话看输出 | 独立窗格全程直播 |
| 接管 | 只能等 | 人随时点进去敲 |
| 通讯 | session 内 | 跨 session SendMessage |
| 生命周期 | 任务完就散 | 长驻，可反复派活 |

## 编制约定

- **旗舰** = 主会话自己（`<your-flagship-session-id>` 之类）
- **助理** = 右侧 split 出来的 claude 进程，编号 `助理-N`（1/2/3 顺延），职能写在括注里：**助理-1-哨兵 / 助理-2-苦力 / 助理-3-侦察**
- 每个助理 = 一个 `block:` + 一个 `<wingman-session-id>` session ID，**两张身份证都要记**
- ⚠️ 编号是固定身份证，职能是当前分工——改职能不能改号（哨兵今天跑监控、明天跑资料也是助理-1）

## ⭐ 旗舰不动手原则（2026-10-28 Danco 定）

**旗舰只做调度+决策，所有动手活派助理。** Danco 跟主会话聊天时，旗舰自己跑长任务会卡住对话——他得等你干完才能接着说。所以：

- ✅ 旗舰干的：接需求、判该派谁、写 dispatch prompt、收回执、汇总报给 Danco
- ⛔ 旗舰不干的：跑脚本、查库、写文件、批量操作、调试——一律 `rex send` 派给助理
- 唯一例外：**一句话能答的**（不用工具、不用查文件）直接答；以及**没人可派时的急救**（比如助理全挂、Danco 说"你现在就改"）
- 判据：这个活要不要超过 30 秒 / 要不要跑 3+ 个工具调用 → 是就派出去
- 助理不够就 `/fleet up` 开新的，别自己顶

## 命令

### `/fleet` 或 `/fleet up [N]`

开 N 个助理（默认 1），纵向 split 在右侧（已有助理就 stacked 在下）：

```bash
# 第一个助理（锚主块，split=right）
rex split -s "$REX_SESSION" -b "$REX_BLOCK" --split=right --ratio 40 \
  --label "助理-1" --focus=false --json -- claude

# 第二个起（锚上一个助理的 block，split=below）
rex split -s "$REX_SESSION" -b <上一个助理的block> --split=below --ratio 50 \
  --label "助理-2" --focus=false --json -- claude
```

返回 `.block_ids[0]` 是助理的 block ID，存起来。

**起来后必做两件事**：
1. **改名**：`rex block rename -s "$REX_SESSION" <block_id> "助理-N"` 或职能名
2. **对号 session**：`ListAgents` 看新出现的 `danco-XX`（启动时间就是刚才），记下「助理-N ↔ block ↔ danco-XX」映射

### `/fleet send <助理> <任务>`

派活。两步：

```bash
rex send -s "$REX_SESSION" -b <block_id> '<任务文本>' && \
rex send-key -s "$REX_SESSION" -b <block_id> enter && \
sleep 1 && rex capture -s "$REX_SESSION" -b <block_id> --trim | tail -3  # 核验：输入框应空、有 Working/思考标记
```

⚠️ **核验必做（2026-10-28 三次漏按教训）**：send-key enter 后必须 `rex capture` 确认——屏幕应显示「Working/thinking」或输入框已空。若文字仍停在输入框（↑标记），说明没提交，补发 enter。

任务文本里**必须带这三条指令**（否则它不回报、乱动 rex、舰内协同断）：

```
干完用 SendMessage 告诉 <旗舰-session-id> 结果（一句话+关键产物路径）。
别动 rex session/window/block/rename。
你是助理-N（职能名），每干完一件可复用的事就往 /tmp/fleet-shared.log 追加一行 `HH:MM <助理名>: <干了什么> | 产物:<路径>`。
```

⚠️ **任务命中已有 skill 时（2026-10-28 补）**：派活 prompt 里加一句「**先读 `~/.openclaw/skills/<skill-name>/SKILL.md` 按契约走**」。skill 是唯一真源（msg-monitor 有控制台命令+展示约定、yishitong 有标准审批流），不要让助理按口述流程瞎跑——它不会自动加载 skill，只会按你写的字面做。

### `/fleet ls`

列编队状态：`rex block ls -s "$REX_SESSION"` + `ListAgents` 对号。

### `/fleet read <助理>` 或 `/fleet capture <助理>`

读助理屏幕：`rex capture -s "$REX_SESSION" -b <block_id> --trim | tail -50`

### `/fleet down <助理>` 或 `/fleet down all`

收编：`rex block close -s "$REX_SESSION" <block_id>`（会杀掉那个 claude 进程）

### `/fleet radar`

挂雷达：主会话定期 `rex capture` 各助理 + 检查 `/tmp/w*-*.log` 有新行，**只有异常/完成/新产物才推给用户**，把人从盯梢里解放出来（抄 muxboard 的 "what needs me?" 思路）。实现：`run_in_background` 起一个 bash while 循环 + `ScheduleWakeup`，或在主会话里每次新消息顺手 capture 一遍。

### `/fleet share`（共享黑板，2026-10-28 沉淀）

**问题**：助理各自独立跑，A 干完了 B 不知道，B 重复报/重复干。

**方案**：共享黑板 `/tmp/fleet-shared.log`——单向、append-only、零依赖。

- **写入约定**：助理每干完一件可复用的事就追加一行：`HH:MM <助理名>: <干了什么> | 产物:<路径>`
- **消费方**：哨兵/监控 loop 每轮开头 `tail -30 /tmp/fleet-shared.log`，把里面已干的事从「新待办」里剔除，并 stdout 打一行 `[fleet] <助理名>已处理:<条目>` 让旗舰看见协同
- **装配**：给助理的 prompt 模板里写清楚「你叫什么名/干完追加到哪一行」；监控 loop 的 bash 加 `tail` 前置步骤

比起互发 SendMessage 的双向通讯，黑板胜在：不占对方上下文、可重放、人去翻也看得懂。

⚠️ **粒度红线（2026-10-28 哨兵实战）**：监控助理标 `[fleet] 已处理` 时**只标不藏**——人名/关键词粒度太粗（"吕碧雯批了A"不能推"吕碧雯的B也批了"），隐藏原候选行会漏待办。规则：`[fleet]` 行+原候选并列展示，由旗舰裁决。

## 通讯协议（已验证 2026-10-28）

- **我 → 助理**：`rex send` 打字（推荐，你能看到）或 `SendMessage to <助理-session-id>`
- **助理 → 我**：`SendMessage to <旗舰-session-id>`（在 prompt 里点我的名字）
- **助理 → 助理**：两条路，按目的选——
  - **发通知**：`SendMessage to <对方-session-id>`（先说对方 session 名）
  - **共享状态（推荐）**：共享黑板 `/tmp/fleet-shared.log`，追加格式 `HH:MM <助理名>: <干了什么> | 产物:<路径>`

`ListAgents` 每行格式：`danco-XX [ref] · interactive · busy|idle · started N ago`。**新起的助理 = started 最短的**。

## 踩过的坑

- ⚠️ **助理会改 rex session 名**：claude 启动会触发 SessionStart hook 叫它 rename，而它和我同 session——会改掉我这个 session 的 label。**派活 prompt 里必带「别动 rex session」**。
- ⚠️ **助理独立跑会断协同**：A 干完 B 不知道，监控类助理会重复报已处理项 → 用 `/fleet share` 共享黑板（`/tmp/fleet-shared.log`）当唯一协同总线，派活 prompt 里教它追加格式。
- ⚠️ **Codex 助理没有 SendMessage**（2026-10-28 实测）：`codex` CLI 是异构引擎，没有 CC 的 SendMessage/ListAgents，只能**单向 rex send 派活 + 在窗格里输出**——旗舰须 `rex capture` 轮询读屏，不能等回报。派活 prompt 里别写「SendMessage 给我」，写了它也做不到。
- ⚠️ **`rex send` 不会自动回车**（2026-10-28 实测）：send 完文字停在输入框（↑标记），必须紧跟 `rex send-key enter`。fleet skill 的 `/fleet send` 流程已含两步，别拆。
- ⚠️ **Codex 卡住会问「是否继续」**：权限弹窗/敏感操作它会停下来等指示，不会自己动——要么 prompt 里预先授权（"定位弹窗你自己点"），要么捕获到它在等指示时补发指令。
- ⚠️ **异构助理命名**：助理-N 编号不变，但职能标引擎——`助理-3-codex-跑腿` 比 `助理-3-跑腿` 清楚（一看就知道它通讯能力不同）。
- ⚠️ **session 名 ≠ session ID**：助理自己 `rex session rename` 后，label 变了但 `danco-XX` ID 不变，SendMessage 仍可达。记映射要记 **ID 不是 label**。
- ⚠️ **bypass 权限模式才有用**：起助理用 `claude`（继承 ~/.claude/settings.json 的 bypassPermissions），普通模式会每步都弹权限框废掉。
- ⚠️ **新 pane 会抢焦点**：`--focus=false` 必带，否则用户的输入焦点被切走。
- ⚠️ **跨 session 消息有延迟**：助理 idle 时收件快，busy 时会延迟到下个 tool round 才看。急事还是 `rex capture` 直接看屏。
- ⚠️ **ListAgents 看不到自己**：`danco-ef`（本 session）不在列表里，发消息给它=发给自己。

## 收队仪式感（2026-10-28 沉淀）

任务收尾时让助理自报一句「我是 danco-XX，block 是 XXX，任务已完成」，主会话核完 SendMessage 一句「收队」——它能听到，然后 `rex block close` 收编。

### ⚠️ 收队别抢跑（2026-10-28 Danco 定）

**助理干完 ≠ 可以关。** Danco 要自己进窗格翻原始对话看细节（SendMessage 回执是压缩过的，再经旗舰转述就二次压缩）。

- ⛔ 别一收到 SendMessage 就 `rex block close`——他还没看。
- ✅ 助理完工 → 旗舰**照样汇报**（实质内容、产物路径、关键发现，别"详见报告"打发）→ **窗格留着** → 等 Danco 说"收了/关了"再 close。
- 汇报和留窗格**两件事都做**，不是二选一。
