# 最新两笔 PR 描述（基线 v4.3.0）

> 两份**互相独立**的 PR 正文，都基于 tag `v4.3.0`，可分别开。按仓库纪律（`AGENTS.md` 三证据 + `PR-REPORT-TEMPLATE.md`），正文只放摘要与报告链接，逐文件/实测/实机细节留在 `docs/` 的报告里。
>
> 需要改的只有：分支名、PR 编号占位、以及勾选状态。

---

# PR ① `fix(voice-input)`: 中文 Windows 下语音输入汉字乱码——子进程协议行改 UTF-8 字节

> **基线**：tag `v4.3.0` → **分支**：`fix/voice-input-gbk-stdout`（按实际填）
> **范围**：5 个文件（实现 1、测试 1、工具 1、文档 2）
> **报告**：[`docs/PR-REPORT-VOICE-INPUT-ENCODING-2026-10-16.md`](docs/PR-REPORT-VOICE-INPUT-ENCODING-2026-10-16.md)

## 摘要

用户实机报告：冻结包里语音输入「阿拉伯数字正常、汉字全乱码」。

根因不是识别模型，是**协议行的编码**：识别子进程用 `sys.stdout` 写结果 JSON，中文 Windows 上 `sys.stdout` 按 GBK/cp936 编码；宿主 `pet/voice_input/engine.py::_parse` 却固定按 UTF-8 解码（`out.decode("utf-8", "replace")`）。ASCII 在两种编码下字节相同，所以数字、路径、JSON 骨架全对；汉字被写成 GBK 字节再按 UTF-8 读，整句变成 `U+FFFD`。

修法是**修生产者、不动消费者**：新增 `worker._write_stdout()`，用 `os.write(1, …)` 直接写 UTF-8 字节，绕开 `sys.stdout` 的文本层。

**为什么开发期一直没暴露**（三条，都是结构性盲区）：

1. `scripts/build_onedir.ps1` 第 78-79 行把整个构建过程钉成 `PYTHONUTF8=1` + `PYTHONIOENCODING=utf-8`；
2. 产物自检 `check_bundle_encoding.py` 扫的是**字面量/资源/文件名**，而故障在运行期子进程的 stdout；
3. 既有探针 `tools/probe_voice_input.py` 只跑源码态（`-m pet.voice_input.worker`），从未跑过冻结包的 `--voice-input-worker`。

## 改动

| 文件 | 增删 | 要点 |
|---|---|---|
| `pet/voice_input/worker.py` | +25 / −10（196→211 行） | 新增 `_write_stdout(payload: bytes)`：`memoryview` + `os.write(1, …)` 写满循环；`_emit` 由「两次 `sys.stdout.write` + `flush`」改为「一次字节写」；`import os`；模块 docstring 的协议段标注 `← UTF-8 字节` |
| `tests/test_voice_input_engine.py` | +49 / −1（189→237 行） | 两条**字节层面**回归：① 真实子进程 + 显式 `PYTHONIOENCODING=gbk` + `env.pop("PYTHONUTF8")`，断言 stdout 可被 UTF-8 严格解码；② 同一批字节交给 `VoiceInputEngine._parse`，断言结果不含 `\ufffd`。另把一条 `capsys` 改为 `capfd`（协议行走 fd 1，`capsys` 只截文本层——这是改动意图的一部分，不是为变绿放宽断言） |
| `tools/probe_voice_input.py` | +53 / −2（124→175 行） | 新增第 5 节「冻结包 worker」：`probe_exe()` 定位产物（`DSH_PET_PROBE_EXE` 可覆盖），`run_frozen()` 在默认区域与 `PYTHONIOENCODING=gbk` 两种环境下各跑一次，打印 stdout 前 16 字节 hex 并断言合法 UTF-8 |
| `docs/PR-REPORT-VOICE-INPUT-ENCODING-2026-10-16.md` | 新增 | 三份交付证据载体 |
| `docs/INDEX.md` | +1 | 按「新文档入场规则第 1 条」登记，并写下两条先例 |

**为什么不在宿主侧兼容 GBK**：`utf-8 → 失败再退 gbk` 能让这台机器看起来正常，但会把「机器的区域设置」变成协议的一部分，且在冻结包/源码态/CI 三种区域下行为不同——这正是本故障的成因。字节写死，消费者口径不动。

**顺带修掉的一个隐患**：冻结的 windowed 包里 `sys.stdout` 可能是 `None`（spec `console=False`），直写 fd 1 不吃这个假设——与 `pet/screen_capture/ocr_worker.py::write_result` 是同一条纪律。

## 证据摘要

- **先红后绿（实测）**：两条新用例改动前都红，且红在正确的点上：
  `UnicodeDecodeError: 'utf-8' codec can't decode byte 0xd3 in position 23: invalid continuation byte`；
  `AssertionError: assert '尚未下载' in '\ufffd\ufffd\ufffd\ufffd…'`。
- **性能**：协议行 142 字节，n=20000，6.092 → **5.357 µs/次**（−0.736 µs，是**减少**系统调用：2 文本层写 + flush → 1 次 `os.write`）。端到端识别 2.14–2.18 s（源码态）/ 2.31–2.38 s（冻结包），增量在测量噪声内，**只说「不变」**，不声称更快。
- **门禁**：`ruff → All checks passed`；语音族 212 passed（含 `test_pr_report_discipline.py`）；全量 `7572 passed / 10 failed / 14 skipped`，那 10 条经**三条独立证据**证明与本改动无关（失败文件 `grep -c voice_input` 全为 0；把 `_write_stdout` 临时改回旧写法后失败集合完全一致；其中一条本就是 `INDEX.md` 已登记的既有红）。
- **实机**：用户本人在冻结包里点麦克风说中文，**汉字正确显示**——修复前同一动作是乱码。

## 已知限制

- `--vad` **仍未接线**（宿主从未传该参数，silero 闸门不在生效路径上，`silero_vad.onnx` 下载了但没人用）。**本次刻意不改**：接上会改变用户可见行为（放音乐时从「幻觉文本」变成「没听到说话」），且实测 VAD 会切掉句尾（同一段 7.54 s 音频，`…这是语音输入的测试。` → `…这是。`）。这是产品口径取舍，应单独决策。
- 真人真麦样本：本次主证据是 SAPI 合成语音（真 16 kHz PCM / 真模型 / 真子进程），真人 GUI 用例另行通过（见报告 5.10）。
- 模型首次下载分支未跑（本机三件套已就位）。

## 风险与回滚

单文件实现改动，无配置项、无迁移、无落盘。回滚即 `git revert`，无残留。唯一的行为面是「子进程 → 宿主」的结果行，宿主解析逻辑、退出码语义（0/2/3）、PCM 走 stdin 不落盘等约束全部不变。

## 检查清单

- [ ] `ruff` 通过
- [ ] 聚焦 + 语音族 212 项通过
- [ ] 全量 `pytest` 通过（或差异已 A/B 归因并登记）
- [ ] `tests/test_pr_report_discipline.py` 通过（报告已登记 INDEX）
- [ ] 三端 CI 绿
- [ ] 冻结包实机复验（`tools/probe_voice_input.py` 第 5 节 + 人工按麦说中文）

---

# PR ② `feat(chat)`: 助手回复朗读（默认关，复用进程内唯一音频通道）

> **基线**：tag `v4.3.0` → **分支**：`feat/chat-reply-voice`（按实际填）
> **范围**：10 个文件（实现 4、测试 2、工具 1、文档 3）
> **报告**：[`docs/PR-REPORT-CHAT-REPLY-VOICE-2026-10-16.md`](docs/PR-REPORT-CHAT-REPLY-VOICE-2026-10-16.md)

## 摘要

把**助手回复念出来**：AI 回复（含识图提问的回答）到达后，按句拆开交给进程内已有的 `VoiceChimeService` 逐步朗读。

- **一个订阅点覆盖全部对话来源**：接在 `ChatService.register_global_finished`（快速对话 / 灵动岛对话 / 完整聊天窗三个入口都经过它），不在三个 UI 里各埋一份。
- **按句滴灌**：音频通道的待播槽是**深度 1 且新覆盖旧**（为报时设计），所以必须问「现在空闲吗」再发下一句，否则多句只剩最后一句。
- **不新起通道**：朗读是这条通道的第四个使用者（报时 / 节日 / 点击自言 / 回复朗读），结构上不可能叠音。
- **只念对话，不念歌词**：歌词链与 `ChatService` 无交集，另有测试守卫挡后人误接。
- **默认关**（`chat_reply_voice_enabled: False`）：关着时不建通道、不订阅、不建定时器，行为与 v4.3.0 完全一致。

## 改动

| 文件 | 增删 | 要点 |
|---|---|---|
| `pet/chat_reply_voice.py` | +177（新增） | `clean_for_speech()`（去代码块/链接/markdown 记号/表情，代码块留「代码略」）、`split_sentences()`（句末标点切句 + 超长硬切 80 字）、`ChatReplyReader`（订阅、队列、`pump()`、`enabled` 热读）。模块顶层**不** import `pet.chat`，改惰性带守卫——无聊天打包变体里 `pet.chat` 不存在，本模块仍须可 import |
| `pet/voice_chime_service.py` | +12 | 把既有 `_channel_idle()` 提成公开 `channel_idle()`：没有它就无法按句滴灌（深度 1 的槽会把多句吃掉） |
| `pet/app.py` | +35 / −2 | `_chime_wanted()` 增加「朗读需要通道」；新增 `_ensure_chat_reply_reader()` / `_sync_chat_reply_reader()` 并与通道同生共死；**先摘订阅再停通道**，免得收口期还有回复进队列 |
| `pet/chat/ai_settings_page.py` | +15 | 「AI 与对话」域新增「助手回复朗读」一节，复用同页既有 `SettingsSection`/`SettingRow`，未新增布局代码 |
| `tests/test_chat_reply_voice.py` | +180（新增） | 9 项：两个纯函数、滴灌顺序与「通道未就绪不丢队列」、新回复覆盖旧队列、默认关门禁、启停订阅幂等、`channel_idle` 公开面守卫、三个歌词模块零引用守卫 |
| `tests/test_config_schema.py` | +1 | 新键进 schema 快照（另两处登记在 `pet/config.py` 的默认值与 reload 白名单） |
| `tools/probe_chat_reply_voice.py` | +148（新增） | 真机探针（真 Config / 真通道 / 真 edge-tts 合成 / 真播放），产出报告 §五 的原始输出，可复现 |
| 文档 ×3 | — | 本报告、`SETTINGS-CHANGE-GATES.md`（准入逐条对照 + 准出五项）、`INDEX.md` 登记 |

**关键设计取舍**：

- **为什么不自己算时长**：靠估算音频长度决定何时发下一句，会在网络抖动/合成失败时错位；用服务自己的占用判据是唯一与实际播放一致的信号。
- **未改动（看起来相关但故意没动）**：`chat/widgets.py`、`quick_chat.py`、`island_chat.py`（不在 UI 埋朗读）；`music_lyric_controller.py`、`music_lyric.py`、`now_playing.py`（一个字没动，歌词不接是设计）；右键菜单（设置项已能表达该偏好，再加是重复入口）。

## 证据摘要

- **端到端（真合成 + 真播放，音量 15）**：三句总耗时 **14.70 s**，相邻出声起点间隔 **5.25 s** ≈ 前段音频 3.26 s + 后段合成延迟 → 证明**串行、无叠音**。
- **覆盖发生在入队阶段**：旧队列 2 句 → 只合成 **2 个** mp3（旧的甲 + 新的），旧的乙连合成都未发生（缓存目录文件级差分佐证），省掉无用的 TTS 请求。
- **文本处理成本**（n=300）：45 字 **0.008 ms**、1260 字 **0.104 ms**、6400 字 **0.526 ms**；每条回复一次。之后每句一次 `speak()`，复用既有合成/缓存/播放路径，与语音报时同量级（本机单句合成约 1.5–2.2 s，主导是 edge-tts 网络往返）。
- **默认关**：通道 `None`、订阅数 `0`、队列空——不是「关着也建好只是不念」。
- **门禁**：`ruff → All checks passed`；本功能 9 passed；受影响既有族 124 passed；**断言有效性**：删掉 schema 里的键登记后重跑，2 条精准变红，还原后 64 passed。

## 已知限制（如实登记）

- **全量套件本轮未跑完**：本机首次拉齐环境后运行 >45 分钟仍未结束，未取到结论，**不作为已通过**。同一工作树上，同日另一份报告完成了全量（7572 passed / 10 failed，10 条经 A/B 证明无关），可作旁证，但那不是本 PR 自己的门禁记录。
- **未验**：设置页新增行的视觉验收（窄宽/放大字体/明暗主题）、完整 GUI 里「发消息 → 听到回复」的人工端到端、macOS/Linux（本机只有 Windows）。
- **刻意不做**：流式生成过程中边收边念（入口在 `ChatService.delta`，但要处理「同一句被增量改写」，另开一轮）；长回复不设上限（6400 字约 100 句，若反馈嫌长，下一步加「只念前 N 句」策略）；麦克风 ASR 不在本轮范围。
- **未做**：右键菜单快捷开关（理由见上，建议复用同一个键另开一轮）。

## 风险与回滚

默认关，关着时零行为变化（探针第 1 步实测）。开着时只新增「说话」这一种效果，不改任何既有朗读语义。新增键无迁移（缺失即 `False`，旧版本读到会忽略）。`git revert` 后无残留落盘，唯一副作用是 `voice_chime_cache` 里多几个 mp3（既有 LRU 自行清理）。

## 检查清单

- [ ] `ruff` 通过
- [ ] 本功能 9 项 + 受影响族 124 项通过
- [ ] 全量 `pytest`（**待跑**；跑完把结论补进报告，或注明旁证来源）
- [ ] `tests/test_pr_report_discipline.py` 通过（报告已登记 INDEX）
- [ ] 三端 CI 绿
- [ ] 设置页视觉验收 + 完整 GUI 人工确认（**待补**）

---

<!-- 不进 PR 正文 -->

**合并成一个 PR 的话**：两条改动无代码耦合（一个在 `pet/voice_input/`、一个在 `pet/chat*` + `pet/voice_chime_service.py`），但语义上是「修 bug」+「加功能」，评审关注点不同，建议分开。若确实要合，标题用 `feat(voice): 语音输入编码修复 + 助手回复朗读`，正文把两份「摘要」拼起来、证据各自指向报告即可——不要合并两边的测试结论，避免用一份的门禁证明另一份。

**注意**：两份报告都写明本机无 git 元数据，增删行数由 `wc -l` 差值或 `difflib.unified_diff` 得出（等价 `git diff --numstat`），PR 里引用数字时别声称是从 git 取的。
