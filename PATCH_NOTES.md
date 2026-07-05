# 二改版说明（相对官方 tutusagi/ai-fishing-game）

在原版 engine.py 基础上加了三块，靠 build_blind.py 重新打包进 fishing.py，AI 玩家只调 fishing.cmd() 就行，不用额外学新接口。

## 1. 存档导出提醒（防止忘记同步到 Drive/本地文件夹）
- 存档里新增字段 `last_export_turn`，记录"上次确认已同步到外部存储时的回合数"。
- 每次 `cmd()` 调用（纯查看类指令除外）都会检查：当前回合 - last_export_turn ≥ 15，就在输出末尾强制打印警告。
- 同步完存档后，调用 `cmd("__exported__")` 更新标记、清掉警告。
- 若环境不能覆盖文件，或要跨窗口/跨机器带走进度，可调用 `cmd("savecopy")`，生成 `fishing_save_YYYYMMDD_HHMMSS.json` 时间戳备份，并同时更新导出标记。
- `__exported__` 与 `savecopy` 现在都走 `_run_one()`，所以单独调用和批量调用（如 `cmd("savecopy; status")`）都有效。

## 2. 收藏缸（collection）
- 新指令：`collection default`（稀有及以上自动收藏）/ `collection off`（不用）/ `collection custom <rarity>`（自定义门槛，可选 common/uncommon/rare/epic/legendary/mythic）/ `collection status`（查看当前方案）。
- 配置存在同目录 `collection_config.json`，跟 `fishing_save.json` 分开存。
- 第一次玩（配置文件不存在）时，任何指令的输出末尾都会带一次性提示，问要不要开启、用什么方案，直到你显式设置过为止。
- 每次抛竿/潜水，只要渔获稀有度达到配置门槛，输出里会自动标注"该进收藏缸了"。

## 3. 画面感提醒（配图用）
- 每次抛竿/潜水，输出末尾会汇总本轮出现的"值得配图/记录"的时刻：新种（🆕）、漂流瓶（📜）、藏宝图碎片/解锁（🧩🗺️）、潜水场景（🤿）、换季（🍃）、宝箱（📦），外加上面第2点的收藏缸提醒。
- 纯查看类指令（status/help/shop/encyclopedia/inventory/look）不会触发这些提醒，避免刷屏噪音。
- 暂不在游戏里写死具体搜图关键词；游戏只提醒“这里值得配图/记录”。具体关键词由当次 AI 根据鱼名、描述和实际搜索结果临场生成，避免未测试关键词导致跑偏。

## 怎么跑
```
python3 build_blind.py   # 改了 engine.py 之后，重新生成 fishing.py
python3 -c "import fishing; print(fishing.cmd('status'))"
```

## 后续要接着改
- 这三块都焊在 `cmd()` 本体和 `_run_one` 的指令分发里，不是外挂脚本——不管从哪个入口调用都躲不掉，这是刻意设计。
- 如果想加新提醒类型（比如"某地点已集齐"），改 `_VISUAL_MARKERS` 或 `_catch_rarity_from_line` 就行。
