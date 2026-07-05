# 二改版说明（相对官方 tutusagi/ai-fishing-game）

在原版 `engine.py` 基础上加了四块，靠 `build_blind.py` 重新打包进 `fishing.py`，AI 玩家只调 `fishing.cmd()` 就行，不用额外学新接口。

本版重点：先保证 AI 能正确找到项目入口，再进入首次陪玩引导；避免“看见 README 但说缺 fishing.py”的假故障。

---

## 0. 文件定位规则（File Discovery）

新增 README 入口规则：AI 如果能看到项目文件夹或 `README.md`，但暂时看不到 `fishing.py`、`engine.py` 或其他运行文件，不能立刻判断“文件缺失”。

在要求玩家重新上传前，AI 必须先检查：

- 是否在正确的项目文件夹；
- 运行文件是否在子文件夹里；
- Drive / connector 返回结果是否不完整、分页或延迟；
- 仓库上传时是否多套了一层目录；
- 同步延迟或连接器可见性是否影响了文件列表。

如果检查后仍然看不到运行文件，AI 应该说明：

```text
I currently cannot locate the runtime files. This does not necessarily mean they are missing. They may be in a subfolder, hidden by the connector view, or not fully listed yet.
```

然后让玩家确认文件夹结构，而不是直接要求重新上传。

总顺序固定为：

```text
先完成文件定位 → 再做首次陪玩引导 → 再执行 handshake → 再开始游戏命令
```

---

## 1. 存档导出提醒（防止忘记同步到 Drive/本地文件夹）

- 存档里新增字段 `last_export_turn`，记录“上次确认已同步到外部存储时的回合数”。
- 每次 `cmd()` 调用（纯查看类指令除外）都会检查：当前回合 - `last_export_turn` ≥ 15，就在输出末尾强制打印警告。
- 同步完存档后，调用 `cmd("__exported__")` 更新标记、清掉警告。
- 若环境不能覆盖文件，或要跨窗口 / 跨机器带走进度，可调用 `cmd("savecopy")`，生成 `fishing_save_YYYYMMDD_HHMMSS.json` 时间戳备份，并同时更新导出标记。
- `__exported__` 与 `savecopy` 都走 `_run_one()`，所以单独调用和批量调用（如 `cmd("savecopy; status")`）都有效。

---

## 2. 收藏缸（collection）

- 新指令：`collection default`（稀有及以上自动收藏）/ `collection off`（不用）/ `collection custom <rarity>`（自定义门槛，可选 `common/uncommon/rare/epic/legendary/mythic`）/ `collection status`（查看当前方案）。
- 配置存在同目录 `collection_config.json`，跟 `fishing_save.json` 分开存。
- 每次抛竿 / 潜水，只要渔获稀有度达到配置门槛，输出里会自动标注“该进收藏缸了”。

### 2.1 本次文案调整

旧表达容易把“收藏模式”和“入缸方式”拆成两项，让用户困惑：收藏了但不入缸？入缸了但不收藏？鱼到底去哪了？

新版规则：

```text
收藏 = 达到门槛的鱼进入鱼缸记录
```

收藏和入缸是一条流程：

```text
达到收藏门槛的鱼 → 提醒收藏 → 能写文件就保存进鱼缸文件夹 → 写入失败时生成可复制鱼缸记录
```

“入缸方式”只是收藏的执行方式，不是另一个独立设置。

首次开局只让玩家确认两件事：

1. 稀有鱼怎么收藏
2. 遇到画面感事件怎么配图

文件写入失败、路径处理、兜底记录生成，属于 AI 自己处理的执行细节，不在开局时让玩家选择。

### 2.2 术语替换

把以下旧词替换掉：

```text
收藏卡 → 鱼缸记录
生成一张收藏卡 → 生成一条可复制鱼缸记录
可复制收藏卡 → 可复制鱼缸记录
collection card → aquarium record
copyable collection card → copyable aquarium record
```

保留“该进收藏缸了”这个提醒，因为它表达的是“这条鱼值得进入鱼缸记录”。

---

## 3. 画面感提醒（配图用）

- 每次抛竿 / 潜水，输出末尾会汇总本轮出现的“值得配图 / 记录”的时刻：新种（🆕）、漂流瓶（📜）、藏宝图碎片 / 解锁（🧩🗺️）、潜水场景（🤿）、换季（🍃）、宝箱（📦），外加收藏缸提醒。
- 纯查看类指令（`status/help/shop/encyclopedia/inventory/look`）不会触发这些提醒，避免刷屏噪音。
- 暂不在游戏里写死具体搜图关键词；游戏只提醒“这里值得配图 / 记录”。具体关键词由当次 AI 根据鱼名、描述和实际搜索结果临场生成，避免未测试关键词导致跑偏。

---

## 4. 开局握手门禁（防止 AI 一上来就钓鱼）

- 新增 `handshake` 指令：`handshake status` / `handshake template` / `handshake defaults` / `handshake set ...` / `handshake reset`。
- 未完成握手前，游戏会拦截 `cast` / `dive` / `buy` / `goto <地点id>` / `sell` / `open` 等推进动作。
- 新版默认握手方案：稀有及以上自动收藏进鱼缸；能写文件就保存到「我的鱼缸」；写入失败时给可复制鱼缸记录；视觉模式为现实 / 气质网图参考。
- 存档和鱼缸记录必须写回原游戏项目目录：从 Google Drive 打开就写回同一个 Drive 文件夹，从本地打开就写回同一个本地文件夹；`/mnt/data` 等沙盒目录只能作为临时中转，不算长期保存。
- 鱼缸记录的正式格式是原项目目录「我的鱼缸」里的 `.md` 文件。Drive 稳定路线是 create Google Doc → 写入 Markdown 正文 → update_file/addParents 移到「我的鱼缸」文件夹；成功移动后可视为 Drive 正式落地，但要说明它是 Google Doc，不是原生 `.md`。如果无法指定/移动父目录，允许创建标题带 `【我的鱼缸】` 前缀的 Google Doc 作为 Drive 兜底保存，但必须说明可能尚未物理归位。
- 新增并发待办规则：同一轮如果同时出现收藏/存档/视觉任务，输出会给 `PENDING_TASKS`，AI 必须逐项清空，不能因为写鱼缸或存档就吃掉配图。
- 存档正式格式是 `fishing_save.json`。如果连接器不能创建或上传 `.json`，可临时保存为 `fishing_save.json.txt` 或带 `json` 代码块的 `fishing_save_backup.md`，但这只是可恢复备份，不是活跃存档。
- `cmd("start")` 和 `cmd("help")` 已加入握手说明；`new_game()` 后也会提示必须先握手。
- 视觉 / 收藏提醒追加机器可读标记：`VISUAL_REQUIRED: true` / `COLLECTION_REQUIRED: true`，方便 AI 不漏处理。

---

## 5. README 入口结构调整

README 的入口结构调整为两大块：

1. **File Discovery / 文件定位规则**：先确认项目文件完整、能找到运行文件。
2. **First Play Onboarding / 首次陪玩引导**：解释收藏 / 鱼缸 / 配图，然后等用户确认。

必须顺序：

```text
AI must complete file discovery before the opening onboarding.
AI must complete opening onboarding before any gameplay command.
```

这样避免下一轮出现“能看到 README，但误判缺少 fishing.py”的假故障。

---

## 怎么跑

```bash
python3 build_blind.py   # 改了 engine.py 之后，重新生成 fishing.py
python3 -c "import fishing; print(fishing.cmd('status'))"
```

---

## 后续要接着改

- 这几块都应该焊在 `cmd()` 本体和 `_run_one` 的指令分发里，不是外挂脚本——不管从哪个入口调用都躲不掉，这是刻意设计。
- 如果想加新提醒类型（比如“某地点已集齐”），改 `_VISUAL_MARKERS` 或 `_catch_rarity_from_line` 就行。
- 如果未来真的要做带图卡片，再恢复“收藏卡”这个名称；当前只是文字记录，所以统一叫“鱼缸记录”。