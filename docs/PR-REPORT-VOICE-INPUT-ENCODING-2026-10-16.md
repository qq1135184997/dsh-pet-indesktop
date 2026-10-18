# PR 报告：语音输入汉字乱码（协议行区域编码）— 2026-10-16

> **基线**：本目录**不是 git 工作树**（`git rev-parse` → `fatal: not a git repository`），
> 无法给出 `git diff --numstat` 或 `head_sha`。本报告的增删行数由**改动前 `wc -l` 实测值**
> 与改动后实测值求差（改动前的 6 个文件行数在动手前已记录），并在下文逐条给出计数依据。
> **分支**：无（非 git 工作树）　**日期**：2026-10-16
> **范围**：5 个文件（实现 1、测试 1、工具 1、文档 2）
> **关联**：本版集成交付汇总见 [`PR-REPORT-CHAT-REPLY-VOICE-2026-10-16.md`](PR-REPORT-CHAT-REPLY-VOICE-2026-10-16.md)（同一批交付的「回复朗读」一侧）；本档接手自 2026-10-16 的语音输入验证档。

## 一、核心特性

**用户报告的现象**（实机，冻结包 `dsh-pet-standalone-webm-chat.exe`）：

> 「目前是语音输入阿拉伯数字正常，汉字会变成乱码」

**根因（本次已定位并实测复现）**：语音识别的**子进程**用 `sys.stdout` 写那一行结果 JSON，
而 `sys.stdout` 在 Windows 上按**区域编码**（中文机器 = GBK/cp936）编码；**宿主**却固定按
**UTF-8** 解码（`pet/voice_input/engine.py::_parse` 的 `out.decode("utf-8", "replace")`）。

于是：ASCII（阿拉伯数字、路径、JSON 骨架）在两种编码下字节相同 → **一切正常**；汉字被写成
GBK 字节、再按 UTF-8 解码 → 整句变成 `U+FFFD` 替换字符。**这正是"数字正常、汉字乱码"的由来**，
也是它能在开发机上一直不被发现的原因：

* `scripts/build_onedir.ps1` 第 78-79 行把**整个构建过程**强制成 `PYTHONUTF8=1` +
  `PYTHONIOENCODING=utf-8`——构建期与构建机上的任何冒烟都看不到这个问题；
* 打包产物自带的中文编码自检（`check_bundle_encoding.py`）扫的是**产物里的字面量/资源/文件名**，
  而本故障发生在**运行期子进程的 stdout**，不在它的扫描面内；
* 既有的真机探针 `tools/probe_voice_input.py` **只跑源码态**（`-m pet.voice_input.worker`），
  从未跑过冻结包的 `--voice-input-worker` 子进程。

| # | 能力 | 说明 |
|---|---|---|
| 1 | 协议行改为写 **UTF-8 字节** | `worker._write_stdout()` 直接写 fd 1，绕开 `sys.stdout` 的区域编码 |
| 2 | 两条**字节层面**的子进程回归测试 | 显式把子进程区域钉成 GBK，断言 stdout 是合法 UTF-8、且宿主解析不含替换字符 |
| 3 | 真机探针补上**冻结包路径** | `tools/probe_voice_input.py` 新增第 5 节，覆盖本次故障所在的 exe + GBK 区域条件 |

**红线 / 不变量**：

* 宿主 `engine._parse` 的解码口径（UTF-8）**不变**——修的是生产者，不是放宽消费者；
* 子进程仍然**不 import PySide**（`models.py` 纯标准库那条约束不变）；
* 结果 JSON 的字段与退出码语义（`0` / `2` 模型缺失 / `3` 识别期失败）逐字不变；
* 不新增线程、网络、磁盘写入；PCM 仍然只走 stdin 管道，不落盘。

## 二、修改文件说明

> 本目录不是 git 仓库，`git diff --numstat` 不可用。下表增删行数的算法：**净变化** =
> 改动前实测 `wc -l` − 改动后实测 `wc -l`（两个值都在本机命令行里取过）；**增/删拆分** =
> 对每个改动块逐行计数（改动块已在上文「改动前 → 改动后」对照中逐条列出）。

### 实现

| 文件 | 增删 | 改动意图 |
|---|---|---|
| `pet/voice_input/worker.py` | **+25 / −10**（196 → 211 行，净 +15） | 见下 |

`pet/voice_input/worker.py` 逐块说明：

1. **新增 `_write_stdout(payload: bytes)`**（+12，含 docstring）：`memoryview` + `os.write(1, …)`
   循环写满，把协议行按 **UTF-8 字节**写进 fd 1。绕开 `sys.stdout` 的文本层是**两件事**同时解决：
   ① 区域编码不再是协议的一部分；② 冻结的 **windowed** 包里 `sys.stdout` 可能是 `None`
   （spec 第 52 行 `console=False`），直写 fd 1 不吃这个假设。后一条与先例
   `pet/screen_capture/ocr_worker.py::write_result`（「不经过 `sys.stdout`：冻结的 windowed 包里它可能是 None」）
   是**同一条纪律**——本模块开头自称与 OCR 那条链「逐字对齐」，之前只在命令行的 split 上对齐了，
   在**输出通道**上没有。
2. **`_emit` 改写**（+6 / −9）：原来 `sys.stdout.write(json) + write("\n") + flush()`（两次文本层写 + 一次 flush），
   现在 `_write_stdout(json.dumps(…).encode("utf-8") + b"\n")`（一次字节写）。
3. **`import os`**（+1）；模块 docstring 的协议段补一句 `← **UTF-8 字节**` 并新增 5 行说明（+5 / −1 改行），
   把「为什么不能走 `sys.stdout`」写在协议定义旁边而不是藏在实现里。

**为什么不在宿主侧兼容 GBK**（明确取舍）：宿主「先按 UTF-8 解、失败再退回 GBK」也能让这台机器
看起来正常，但那会让**协议依赖机器的区域设置**，并且在冻结包/源码态/CI 三种区域下行为不同——
正是本次故障的成因。修生产者、把字节写死，消费者口径不动。

### 测试

| 文件 | 增删 | 覆盖 |
|---|---|---|
| `tests/test_voice_input_engine.py` | **+49 / −1**（189 → 237 行，净 +48） | 见下 |

1. **`test_worker_stdout_is_utf8_even_when_locale_is_gbk`**（新增）：在**真实子进程**里跑
   `-m pet.voice_input.worker`，显式 `env.pop("PYTHONUTF8")` + `PYTHONIOENCODING=gbk`
   —— 复现「中文 Windows + 管道」这个真实条件，使断言**与跑测试的机器区域无关**。
   模型目录指向不存在的路径，所以**不加载 228 MB 模型**（≤0.7 s），符合本文件「确定性、不碰模型/麦克风」的既有约束。
   断言：`proc.stdout.decode("utf-8")`（非法即 `UnicodeDecodeError`）+ 退出码 2 + 文案含「尚未下载」。
2. **`test_engine_parses_worker_output_without_mojibake`**（新增）：把**同一批字节**交给宿主的
   `VoiceInputEngine._parse`，断言 `"\ufffd" not in result.error`。与上一条合起来才是完整回归：
   生产者写 UTF-8 ∧ 消费者按 UTF-8 读。
3. **`test_worker_main_reports_missing_model_as_json`**（+2 注释 / 改 1 行）：`capsys` → **`capfd`**。
   协议行现在走 fd 1 而不过 `sys.stdout` 文本层，`capsys` 只截文本层 —— 这是**改动意图的一部分**
   （测试必须跟着协议的落点走），不是为了让测试变绿而放宽断言。改动后有注释写明为什么是 `capfd`。

**先红后绿（实测）**：两条新用例在改动前**都红**，且红在正确的点上 ——

```
E       UnicodeDecodeError: 'utf-8' codec can't decode byte 0xd3 in position 23: invalid continuation byte
E       AssertionError: assert '\u5c1a\u672a\u4e0b\u8f7d' in '\ufffd\ufffd\ufffd\ufffd\u0123\ufffd\ufffd…'
2 failed, 15 deselected in 0.65s
```

改动后：`39 passed`（engine + controller + mic + ui），语音族全量 `63 passed`；
连同 `test_pr_report_discipline.py` 的口径为 `212 passed`（见 5.8）。

### 文档

| 文件 | 增删 | 改动意图 |
|---|---|---|
| `docs/PR-REPORT-VOICE-INPUT-ENCODING-2026-10-16.md` | 新增（本文件，363 行） | 三份交付证据（本文件即第 2、3 份的载体） |
| `docs/INDEX.md` | **+1 / −0** | 按「新文档入场规则第 1 条」在「PR 报告存档」登记本报告，并写明**何时必读**与两条**先例**（协议通道写字节、子进程 stdout 测试要钉死 `PYTHONIOENCODING`） |

### 工具

| 文件 | 增删 | 覆盖 |
|---|---|---|
| `tools/probe_voice_input.py` | **+53 / −2**（124 → 175 行，净 +51） | 见下 |

新增第 5 节「冻结包 worker（2026-10-16 汉字乱码故障的路径）」：`probe_exe()` 定位产物 exe
（可用 `DSH_PET_PROBE_EXE` 覆盖），`run_frozen()` 在**两种区域环境**下各跑一次（默认区域 /
`PYTHONIOENCODING=gbk`），打印 stdout 前 16 字节的 hex 并断言它是合法 UTF-8、且文本不含替换字符。
**为什么必须补这一节**：这份探针本来是为「实机运行记录」提供原始输出的工具，但它只覆盖源码态，
所以它对本次故障**结构性失明**——补上冻结路径，这个盲区才不会在下次复发时再次漏过。

**未改动但相关的既有事实（如实登记，不在本次改动内）**：宿主从未向子进程传 `--vad`。`worker.py` 的 silero VAD 闸门（以及 `controller._on_recognized` 里 `speech=False` →
「没听到说话（只听到背景音）」那条分支）因而不在生效路径上；`silero_vad.onnx` 实际未被使用。
详见下文「四、遗留问题」。

## 三、性能分析

**测量环境**：Windows，`G:\DPW\pet-venv\Scripts\python.exe`（Python 3.12.14），
同一台机器，构建产物 `dist-onedir\dsh-pet-standalone-webm-chat`（exe mtime 2026-10-16 13:05:01）。

### 3.1 新增路径成本（微基准，`n=20000` 每次）

协议行长度 142 字节（本条真机识别结果的 JSON）。旧路径逐字复刻改动前的写法
（`sys.stdout.write` ×2 + `flush`），新路径调用改动后的 `worker._emit`；两者都写进同一个临时文件 fd。

| 路径 | µs/次 |
|---|---|
| 改动前 `sys.stdout.write` ×2 + `flush` | 6.092 |
| 改动后 `os.write(1, …)`（单次） | **5.357** |
| 差 | **−0.736 µs** |

新的写法**不慢反快**：原来要走两次文本层写 + 一次显式 flush（底层多次 `write` 系统调用），
现在是一次字节写。**如实登记**：这条微基准量的是「构造 + 写」的 CPU 成本，不含系统调用被
真正调度到管道的开销；即便按 `os.write` 一次系统调用计，量级也不变。

### 3.2 稳态开销与触发频率

* **触发频率**：`_emit` 每次识别**只调一次**（成功/失败/无语音各一条路径）。语音输入是低频动作，
  稳态（没人按麦克风时）**零调用**——本改动不落在任何 tick/事件循环路径上。
* **新增的系统调用**：**没有新增**，而是**减少**（2 文本层写 + flush → 1 `os.write`）。
* **新增的网络 / 磁盘 / 线程**：**均无**。PCM 仍只走 stdin 管道；子进程仍是「用完即退」。
* **内存**：每次识别多一份 ~142 字节的 `bytes` 副本（`payload.encode("utf-8")`），
  随函数返回即回收；无缓存、无常驻结构。

### 3.3 端到端识别耗时（未受影响的既有成本，如实给出）

7.54 s 音频、SenseVoice int8、`--threads 2`：

| 路径 | 墙钟 |
|---|---|
| 源码态（`-m pet.voice_input.worker`） | 2.14 – 2.18 s |
| 冻结包（`exe --voice-input-worker`） | 2.31 – 2.38 s |

耗时由「进程启动 + 加载 228 MB 模型（子进程每次重载）」主导（`seconds` 字段报的识别本身只有
1.02 – 1.08 s）。本改动在这个量级下不可分辨；两次同路径测量的离散（±0.07 s）已大于本改动
的微秒级成本。**不声称"更快"**：这里只能说"不变"。

## 四、遗留问题（**未修**，如实登记）

**宿主从未传 `--vad`，silero VAD 闸门不在生效路径上。** 证据：

```
$ grep -rn '"--vad"' pet tests packaging        # 只有 docstring 与 worker 的 argparse
pet/voice_input/worker.py:55:    parser.add_argument("--vad", default="")
$ grep -rn "vad" pet/voice_input/engine.py pet/voice_input/controller.py   # 无输出
```

* `engine.VoiceInputEngine._argv()` 只拼 `--model-dir / --language / --threads`，**没有 `--vad`**；
  类上也**没有** `vad` 字段。`controller` 构造 `VoiceInputEngine(self.model_dir)`，同样没传。
* 受影响的是**已写好的能力**：`worker.transcribe(vad=…)` 的闸门与
  `controller._on_recognized` 里 `speech=False` → 「没听到说话（只听到背景音）」这条分支
  **当前不可达**；`models.VAD_FILE`（`silero_vad.onnx`，643,854 字节）被下载与校验，但**没人用**。
* `pet/voice_input/mic.py` 的 docstring 已经写明「后者由识别子进程的 silero VAD 判定
  （`worker.py --vad`）」——**文档承诺了、接线没做**。

**为什么本次不顺手接上**：这一条会**改变用户可见行为**（放音乐时输出从"幻觉文本"变成"没听到说话"），
且本次实机样本已显示 VAD 闸门会**切掉句尾**——同一段 7.54 s 音频，不启 VAD 识别出
`今天天气很好，我们一起去公园散步吧。这是语音输入的测试。`（逐字正确），启 VAD 后变成
`今天天气很好，我们一起去公园散步吧。这是。`（**丢掉「语音输入的测试」**）。这是产品口径取舍
（剪掉句尾静音 vs 剪掉真话），应由产品拍板，不适合夹在本次乱码修复里静默改变行为。

## 五、实机运行记录

全部在本机真实环境执行（不是 CI、不是 mock）。产物：`dist-onedir\dsh-pet-standalone-webm-chat\`。

### 5.1 模型三件套：逐文件 sha256 与钉版表一致

```
$ python -c "…from pet.voice_input import models; models.model_problem(<数据目录>/voice_input/models)…"
problem: ''
model.int8.onnx -> ''
tokens.txt -> ''
silero_vad.onnx -> ''
elapsed 0.4s
```

（`''` = `verify_file` 通过：体积与 sha256 都对上钉版表。数据目录 =
`%APPDATA%\dsh-pet-standalone-webm-chat\voice_input\models`。）

### 5.2 麦克风：存在、默认、支持 16 kHz 单声道

```
$ python -c "…QMediaDevices.audioInputs()…"
inputs: 1
 - 麦克风 (HECATE G2耳机版) | default= True
  16000Hz ch1 int16 -> True
  44100Hz ch1 int16 -> True
  48000Hz ch1 int16 -> True
  preferred: 48000Hz ch1 Float   min 8000 - max 192000
```

**这一条是必须现场测的**：`mic.py::start` 会在 `device.isFormatSupported(16k/单声道/Int16)` 为假时
直接返回「麦克风不支持 16000 Hz 单声道 16 位采集」并**拒绝录音**。本机该设备返回 `True`，不触发。

### 5.3 采集链路：打开、出帧、断句判据都正常（4 s 环境音，无人说话）

```
$ python .scratch/voice-probe/probe_mic.py 4
state log     : ['recording', 'idle']
level samples : 249 peak_rms=0.0221 mean_rms=0.0004
captured PCM  : 0 bytes = 0.00 s
ASR           : no PCM captured (Endpointer never latched a speech start)
```

4 s 收到 **249** 个电平样本（16 ms 节拍 ≈ 250 个，符合 `QUEUE_READ_MS`）；
峰值 0.0221 瞬时越过 `start_rms=0.020` 但未持续 `start_ms=250`，**Endpointer 正确地没有起录**
（`captured PCM 0 bytes`）——即"咳嗽/咔哒不触发"这条设计在真机上成立。

### 5.4 故障复现（改动前的 exe，12:49:32）：GBK 字节让 UTF-8 解码失败

```
[frozen/novad] stdout = {"text": "…", "error": "", "seconds": 1.083, "speech": true}
[frozen/novad] host-parse = '{"text": "\ufffd\ufffd…\u0530…", …}'      ← 宿主按 UTF-8 解出替换字符
```

原始字节（hex，干净环境、无 `PYTHONIOENCODING`）：

```
7b 22 74 65 78 74 22 3a 20 22 bd f1 cc ec cc ec c6 f8 ba dc ba c3 a3 ac …
                        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ = 「今天天气」的 **GBK** 字节
utf-8 decode -> UnicodeDecodeError: 'utf-8' codec can't decode byte 0xbd in position 10
gbk   decode -> 正常
```

**阿拉伯数字正常**的机制同屏可见：JSON 骨架与路径（纯 ASCII）解析照旧，坏掉的只有非 ASCII 那一段。

### 5.5 修复后（exe 13:05:01）：四种「区域 × 形态」组合全部是 UTF-8

```
[frozen, clean env]        hex = 7b 22 74 65 78 74 22 3a 20 22 e4 bb 8a e5 a4 a9 e5 a4 a9 e6 b0 94 …
[frozen, PYTHONIOENCODING=utf-8]  同前
[source, clean env]               同前
[source, PYTHONIOENCODING=utf-8]  同前
utf-8 decode -> OK
gbk   decode -> UnicodeDecodeError: 'gbk' codec can't decode byte 0xac in position 36
```

`e4 bb 8a e5 a4 a9` = 「今天」的 **UTF-8** 字节。**与 5.4 正好互为倒数**：改动前 `utf-8` 失败、
`gbk` 成功；改动后 `utf-8` 成功、`gbk` 失败——这是"编码真的换了"而不是"看起来对了"的证据。

### 5.6 端到端识别文本（改动后，真子进程，真语音）

样本：Windows SAPI（`Microsoft Huihui Desktop`，zh-CN）合成、**16 kHz 单声道 16 位** WAV，
7.54 s / 241,280 字节 PCM；原文 =「今天天气很好，我们一起去公园散步吧。这是语音输入的测试。」

```
exe mtime      : 2026-10-16 13:05:01
[frozen/novad] parsed = {"text": "今天天气很好，我们一起去公园散步吧。这是语音输入的测试。", "error": "", "seconds": 1.02, "speech": true}
[engine/source] wall=2.14s -> text='今天天气很好，我们一起去公园散步吧。这是语音输入的测试。' ok=True
[engine/frozen] wall=2.31s -> text='今天天气很好，我们一起去公园散步吧。这是语音输入的测试。' ok=True
[frozen/vad-silence] parsed = {"text": "", "error": "", "seconds": 0.0, "speech": false}
[frozen/missing-model] parsed = {"text": "", "error": "语音模型尚未下载（目录还不存在：…）", "seconds": 0.0, "speech": true}
```

* 冻结包与源码态**逐字一致**，且与 TTS 原文逐字一致；
* 静音段经 VAD → `speech: false`（**不是**幻觉文本，也不是错误）；
* 模型缺失 → 退出码 2 + 中文人话（**这条错误信息本身以前也会乱码**，一并修好）。

### 5.7 构建与产物

```
[1/3] PyInstaller --onedir building dsh-pet-standalone-webm-chat … Build complete!
[slim] removed 129 files, freed 83.01 MB
[encoding-check] PASS: 中文编码检查全部通过（字面量/资源/文件名原样保持 UTF-8）
[smoke] exe window appeared after 1.0s / --settings window appeared after 1.3s
BUILD_EXIT=0
```

**打包面逐项核对**（语音输入要用到的东西有没有被 slim 掉）：

```
PySide6/QtMultimedia.pyd        PySide6/Qt6Multimedia.dll
PySide6/plugins/multimedia/ffmpegmediaplugin.dll
PySide6/plugins/multimedia/windowsmediaplugin.dll
PySide6/avcodec-63.dll  avformat-63.dll  avutil-61.dll  swresample-7.dll
_internal/sherpa_onnx/          ← 识别引擎在
```

进程与完整性级别（沿用本档既有流程）：

```
icacls <dist> /setintegritylevel Medium /T /C /Q
  -> Successfully processed 26956 files; Failed processing 0 files
PID 4188  dsh-pet-standalone-webm-chat.exe   WorkingSet 153,264,128
```

### 5.8 测试与静态检查

```
$ python -m ruff check pet tests tools --no-cache
All checks passed!

$ python -m pytest tests/test_voice_input_*.py tests/test_chat_open_voice_controller.py \
      tests/test_pr_report_discipline.py -q
212 passed in 9.80s

$ python -m pytest -q          # 全量
10 failed, 7572 passed, 14 skipped, 28 warnings in 849.13s (0:14:09)
```

`test_pr_report_discipline.py` 的三节标题与本报告在 `INDEX.md` 的登记由该测试机器化校验。

**全量那 10 条红与本次改动无关，且已用 A/B 实测证明（不是推断）**。证据链三条：

1. **没有任何一个失败文件引用本模块**：对 7 个失败测试文件逐个 `grep -c "voice_input"` → **全部 0**。
   本改动只落在 `pet/voice_input/worker.py` 的**子进程 stdout 输出通道**上。
2. **A/B 对照**：把 `_write_stdout` **临时改回**改动前的 `sys.stdout` 写法，重跑那 10 条 ——
   `5 failed, 56 passed`，**失败集合与全量运行时完全一致**（`test_modern_settings_dialog_py_line_budget`、
   `test_library_shared_media` ×2、`test_settings_copy_lint`、`test_settings_page_guide`）。
   即：**没有本次修复，这些照样红**。（另 5 条在隔离运行时转绿 —— 它们是全量运行里的
   测试顺序/状态污染型脆弱，与本报改动同样无关；对照组跑完已把 `worker.py` 逐字还原，
   还原后 211 行、`os.write(1, view)` 命中 1 处、语音族 212 项全过。）
3. **其中一条本来就是已登记的既有红**：`test_discard_unverified_notice_is_a_real_visible_box`
   在 [`docs/INDEX.md`](INDEX.md) 中已被
   [`PR-REPORT-SETTINGS-ROWS-TOPWINDOW-2026-10-13.md`](PR-REPORT-SETTINGS-ROWS-TOPWINDOW-2026-10-13.md)
   登记为「在本分支上必红」，并给出「基线代码上只加 30ms sleep 也能复现同一条红」的对照。

**如实登记一条旁证**：全量运行的 pytest 自身在 `_readerthread` 里抛了
`UnicodeDecodeError: 'gbk' codec can't decode byte …` —— 那是 **pytest 读子进程输出时**
踩的同一类区域编码问题（不是本模块的代码路径），但它正面说明：这台中文 Windows 上
「子进程字节 → 宿主按固定编码解」这条缝**到处都是**，本报告的「先例」一节因此值得被后来者读到。

### 5.9 **未验**（如实分栏，不用沉默代替结论）

* **真人语音样本未采集**：5.6 用的是 SAPI 合成语音（真 16 kHz PCM、真模型、真子进程），不是真人对着
  HECATE G2 耳机麦说话。真人的口音/音量/环境噪声未覆盖；一个真人真麦的 GUI 用例已另行通过（见 5.10）。
* **模型首次下载路径未跑**：本机模型三件套已就位（5.1），本次没有走「第一次按麦克风 → 下载 228 MB」
  那条分支。
* **`--vad` 闸门仍未接线**（见 §四），本次刻意不改。

### 5.10 人工 GUI 验收（已通过）

改动后重启的冻结包（`PID 4188`，overlay 拓扑，日志 `pet-4188.log`）上，由**用户本人**在聊天窗
点击麦克风按钮并说中文：

> 结果：**汉字正确显示**

这条把 5.9 里原本「需要人来按并说话」的那一段补齐了 —— 即
「按钮按下 → `MicRecorder` 采集 → 子进程识别 → 文本框呈现」的**完整链路**在实机通过，
与 5.6 的脚本证据（除按钮与采集之外的同一条链）互为补充。**修复前**用户报告的是同一动作
「阿拉伯数字正常、汉字乱码」，**修复后**同一动作得到正确汉字。
