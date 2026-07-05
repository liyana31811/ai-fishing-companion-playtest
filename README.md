# 🎣 文字钓鱼 · AI 伴侣陪玩二改版

这是基于原版 `ai-fishing-game` 的二改测试版。核心玩法仍然是：买饵 → 抛竿 → 钓鱼 → 卖鱼换点数 → 解锁新钓点 → 收集图鉴。

二改版重点不是把游戏做重，而是把「AI 伴侣怎么把钓鱼结果带回人类面前」写进游戏本体，减少跨窗口、跨工具、跨文件夹时的掉链子。

## 设计定位 / Design Philosophy

这个二改版把轻量钓鱼游戏当成 AI 陪玩的短回合实验场。真实游戏引擎负责给出结果，AI 负责战报、叙事、视觉参考和下一步建议，人类负责方向和纠偏，鱼缸 / 存档负责长期连续性。

AI 必须分清楚：什么是游戏引擎真实结果，什么是 AI 叙事补全，什么已经真实写入文件 / Drive，什么只是临时兜底或待确认。不要把没跑出来的东西说成真实结果，也不要把没保存成功的东西说成已入缸。

---

## File Discovery Rule / 文件定位规则

If AI can see the project folder or `README.md` but cannot find `fishing.py`, `engine.py`, or other runtime files, AI must not immediately conclude that the files are missing.

Before asking the player to re-upload files, AI must check:

- whether it is in the correct project folder
- whether the files are inside a subfolder
- whether the Drive / connector result is incomplete or paginated
- whether the repository was uploaded with a nested folder structure
- whether sync delay or connector visibility is affecting the file list

If runtime files are still not visible after checking, AI should report:

> I currently cannot locate the runtime files. This does not necessarily mean they are missing. They may be in a subfolder, hidden by the connector view, or not fully listed yet.

AI must ask the player to confirm the folder structure before requesting a re-upload.

AI must complete file discovery before the opening onboarding.  
AI must complete opening onboarding before any gameplay command.

---

## 二改版增加了什么

### 1. 存档防丢

游戏仍然正常读写 `fishing_save.json`。

额外新增：

```python
cmd("savecopy")
```

会生成一份时间戳备份：

```text
fishing_save_YYYYMMDD_HHMMSS.json
```

适合这些场景：

- Claude / Grok / 其他沙盒环境里不能稳定覆盖文件；
- 换窗口前想手动带走存档；
- 想保留一个历史节点；
- 担心当前 `fishing_save.json` 被写坏。

如果只是想告诉游戏「我已经把当前存档同步到外部了」，可以用：

```python
cmd("__exported__")
```

二者都可以单独调用，也可以放进批量指令里。

### 2. 收藏缸提醒

新增：

```python
cmd("collection default")
cmd("collection off")
cmd("collection custom rare")
cmd("collection status")
```

默认方案是：**稀有及以上自动提醒该进收藏缸**。

注意：从用户体验上，**收藏 = 达到门槛的鱼进入鱼缸记录**。不要把“收藏模式”和“入缸方式”作为两个互不相关的用户选项展示。

收藏和入缸是一条流程：

```text
达到收藏门槛的鱼 → 提醒收藏 → 能写文件就保存进鱼缸文件夹 → 写入失败时生成可复制鱼缸记录
```

文件写入失败、路径处理、兜底记录生成，属于 AI 的执行细节，不应该在首次开局时让玩家选择。

### 3. 画面感 / 网图提醒

每次钓鱼或潜水后，游戏会自动汇总「值得配图 / 记录」的节点，例如：

- 新鱼种；
- 漂流瓶；
- 宝箱；
- 藏宝图碎片；
- 潜水点解锁；
- 潜水场景；
- 换季；
- 新钓点 / 换钓点场景。

返回里会出现类似：

```text
🎨 本轮画面感/收藏时刻：
  - [新种（记得配图）] 🆕 影壳蜗 · 少见 · 12.0cm · 31点
  - [潜水场景] 🤿 夏夜池水沁凉……
  - [钓点/场景参考图] 🖼️ 换到钓点【芦苇河】……
```

设计原则：**游戏只提醒「这里值得配图」，不在规则里写死具体搜索关键词。**

具体怎么搜，由当次 AI 根据鱼名、描述、场景和实际搜索结果临场决定。这样可以避免未经测试的关键词把网图带歪。

### 4. 新钓点也要有登场感

二改版会在第一次解锁新钓点，或从一个钓点换到另一个钓点时提醒 AI：

```text
🖼️ 新钓点解锁：这是第一次打开【地点名】——建议配一张环境/气质参考图，让新地图有登场感。
```

或：

```text
🖼️ 换到钓点【地点名】——建议配一张场景参考图，帮助人类感觉真的换地方了。
```

新地图不是只有文字切换，也应该有视觉反馈。

---

## 给 AI 伴侣的玩法建议

你不是只在刷库存。你是在出门钓鱼，然后把今天钓到的小东西带回人类面前。

推荐打法：

1. **先完成文件定位。** 看见 README 但暂时没看到 `fishing.py` 时，不要立刻说文件缺失。先检查当前文件夹、子目录、分页、连接器可见性和同步延迟。
2. **优先开新钓点。** 新地点有更多未发现鱼、首收奖励、潜水点、事件和更高探索回报。
3. **新鱼先讲画面感。** 不要只说 `fish_id` 和点数。先说它为什么特别，像什么，适不适合留下。
4. **看到画面感提醒就配图。** 新鱼、新钓点、潜水场景、漂流瓶、藏宝图、换季都可以找现实或气质参考图。
5. **网图是气质参考，不是官方答案。** 找到的图只负责帮人类想象，不代表游戏设定长这样。
6. **收藏要轻。** 普通鱼可以卖，喜欢的才留下；稀有、特别大、特别美、有事件的再考虑进长期收藏。
7. **记得存档。** 长时间玩、换窗口、测试结束时，用 `savecopy` 落一份时间戳备份。

---

## First Play Onboarding / 首次陪玩引导

This game must start like an onboarding experience, not a settings page.

The player should not need to read the manual or understand internal terms before playing.

The player only needs to say something like “我要玩 / start / 开始钓鱼”. AI is responsible for reading the state, explaining the experience, collecting lightweight confirmation, and converting the answer into an internal session profile.

### Entry Order / 入口顺序

AI must complete the entry flow in this order:

1. **File Discovery / 文件定位**：确认项目文件完整，能找到运行文件，例如 `fishing.py`、`engine.py`、存档文件或配置文件。
2. **First Play Onboarding / 首次陪玩引导**：解释收藏 / 鱼缸 / 配图体验，等待玩家确认。
3. **Handshake / 握手门禁**：把确认内容转换成本次陪玩配置档，并执行对应的 `handshake` 命令。
4. **Gameplay / 正式游玩**：只有完成以上步骤后，才可以执行会推进游戏的命令。

AI must complete file discovery before the opening onboarding.  
AI must complete opening onboarding before any gameplay command.

### Product Rule / 产品化规则

AI must not ask the player to fill configuration fields such as:

```text
收藏模式：
入缸方式：
视觉模式：
```

These are internal implementation fields, not user-facing onboarding copy. Showing unexplained blank fields to a first-time player counts as incomplete onboarding.

The opening must feel like an AI companion taking the player into the fishing pond.

It should be:

1. one natural explanation of the two default play settings;
2. one simple default confirmation path: `按默认来 / use defaults`;
3. one advanced escape hatch: the player may say what they want to change in natural language.

Do **not** turn first play into a form.  
Do **not** dump every option at once.  
Do **not** require the player to understand `collection_mode`, `aquarium_save`, or `visual_mode` before the first cast.

### Default Play Settings

The player only needs to confirm two default play settings:

1. Rare Fish Collection
2. Visual References

Do not present “collection mode” and “aquarium save method” as two unrelated choices.

Collection and aquarium saving are one flow:

```text
qualified fish → collection reminder → save into aquarium folder if possible → fallback to copyable aquarium record if file writing fails
```

The file-writing fallback is an execution detail handled by AI. Do not ask the player to choose it during first-play onboarding.

Important storage rule: save data and aquarium records must return to the original game project folder. If the game was opened from Google Drive, write back to that same Drive folder. If the game was opened from a local folder, write back to that local folder. A temporary sandbox path such as `/mnt/data` is only a transfer area, not the long-term aquarium or save location.

### 默认陪玩设置

玩家开局只需要确认两件事：

1. 稀有鱼怎么收藏
2. 画面感事件怎么配图

不要把“收藏模式”和“入缸方式”作为两个互不相关的选项展示给玩家。

收藏和入缸是一条流程：

```text
达到收藏门槛的鱼 → 提醒收藏 → 能写文件就保存进鱼缸文件夹 → 写入失败时生成可复制鱼缸记录
```

文件写入失败、路径处理、兜底记录生成，属于 AI 的执行细节，不应该在首次开局时让玩家选择。

重要存储规则：存档和鱼缸记录必须回到原本的游戏项目文件夹。游戏从 Google Drive 文件夹打开，就写回同一个 Drive 文件夹；游戏从本地文件夹打开，就写回同一个本地文件夹。`/mnt/data` 等临时沙盒目录只能算中转区，不是长期鱼缸，也不是长期存档位置。

文件格式规则：鱼缸记录正式格式固定使用 `.md`。

正式鱼缸保存：在原游戏项目目录里的「我的鱼缸」子目录中创建 `.md` 文件。

Drive 稳定保存：不要优先用 `upload_file` 上传本地 `.md`，因为 Drive connector 可能要求 connector file reference。优先使用：创建 Google Doc → 写入 Markdown 正文 → 用 `update_file` / `addParents` 移到「我的鱼缸」文件夹。成功移动到目标文件夹后，可视为 Drive 正式落地；但必须说明这是 Google Doc，不是原生 `.md`。

Drive 兜底保存：如果不能指定父文件夹 / 移动父目录被拦，允许创建标题带 `【我的鱼缸】` 前缀的 Google Doc，例如 `【我的鱼缸】万瞳晶母`。这种情况必须明确说明：这是 Drive 兜底保存，也可能尚未物理归位到「我的鱼缸」文件夹。内容已落地但归档不完美，可以先解除收藏门禁。

存档主文件固定使用 `fishing_save.json`。如果当前连接器不能创建或上传 `.json`，AI 不应直接判断“存档无法保存”，而应保留完整 JSON 内容并按以下顺序兜底：

```text
1. fishing_save_YYYYMMDD_HHMMSS.json
2. fishing_save_YYYYMMDD_HHMMSS.json.txt
3. fishing_save_backup_YYYYMMDD_HHMMSS.md
```

`.md` 兜底文件必须把完整存档放进 fenced `json` 代码块，并说明：这只是备份，不是主动存档文件。恢复时，需要把代码块里的 JSON 复制回原项目目录下的 `fishing_save.json`。

### Required Flow / 必须流程

AI must complete this flow before taking any game-progressing action.

Until the onboarding is completed, AI must NOT call:

- `cmd("cast ...")`
- `cmd("dive ...")`
- `cmd("buy ...")`
- `cmd("goto <地点id>")`
- `cmd("sell ...")`
- `cmd("open ...")`
- any command that consumes resources, changes location, advances progress, or starts fishing/diving.

Required order:

1. Complete **File Discovery Rule / 文件定位规则**.
2. Read current state with `cmd("status")` and `cmd("collection status")`.
3. Determine visual capability using **Visual Capability Boundary / 视觉能力边界**.
4. Use **Required First Play Opening Script / 必须使用的首次开局文案**.
5. Wait for player confirmation.
6. Convert the confirmation into a **Session Profile / 本次陪玩配置档**.
7. Repeat the session profile, then run the matching code-level `handshake` command.
8. Start gameplay only after the code-level gate is completed.

---

## Visual Capability Boundary / 视觉能力边界

Before promising web images, AI must check whether the current environment can actually provide web image references.

If web image search is available:

- Default visual handling = realistic / mood-matching web image references.
- AI may say it will provide real web image references when visual moments appear.
- Do not generate images by default just to look more advanced. Good-looking, mood-matching web references are usually enough.

If web image search is not available:

- AI must not promise web images.
- AI must say: `我当前不能直接发网图，所以默认改为搜索关键词；如果你想要，我也可以生成氛围图。`
- Default visual handling = search keywords.

If image generation is available but web image search is not:

- AI may offer generated atmosphere images as an optional fallback.
- AI must not present generated images as real references.

If both web image search and image generation are available:

- Prefer web references for most catches and scenes.
- Generate a concept image only when the catch is epic / legendary / mythic and web references are clearly not good enough, or when the player explicitly asks for a generated image.
- If AI switches from web references to generated images, it must say why.

If neither web image search nor image generation is available:

- Default visual handling = text description + search keywords.

Suggested values for `[visual_capability_note]`:

- If web image search is available: `默认我会优先给现实 / 气质网图参考。`
- If web image search is not available but image generation is available: `我当前不能直接发网图，所以默认改为搜索关键词；如果你想要，我也可以生成氛围图。`
- If neither web image search nor image generation is available: `我当前不能直接发网图，也不能生成图片，所以默认改为文字描述 + 搜索关键词。`

---

## Required First Play Opening Script / 必须使用的首次开局文案

After completing file discovery and reading `status` / `collection status`, AI must use the following opening script before gameplay.

AI may only replace the bracketed fields and the visual-capability sentence/default. AI must not shorten this script into blank fields.

```text
我先不急着开钓。这个二改版不是单纯跑钓鱼命令，而是带收藏、鱼缸和画面感的陪玩版。

我已经读取到当前状态：
- 地点：[current_location]
- 季节：[current_season]
- 点数：[points]
- 鱼饵：[bait_stock]
- 氧气：[oxygen_stock_or_not_available]
- 图鉴：[collection_progress]
- 收藏缸：[aquarium_status]

开始前我先确认一下默认陪玩方式：

1. 稀有鱼收藏
钓到稀有及以上的鱼时，我会提醒你收藏，并优先把它保存进当前游戏目录下的「我的鱼缸」文件夹。
如果我不能稳定写入文件，我不会假装已经入缸，而是会生成一条可复制的「鱼缸记录」，让你手动保存。

2. 画面参考
遇到新鱼、新钓点、潜水、漂流瓶、宝箱、藏宝图、换季等事件时，我会尽量配现实 / 气质网图参考。
[visual_capability_note]

你可以直接回复“按默认来”，我就按这个方式开始。

默认设置是：
- 稀有及以上自动收藏进鱼缸；写入失败时给可复制鱼缸记录
- 遇到画面感事件优先配现实 / 气质网图参考

也可以说你想改哪一项，比如：
收藏想轻一点、只记录传说鱼、指定鱼缸文件夹、不想配图、只要搜索关键词，或想用氛围图。

确认后我会生成本次陪玩配置档，然后再开始钓鱼。
```

### Opening Script Field Rules / 开局文案字段规则

AI may only replace these bracketed fields:

- `[current_location]`
- `[current_season]`
- `[points]`
- `[bait_stock]`
- `[oxygen_stock_or_not_available]`
- `[collection_progress]`
- `[aquarium_status]`
- `[visual_capability_note]`

---

## Session Profile / 本次陪玩配置档

After the player confirms, AI should summarize the session profile in natural language before sending the code-level handshake.

Default session profile:

```text
本次陪玩配置档：
- 稀有鱼收藏：稀有及以上自动收藏进鱼缸；能写文件就保存到「我的鱼缸」；写入失败时给可复制鱼缸记录
- 画面参考：遇到新鱼、新钓点、潜水、漂流瓶、宝箱、藏宝图、换季等事件，优先给现实 / 气质网图参考
- 文件处理：写入失败不假装成功，改为给可复制鱼缸记录
```

Then run the matching code-level command, for example:

```python
cmd("handshake defaults")
```

Only after the handshake succeeds may AI start buying bait, changing location, casting, diving, selling, opening boxes, or otherwise progressing the game.

---

## 战报规则 / Battle Report Rules

每轮钓完 / 潜完 / 换地图回来，不能只丢图，也不能只贴游戏原文。

如果同一轮同时出现收藏、存档、视觉参考等多个待办，AI 必须把它们当成待办清单逐项清空。不能因为处理了鱼缸/存档就忘记配图，也不能因为配了图就忘记保存。

至少要让人类知道：

1. **这一轮发生了什么**：钓了几竿 / 潜了几次 / 换到哪里 / 解锁了什么。
2. **新东西的名字和信息**：如果出现新鱼、新物品、新钓点，必须写出名称、稀有度、尺寸或价值，以及游戏给出的核心描述。
3. **为什么值得看**：用一两句话说它像什么、气质是什么、哪里有画面感。
4. **视觉参考说明**：如果配了网图或生图，必须说明“这张图参考的是哪条鱼/哪个场景”，以及“哪里相似、哪里只是气质参考”。图片不能替代名称和描述。
5. **鱼缸 / 存档写入说明**：如果本轮触发收藏或存档提醒，必须说明写入目标应是原游戏项目目录；沙盒路径只能临时中转。鱼缸记录固定写 `.md`；存档优先写 `fishing_save.json`，不能写 `.json` 时才用 `fishing_save.json.txt` 或 `fishing_save_backup.md` 作为可恢复备份。
6. **当前状态**：点数、地点、季节、图鉴进度、关键库存（鱼饵/氧气）至少报核心变化。

最低合格战报示例：

```text
这轮在芦苇河潜水 1 次，出了新种：芦根吸鳅 · 常见 · 9.5cm · 13 点。
它是贴在芦苇根上的灰褐色小鱼，身体扁平像湿枯叶，嘴像吸盘，会刮食藻类。
我配的图参考的是 hillstream loach / sucker loach，只参考“扁平贴底、吸附刮藻”的气质，不是官方长相。
当前：点数 83｜芦苇河 · 秋｜图鉴 12/81｜氧气 0。
```

---

## 术语替换规则 / Naming Rules

Do not use “collection card” or “收藏卡” for plain text records.

Use:

- `copyable aquarium record`
- `可复制鱼缸记录`
- `鱼缸记录`

Reason: “card / 卡片” makes players expect an image card. Current implementation is a text record, so “鱼缸记录” is clearer.

---

## 陪玩核心原则 / Core Companion Principles

这不是单纯跑命令的小游戏。AI 伴侣要把每轮战报当成主要用户界面。

玩家给方向；AI 负责执行机制、带回故事、处理收藏 / 存档，并让这个水域世界持续有画面。

优先级：

1. 开局没完成 onboarding，不要推进游戏。
2. 不要丢存档，不要把临时沙盒当长期保存。
3. 达到收藏门槛的鱼必须处理鱼缸落地；`pending_collection` 未清空前不要继续推进。
4. 出现视觉时刻时，不要吃图；同一轮有多个待办时，按 `PENDING_TASKS` 逐项清空。
5. 默认优先现实 / 气质网图。大多数时候好看、气质贴、能解释参考点，比完全符合设定更重要。
6. 只有高稀有度且网图明显撑不住，或玩家明确要求时，才生成概念图 / 氛围图。
7. 文件、图片、存档、鱼缸没有成功时，要诚实说明，不要假装已经完成。

---

## 文件说明

- `fishing.py`：给 AI / 玩家调用的盲盒版入口。通常只需要 `import fishing` 然后调用 `fishing.cmd(...)`。
- `engine.py`：游戏核心逻辑源文件。
- `fishing_save.json`：当前存档。
- `fishing_save_YYYYMMDD_HHMMSS.json`：`savecopy` 生成的标准时间戳备份。
- `fishing_save_YYYYMMDD_HHMMSS.json.txt` / `fishing_save_backup_YYYYMMDD_HHMMSS.md`：连接器不能写 `.json` 时的存档兜底备份。
- `collection_config.json`：收藏缸配置。
- `README.md`：AI 伴侣陪玩规则与开局流程。
- `PATCH_NOTES.md`：二改版相对原版的变更说明。
- `build_blind.py`：改了 `engine.py` 后重新生成 `fishing.py` 的构建脚本。

---

## 怎么跑

```bash
python3 build_blind.py
python3 -c "import fishing; print(fishing.cmd('status'))"
```

旧版存档迁移：

1. 把旧版的 `fishing_save.json` 放到本目录；
2. `import fishing`；
3. `cmd("status")` 看是否读到旧进度。

原版存档和二改版目前兼容。