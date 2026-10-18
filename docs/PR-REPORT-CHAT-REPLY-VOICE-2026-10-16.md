# PR 报告：助手回复朗读（Chat Reply Voice）

> **基线**：tag `v4.3.0`（源码取自官方 tarball `codeload.github.com/MerZlin/dsh-pet-indesktop/tar.gz/refs/tags/v4.3.0`；本机无 git 元数据，故无 commit sha）
> **分支**：无（本地补丁，未提交；下同）
> **日期**：2026-10-16
> **范围**：10 个文件（实现 4、测试 2、工具 1、文档 3）
> **关联**：`docs/SETTINGS-CHANGE-GATES.md` 的 2026-10-16 条目；上一轮关于「主动识屏为何不触发」的排查与本报告无共同代码路径，仅共用 `VoiceChimeService` 这条音频通道。

## 一、核心特性

把**助手的对话回复念出来**：AI 回复（含识图提问的回答）到达后，按句拆开交给进程内唯一的那条音频通道逐步朗读。

| # | 能力 | 说明 |
|---|---|---|
| 1 | 回复自动朗读 | 订阅 `ChatService.register_global_finished`（快速对话 / 灵动岛对话 / 完整聊天窗三个入口都经过它），一个订阅点覆盖全部对话来源 |
| 2 | 按句滴灌 | 音频通道的待播槽是**深度 1 且新覆盖旧**（为报时设计，避免堆过期内容），所以朗读必须问「现在空闲吗」再发下一句，不能一次全塞 |
| 3 | 新回复覆盖旧队列 | 未念完的旧回复直接作废——**实测证明它连合成都不会发生**（见 §五第 4 步） |
| 4 | 只念对话，不念歌词 | 歌词走 `music_lyric_controller` 那条链，与 `ChatService` 无交集；本模块另加一条回归守卫防后人误接 |
| 5 | 默认关 | 新增键 `chat_reply_voice_enabled` 默认 `False`：关着时运行行为与 v4.3.0 完全一致 |

**红线 / 不变量**：

1. **不新起音频通道**——朗读只用 `VoiceChimeService`，因此结构上不可能与语音报时 / 节日语音 / 点击自言自语叠音；
2. **默认关不许付出代价**——关着时不创建通道、不订阅、不建定时器、无新线程；
3. **歌词不得进入朗读链**——由「来源只有 `ChatService`」保证，并由测试守卫；
4. **免打扰语义沿用既有契约**——`speak()` 默认 `respect_quiet=True`，免打扰（含全屏等效）期间丢弃不补播。

## 二、修改文件说明

数字口径：与上游 tag `v4.3.0` 原文逐文件文本 diff（本机无 git，故用 `difflib.unified_diff` 统计，等价 `git diff --numstat`）；新文件的行数即「增」的行数。

### 实现

| 文件 | 增删 | 改动意图 |
|---|---|---|
| `pet/chat_reply_voice.py` | +177 / −0（新增） | 新模块：`clean_for_speech()`（去代码块/链接/markdown 记号/表情，代码块留「代码略」提示）、`split_sentences()`（句末标点切句 + 超长硬切 80 字）、`ChatReplyReader`（订阅、队列、`pump()` 滴灌、`enabled` 热读配置）。模块顶层**不** import `pet.chat`，改为 `_chat_service()` 惰性带守卫——无聊天打包变体里 `pet.chat` 不存在，本模块仍须可 import |
| `pet/voice_chime_service.py` | +12 / −0 | 新增公开 `channel_idle()`：把既有的 `_channel_idle()`（无合成在飞 ∧ 播放器未接管）暴露成公开判据。**没有它就无法按句滴灌**——深度 1 的新覆盖旧槽会把多句吃掉只剩最后一句 |
| `pet/app.py` | +35 / −2 | ① `_chime_wanted()` 增加「对话朗读需要通道」这一项（四者共用一条通道，通道是否存在取决于其中之一）；② 新增 `_ensure_chat_reply_reader()` / `_sync_chat_reply_reader()`，并在 `_sync_chime_service()` 末尾连带同步（读者与通道同生共死）；③ 生产退出与 `_shutdown_live_for_tests` 两处收口：**先摘订阅再停通道**，免得收口期还有回复进队列 |
| `pet/chat/ai_settings_page.py` | +15 / −0 | 设置页「AI 与对话」域新增一节「助手回复朗读」：`ToggleSwitch` + `SettingRow("chat_reply_voice_enabled", ...)` + 保存路径写盘。复用同页既有的 `SettingsSection`/`SettingRow`，未新增布局代码 |

### 测试

| 文件 | 增删 | 覆盖 |
|---|---|---|
| `tests/test_chat_reply_voice.py` | +180 / −0（新增） | 9 项：两个纯函数（清洗保留「代码略」提示、切句与硬切）、滴灌顺序与「通道未就绪不丢队列」、新回复覆盖旧队列、默认关门禁、启停订阅幂等、`VoiceChimeService.channel_idle` 公开面守卫、三个歌词模块零引用的隔离守卫 |
| `tests/test_config_schema.py` | +1 / −0 | 新键进 schema 快照（三处登记之一；另两处在 `pet/config.py` 的默认值与 reload 白名单） |

### 工具

| 文件 | 增删 | 用途 |
|---|---|---|
| `tools/probe_chat_reply_voice.py` | +148 / −0（新增） | 真机探针（真 Config / 真音频通道 / 真 edge-tts 合成 / 真播放），产出 §五 的原始输出，可复现 |

### 文档

| 文件 | 增删 | 内容 |
|---|---|---|
| `docs/PR-REPORT-CHAT-REPLY-VOICE-2026-10-16.md` | 新增 | 本报告 |
| `docs/SETTINGS-CHANGE-GATES.md` | +39 / −0 | 新设置的准入逐条对照 + 准入记录字段表 + 准出五项结论 |
| `docs/INDEX.md` | +1 / −0 | 本报告登记一行（新文档入场规则第 1 条） |

### 未改动（看起来相关但故意没动）

- `pet/chat/widgets.py`、`pet/quick_chat.py`、`pet/island_chat.py`：**不**在三个 UI 里分别埋朗读——那样是三份实现且会随新 UI 漏接；统一走 `ChatService` 的进程级回调。
- `pet/music_lyric_controller.py` / `music_lyric.py` / `now_playing.py`：**一个字没动**。歌词不接进来是设计，不是遗漏；测试里有守卫挡「以后有人接」。
- `pet/context_menus/registry.py`：没加右键菜单开关。设置项已能表达该偏好，菜单里再加一个是重复入口（违背设置准入第 4 条「归属唯一」）。如需快捷开关，建议做成菜单 toggle 但**复用同一个键**，另开一轮。

## 三、实现要点

**为什么接在 `ChatService` 而不是 UI 层**：`pet/chat/service.py` 里已有一个进程级回复完成广播（`_global_finished_listeners`，注释写明是给灵动岛事件动效用的：`AppShell` 注册一次，回调总在 GUI 线程执行）。三个对话入口都构造自己的 `ChatService`，但 `_finished()` 都走这份广播——一个订阅点覆盖全部来源，且**天然排除歌词**（歌词不经过 `ChatService`）。

**为什么需要 `channel_idle()`**：`VoiceChimeService.speak()` 的排队语义是「通道不空闲就进深度 1 的待播槽，**后来的覆盖前面的**」——这是为报时设计的（避免把过期提醒堆起来）。朗读一条回复会拆出多句，若一次全提交，只有最后一句会活下来。所以朗读自带队列 + 250ms 定时器，等通道真空闲再发下一句。

**为什么不自己算时长**：靠估算音频长度来决定何时发下一句会在网络抖动 / 合成失败时错位；用服务自己的占用判据（`_channel_idle()`）是唯一与实际播放一致的信号，故把它提成公开面而不是绕过去读私有字段。

**失败路径**：通道不存在（未开启语音报时且其它语音功能也关）→ 朗读只保留队列不丢，等宿主 `_sync_chime_service()` 建出通道后继续；`speak()` 返回 False（文本空）→ 忽略；edge-tts 不可用 → 服务只记日志不出声，调用方不受影响。全链路无异常外溢（`ChatService` 的广播本身 `try/except` 包住每个订阅者）。

**配置语义**：`ChatReplyReader.enabled` **热读**配置，所以开关变化只需摘/挂订阅（`_sync_chat_reply_reader`），不必重建对象；通道的生命周期另行由 `_chime_wanted()` 决定。

## 四、性能分析

**方法（可复现）**：`python tools/` 同级微基准（清洗+切句 300 次取均值，见下命令）；真机探针 `python tools/probe_chat_reply_voice.py`（真通道 / 真合成 / 真播放）。
环境：Windows 11 22621.5624 / Python 3.12.14 / PySide6 6.12.0 / edge-tts 7.3.1。

```powershell
$vpy = 'G:\DPW\pet-venv\Scripts\python.exe'   # 本机按 requirements.txt 装的开发环境
& $vpy -m pytest -q tests/test_chat_reply_voice.py
& $vpy tools\probe_chat_reply_voice.py        # 需要真实音频设备，勿设 QT_QPA_PLATFORM=offscreen
```

| 指标 | 实测 | 归属 |
|---|---|---|
| 清洗+切句，45 字回复 | **0.008 ms/次**（n=300） | 新增（每条回复一次） |
| 清洗+切句，1260 字回复 | **0.104 ms/次**（n=300） | 新增（同上） |
| 清洗+切句，6400 字回复 | **0.526 ms/次**（n=300） | 新增（同上） |
| 三句回复端到端（真合成+真播放，音量 15） | **14.70 s**（三段音频合计 9.5 s + 合成延迟） | 新增路径的真实成本 |
| 相邻两句出声起点间隔 | **5.25 s**（≈ 前一段音频 3.26 s + 后一段合成延迟） | 证明**串行、无叠音** |
| 新回复覆盖旧队列 | 旧队列 2 句 → 只合成 **2 个** mp3（旧的甲 + 新的），旧的乙**未被合成** | 覆盖发生在入队阶段，省掉了无用的 TTS 网络请求 |
| 默认关（`chat_reply_voice_enabled=false`） | 通道 `None`、订阅数 `0`、队列空 | 既有路径 |

**结论**（逐条）：① **稳态开销**：默认关时只多一次 `config.get()`（微秒级），不建定时器、无新线程、订阅数为 0；② **新增路径成本与触发频率**：每条助手回复触发一次文本处理（≤0.53 ms，6400 字回复），其后每句一次 `speak()`——该方法**复用既有的**合成/缓存/播放路径，成本与「语音报时」同量级（本机实测单句合成约 1.5–2.2 s，主要由 edge-tts 网络往返决定）；有队列时 250 ms 一次 `pump()`（判据是 `channel_idle()`，无队列时定时器立即停止）；③ **新增系统调用 / 网络 / 磁盘 / 线程**：无新增线程；无新增网络通道（edge-tts 是既有依赖，mp3 落 `voice_chime_cache` 复用既有 LRU 与清理）；无新增文件句柄；④ **内存与缓存**：队列上限是「本条回复的句数」（6400 字 ≈ 100 句短字符串），无累积；缓存沿用既有 `voice_chime_cache`，本探针跑出的 5 个 mp3 合计 71 KB。

## 五、实机运行记录

**本机真实环境**（不是 CI / 不是 mock）：Windows 11 22621.5624，真实声卡，真实 edge-tts 联网合成，`tools/probe_chat_reply_voice.py` 原始输出（音量压到 15，避免测试太吵）：

```
== 1. 开关关（默认值） ==
  chat_reply_voice_enabled = False
  _chime_wanted() = False
  音频通道 = None
  朗读订阅数 = 0
== 2. 开关开 ==
  音频通道 = VoiceChimeService
  朗读器 = ChatReplyReader
  朗读订阅数 = 1
  通道空闲 = True
== 3. 真回复 → 逐句朗读（真 edge-tts 合成 + 真播放，音量 15） ==
  提交回复 40 字
  入队后待读 = 2
  [三句] 出声段起始数=2 起始时刻=[0.0, 5.25] 相邻间隔=[5.25] 待读剩余=0 总耗时=14.70s
== 4. 新回复覆盖旧队列 ==
  旧回复入队后待读=1，新回复到来后待读=1
  [覆盖] 出声段起始数=2 起始时刻=[0.0, 2.52] 相邻间隔=[2.52] 待读剩余=0 总耗时=6.00s
== 5. 歌词链隔离（结构性） ==
  music_lyric_controller.py    引用朗读模块=False 使用朗读标签=False
  music_lyric.py               引用朗读模块=False 使用朗读标签=False
  now_playing.py               引用朗读模块=False 使用朗读标签=False
== 收口 ==
  朗读订阅数 = 0
```

配套的**文件级证据**（真实合成产物，`.probe-cfg/.../voice_chime_cache/`）：

```
  KB Name
 8.2 322443fcef8713c2.mp3      <- 第 4 步「旧的甲」
 7.3 5e3410fcf42a9b32.mp3      <- 第 4 步「新的」
18.7 c6a303d8b3891df1.mp3      <- 第 3 步第 1 句
17.7 d75e065112fdfd37.mp3      <- 第 3 步第 2 句
19.1 dc934667b09d7ed4.mp3      <- 第 3 步第 3 句
```

- **根因/前提的现场复现**：第 3 步三段 mp3 的**时长**（ffmpeg 报 3.26 / 3.19 / 3.02 s）与总耗时 14.70 s、相邻起点间隔 5.25 s 相互印证——如果三句被并行塞进同一条通道，总耗时会接近最长一段而不是三段之和。第 4 步只有 2 个 mp3 产出，直接证明「旧的乙」在入队阶段就被丢弃（不是合成后被丢弃）。
- **用户可见行为的确认**：朗读确实从本机扬声器出声（5 段中文，音量 15）；**「助手回复到达即自动念」这一条需要真人在完整 UI 里确认**，本报告不替用户断言（见 §七）。
- **边界与失败路径**：第 1 步就是「本该不出声」的实机观察——默认关时通道为 `None`、订阅数 0，连队列都不存在；收口后订阅数回到 0（不会跨会话泄漏）。
- **无法自动验证的能力**：① 完整 GUI 里的「发一条消息 → 听到回复」需要真实 API Key 与人工操作，未做；② 设置页新增行的视觉验收（窄宽 / 放大字体 / 明暗主题）未做——本机未启动完整设置窗，属未验证项；③ macOS / Linux 未验（本机只有 Windows，与仓库既有口径一致：不推断未执行平台的结果）。

## 六、测试与验证

| 门 | 命令 | 结果 |
|---|---|---|
| 静态检查 | `python -m ruff check pet/ tests/` | **All checks passed!** |
| 聚焦（本功能） | `pytest -q tests/test_chat_reply_voice.py` | **9 passed** |
| 受影响既有族 | `pytest -q tests/test_config_schema.py tests/test_settings_page_guide.py tests/test_settings_reset_defaults.py tests/test_runtime_dependencies.py` | **124 passed** |
| 断言有效性 | 删掉 `tests/test_config_schema.py` 里的 `chat_reply_voice_enabled` 登记后重跑 | **2 failed**（`test_defaults_snapshot_matches_current`、`test_reload_whitelist_snapshot_matches_current`）→ 还原后 **64 passed**；证明三处登记确实被测试强制 |
| 真机探针 | `python tools/probe_chat_reply_voice.py` | 见 §五 |
| 全量 | `python -m pytest -q` | **未完成**：本机首次拉齐环境后运行超过 45 分钟仍未结束（7514 项，含真实设备/网络相关用例），本轮未取到全量结论——如实登记，不作为已通过 |
| 架构红线 | `tests/test_architecture.py` | 见下轮补记（含在全量套件内） |

## 七、已知限制与后续

- **未做**：麦克风语音输入（ASR）。已定方案为「首次使用时下载 SenseVoice 模型到缓存目录」+ `sherpa-onnx`，不在本轮交付范围内。
- **未做**：设置页新增行的视觉验收（截图）与完整 GUI 的端到端人工确认。这两项需要真人在本机操作，属未验证项。
- **未做**：右键菜单快捷开关（理由见 §二「未改动」）。
- **流式朗读**：当前等回复**完成**后整段按句念。若以后要在流式生成过程中边收边念（更早出声），入口已在 `ChatService.delta`，但需要处理「同一句被增量改写」的边界，本轮刻意不做。
- **长回复**：6400 字回复会拆出约 100 句、全部念完需要十几分钟。当前不设上限（音量/节奏由用户自己开关控制）；如果反馈嫌长，下一步可加「只念前 N 句」或「只念当前会话最后一条」的策略开关。

## 八、风险与回滚

- **影响面**：默认关，关着时零行为变化（§五第 1 步实测）；开着时只新增「说话」这一种效果，不改任何既有朗读语义（报时 / 节日 / 点击自言的优先级与队列契约都未改，朗读只是这条通道的第四个使用者）。
- **开关**：`chat_reply_voice_enabled`（设置页「AI 与对话 → 助手回复朗读」，或直接写配置）。
- **配置迁移**：新增键，无迁移；缺失即 `False`。旧版本读到此键会忽略（未知键不报错）。
- **回滚**：`git revert` 后无残留落盘数据——唯一副作用是 `voice_chime_cache` 里多几个 mp3（既有 LRU 会自行清理），以及配置文件里多一个被忽略的键。
