# 第 2 章实验总结

## 我对第 2 章的理解

第 2 章的标题是「上下文工程」，但读完才明白它的主张有多强：**模型本身的智力只是基础，上下文的质量才是 Agent 能力的真正关键**——书里甚至说，一个中等能力的模型配上精心组织的上下文，往往能胜过顶级模型在信息匮乏下的盲目摸索。

这把第 1 章的公式 `Agent = LLM + 上下文 + 工具` 里的第二项单独拎出来放大了。第 1 章讲三者缺一不可，第 2 章讲的是：**当各家模型能力逐渐拉平时，胜负手转移到了上下文**。这也解释了为什么本章会花这么多篇幅在看似琐碎的事情上——系统提示词怎么写、工具定义放哪、时间戳该不该放进前缀。

本章小结用一句话串起了全章：「API 消息结构定义骨架；稳定前缀提高 KV Cache 命中；Prompt、Skills 和状态栏分别承载规则、按需知识与当前状态；压缩则在保留决策、约束、失败和来源的前提下，提高历史信息密度。」我把它拆成两层来记：

- **放什么**（内容层）：提示词承载规则、Skills 承载按需知识、状态栏承载当前状态、压缩提高历史密度。
- **怎么放**（物理层）：**稳定的放前面、动态的放后面**——因为 KV Cache 只认前缀。

第二层是本章最反直觉、也最容易被忽视的部分。它意味着**上下文的排列顺序不只由语义逻辑决定，还由缓存的经济性决定**。书里举的 Claude Code 例子很说明问题：系统提示词被一个缓存边界劈成两半，边界之前全局复用，之后放用户/会话特定信息；每个运行时条件若放在边界之前，缓存键变体数就翻一倍（N 个二值条件 → 2^N 种组合）。所以「提示词的结构由缓存边界决定」，语义逻辑得往后排。这是个真正的**架构约束**而不是事后优化，书里那句「越早将这个约束纳入架构设计，后续的工程代价越小」是本章最实用的一条结论。

而 2-1 在全章的位置很特别：它排在 KV Cache 那一节**之前**，书里的说法是「在深入理解 Agent 上下文之前，让我们先通过一个实际项目来体验小型模型的能力」。所以它是个**入口实验**，主要目的不是测性能，而是让你亲眼看到 0.6B 的超小模型也能跑通工具调用闭环，从而接受书稿的核心论断：**模型大小不是唯一的决定因素，端侧 Agent 的时代比大多数人预期的更近**。

我做完这个实验，最大的收获反而是三个「**论断与实测的对照**」——它们既验证了书稿，也精确划出了书稿论断的边界：

1. **「0.6B 也能可靠完成工具调用」——成立，但有边界，而边界恰好被架构补上了。** 详见 3.1。
2. **「M2 上 >100 tok/s」——我测到 84.17，未复现；但差距不在硬件上，在软件栈上。** 详见 3.3 与 3.5。
3. **「稳定前缀提高 KV Cache 命中」——34.28× 给了它一个具体数量级；而命中臂的退化曲线，恰好是思考题 9 的实证答案。** 详见 3.2。

带着这三条对照去读后面的小节，会比单看数据表格有用得多。

---

> 🎯 **范围约定**（延续第 1 章）：只用国内可直连的资源，不申请境外 API key。
> 本章 2-1 进一步采用**外部部署 LLM** 方案——模型服务跑在局域网另一台机器上，
> 本机只做客户端。因此本机**不需要 GPU、不需要装 Ollama/vLLM**。

本目录是第 2 章实验项目的**运行产出归档**（代码在原仓库 `ai-agent-book/chapter2/`，这里只保留结果）。

---

## 一、总览

| 实验 | 目录 | 方案 | 状态 | 主要产出 |
|---|---|---|---|---|
| **2-1** ★ 本地 LLM 服务部署与工具调用 | `local_llm_serving/` | 局域网 LM Studio（`192.168.1.2:8234`）<br>+ `qwen3-0.6b` | ✅ 四场景全跑完<br>⚠️ 官方验收脚本不可用 | `result.log`、`benchmark_all.json`、`benchmark_all.log` |
| 2-2 ~ 2-10 | — | — | 未开始 | — |

第 2 章共 10 个实验，本目录当前只归档 2-1。

### 关键数据速查

一张表看完本次所有核心结果，以及与书稿论断的对照：

| 指标 | 实测值 | 书稿参照 | 判定 |
|---|---|---|---|
| 工具调用闭环（gate 3/4/5） | ✅ 全过 | 「0.6B 也能可靠完成工具调用」（`chapter2.md:430`） | ✅ **论断成立** |
| `required_location` 填参 | ❌ 只填 `'Vancouver'` | 同上，但前提是「合理的提示词设计**和系统架构**下」 | ⚠️ **这就是边界**：被工具层 geocoding 补全兜住 |
| 单流 decode 吞吐 | **84.17 tok/s** | 苹果 **M2 >100 tok/s**（`chapter2.md:411`） | ❌ **未复现**（达参照值 84 %）；硬件已识别，差距在软件栈——见 3.5 |
| 显存带宽利用率 | **7.5 %**（0.6B）/ 7.3 %（9B） | RX 5700：448 GB/s | ⚠️ **硬件远未用满**，84 tok/s 不是这块卡的极限 |
| KV Cache 加速比 | **34.28×**（0.285 s vs 9.765 s） | 「稳定前缀提高 KV Cache 命中」（`chapter2.md:1089`） | ✅ **量化印证** |
| 命中臂稳定性 | 首轮 0.116 s → 后四轮 0.29–0.35 s | 思考题 9「动态信息变化破坏前缀命中」 | ✅ **实证答案** |
| 批处理拐点 | 并发 **4**（148.5 tok/s 聚合） | — | 🔑 并发 8 反降至 146.8，TTFT 恶化 ×37 |
| 官方验收（9 条 gates） | 5 条 ✅、3 条 ⚠️、1 条 ❌ | 需 Ollama native `/api/generate` | ❌ **结构性不可达**，非“没跑完” |

> ⚠️ **最重要的一条限制先说在前面**：官方验收脚本 `run_experiment.py` **刻意绑死 Ollama native 接口**（靠 model digest 判定验收），
> 而本方案走的是 LM Studio 的 OpenAI 兼容接口，因此**不能声称「通过了官方验收」**。
> 学习目标能达成，验收结论不能下——具体见第四节。

---

## 二、2-1 运行配置（复现前必读）

本机无 GPU、未装 Ollama，官方三条路径（vLLM / Ollama / `--backend auto`）都跑不通，
因此模型服务放在局域网另一台机器上，本机只做客户端。服务端硬件与软件栈的分析见 3.5。

**服务端**：`192.168.1.2:8234`，LM Studio，加载 `qwen3-0.6b`——与 protocol 的 `reference_model: Qwen/Qwen3-0.6B`
是同一模型（只是 id 风格不同），因此性能数据与官方口径**可比**。

### 2.1 必须改的三处（否则跑不起来）

| # | 位置 | 原值 | 改为 | 原因 |
|---|---|---|---|---|
| 1 | `agent.py:13` | `from config import OPENAI_API_BASE, OPENAI_API_KEY, LOG_LEVEL` | 追加 `, MODEL_NAME` | 下一行才能引用 |
| 2 | `agent.py:248`（`chat()`）<br>`agent.py:379`（`chat_stream()`） | `model="Qwen/Qwen3-0.6B"` | `model=MODEL_NAME` | **硬编码**，完全不读 `.env` |
| 3 | `.env` | `MODEL_NAME=Qwen/Qwen3.5-9B`<br>`VLLM_PORT=1234` | `MODEL_NAME=qwen3-0.6b`<br>`VLLM_PORT=8234` | 按服务端实际 id 与端口填 |

改动 1、2 让 `.env` 真正成为单一配置源——这本来就是 `MODEL_NAME` 的设计意图，只是 `agent.py` 漏了导入。改完后 `config.py` 生效值：

```
MODEL_NAME      = qwen3-0.6b
OPENAI_API_BASE = http://192.168.1.2:8234/v1     ← config.py:31 强制拼 http://，无法用 https
OPENAI_API_KEY  = EMPTY                          ← config.py:32 硬编码，不读环境变量
```

这两处硬编码在局域网场景下无害，但也意味着这套代码**无法直接指向需要真实 key 的 HTTPS 云端 API**。

> ⚠️ **模型名陷阱**：LM Studio 能容忍 `厂商/` 前缀差异（会做规范化），但**不能容忍大小写与版本号差异**。
> 最险的是失败形态——id 不匹配时**不返回 404，而是静默挂起**（实测 60 秒超时、0 字节返回），只表现为“卡住”。
> 排查这类问题应先 `curl /v1/models` 逐字比对 id，而不是先怀疑网络或代码。
>
> | model 值 | 结果 |
> |---|---|
> | `Qwen/Qwen3.5-9B`（服务端是 `qwen/qwen3.5-9b`，差大小写） | ❌ 挂起，60 秒超时，0 字节 |
> | `qwen/qwen3-0.6b` 或 `qwen3-0.6b` | ✅ 都秒回（前缀可省） |

> 📌 另外两个会绊人的细节：
> - **必须显式 `--backend vllm`**。`--backend auto` 在无 GPU 的 Linux 上会选 ollama（`main.py:63`），随后因未安装而 `sys.exit(1)`——它**不会**去连外部服务。
> - `main.py:83` 的健康检查探 `/health`，而 LM Studio 对未知路径一律返回 **HTTP 200**（body 是 error JSON），恰好跳过“本机启动 vLLM”的分支。换成别的推理框架未必有这个行为，届时需改健康检查。

### 2.2 依赖精简：只需 4 个包

`requirements.txt` 要求 `vllm`/`torch`/`transformers`/`accelerate`/`fastapi`/`uvicorn`/`ollama`/`python-weather`/`PyPDF2`，
但本机不做推理，**实际只需要 4 个**：

```bash
uv pip install --python .venv/bin/python openai python-dotenv requests PyPDF2
# 实测装入：openai==3.14.1  python-dotenv==1.2.3  requests==2.34.2  pypdf2==3.0.1
```

其中 `PyPDF2` **不能省**——`tools.py:13` 在模块顶层 import，缺了整个工具注册失败。
其余重型包在外部部署下全用不上（天气工具实际走 Open-Meteo 的 HTTP 接口，不需要 `python-weather`）。

> 📌 本仓库的 `.venv` 当前是**空壳（0 个包，Python 3.12.14）**，上面 4 个包是本次为 2-1 单独装入的。

---

## 三、实测结果

### 3.1 工具调用（protocol 的 `tool_case`）——门控全过，但填参有一处不达标

```bash
python main.py --backend vllm --mode single \
  --task "What are the current time and weather in Vancouver? Call both tools."
```

`result.log`（2026-09-19 23:12 运行，总耗时约 **10 秒**）：

| protocol 门控 | 要求 | 实测 | 判定 |
|---|---|---|---|
| gate 3 | 首次响应恰好含两个要求的工具调用 | `get_current_time` + `get_current_temperature` | ✅ |
| gate 4 | 两工具并发执行且结果可审计 | 均返回结构化 JSON，带时间戳、坐标、`source: Open-Meteo` | ✅ |
| gate 5 | 第二轮消费工具结果并终止，不再调工具 | ReAct 迭代 2 后直接输出答案，无第三次调用 | ✅ |
| `required_timezone` | `America/Vancouver` | `{'timezone': 'America/Vancouver'}` → 08:12:10 PDT / UTC-0700 | ✅ |
| `required_location` | `Vancouver, Canada` | **`{'location': 'Vancouver'}`** | ❌ **只填了城市名** |

时间线：`23:12:02` 迭代 1 → `23:12:04` 首个 HTTP 200（**2 秒**）→ `23:12:12` 迭代 2（中间 8 秒含两跳 Open-Meteo 与流式生成）。

返回数据：温哥华 2026-09-19 08:12:10 PDT、**10.9 °C**、overcast、湿度 95 %、风速 4.8 km/h、坐标 49.24966 / −123.11934。

> ⚠️ **`required_location` 这一项 0.6B 没达标**，模型只给了 `'Vancouver'` 而不是 `'Vancouver, Canada'`。
>
> 但工具**没有失败**：`tools.py:145-167` 的第一跳 geocoding 用 `"Vancouver"` 照样查到了坐标，
> 并在第 167 行用返回的 `name` + `country` 重拼成 `"Vancouver, Canada"`，最终结果完全正确。
>
> 这是小模型的真实代价。有价值的对比是：**上一轮 9B 用同一份工具 schema 填的是 `'Vancouver, Canada'`（达标）**，
> 换成 0.6B 后同样的 schema 就填不全了——变量只有模型规模，所以这是**指令遵循能力差异**，不是 schema 写得不好。
> （此前用 curl + 极简 schema 裸测 9B 时，它也退回只给 `Vancouver`。）
> 两条证据合起来：**schema 描述质量与模型规模都会影响填参准确度，而 0.6B 即使用完整 schema 也补不齐。**
>
> 从第 1 章的角度看，这恰好是「工具要能容忍不完美的模型输入」的实证：
> geocoding 那一跳本是为人话地名设计的，却顺手兜住了小模型的输出缺陷。

> 🔑 **与书稿论断的对照**：`chapter2.md:430` 的实验总结说「0.6B 的小模型在合理的提示词设计下也能**可靠地**完成工具调用」。
> 本次实测的判定是：**论断成立，但必须加上书稿自己在 `:406` 写的那个前提——「和系统架构下」**。
>
> - **成立的部分**：选工具、并发执行、消费结果后终止，三个 gate 全过，10 秒跑完闭环。这确实「可靠」。
> - **边界的部分**：填参精度不如 9B（`location` 填不全）。
> - **边界为何没变成失败**：靠的正是**架构**——`tools.py` 的 geocoding 补全。
>
> 换句话说：**书稿那句「在合理的提示词设计和系统架构下」不是客套的限定语，而是论断成立的必要条件。**
> 0.6B 的能力缺口是真实存在的，只是被工具层的容错设计吸收掉了。
>
> 这对端侧 Agent 的启示很直接：**模型越小，越要把可靠性做在模型外面**——
> 用工具的容错、schema 的描述质量、必要时的重试与校验去补，而不是指望小模型自己填对每一个参数。
> 这也正好呼应第 1 章的 Harness 工程：模型是那匹马，缰绳在模型之外。

### 3.2 KV Cache（protocol 的 `cache_case`）——✅ 完整 5 轮

```bash
python benchmark.py --scenario all --backend vllm \
  --base-url http://192.168.1.2:8234/v1 --model qwen3-0.6b \
  --prefix-tokens 4096 --repeats 5 --temperature 0 --max-tokens 512 \
  --output benchmark_all.json
```

`benchmark.py` 与 `run_experiment.py` 不同，它**走 OpenAI 兼容接口**并支持 `--base-url` / `--model` / `--api-key` 覆盖，
所以外部部署可用。默认值有三处不符 protocol，必须显式传：`--temperature`（默认 0.7 → 要 0）、
`--prefix-tokens`（默认 1024 → 要 4096）、`--max-tokens`（默认 256 → 要 512）。

实测 TTFT（`benchmark_all.json`，`--scenario all` 全程 **163 秒**，`exit_code=0`）：

| 轮次 | 命中 TTFT | 未命中 TTFT | 倍数 |
|---|---|---|---|
| 1/5 | **0.116 s** | 9.766 s | 84× |
| 2/5 | 0.338 s | 9.799 s | 29× |
| 3/5 | 0.328 s | 9.746 s | 30× |
| 4/5 | 0.290 s | 9.736 s | 34× |
| 5/5 | 0.351 s | 9.776 s | 28× |
| **均值** | **0.285 s** | **9.765 s** | **34.28×** |
| 标准差 | 0.097 s | **0.025 s** | — |
| 变异系数 | 34 % | **0.25 %** | — |

**两个值得停下来看的读数**：

**① 未命中臂极其稳定**——标准差 0.025 s，变异系数仅 0.25 %。
prefill 是确定性的矩阵运算，本身不含随机性，所以这个稳定度反证了**局域网链路与测量方法都可信**。
反过来说：如果哪天 TTFT 抖动变大，应该先怀疑网络或服务端负载，而不是缓存机制本身。

**② 命中臂第一轮明显更快**（0.116 s，后四轮稳定在 0.29–0.35 s）。
原因藏在 `benchmark.py:196-206` 的循环结构里：每轮是「hit → miss」交替，
而 miss 请求每次都在 system prompt **开头**插入 unique 串（`[req-{i}-{time.time_ns()}]`），
等于每轮往缓存里塞一个全新的 4096-token 前缀。
第 1 轮 hit 之前只有预热请求（同一个前缀），缓存最干净；从第 2 轮起，每次 hit 之前都刚被一个 unique miss 冲刷过。

> 🔑 所以 **hit TTFT 不是常量，而取决于服务端缓存的驻留与淘汰情况**——
> 这正是 protocol 要求 `matched_repeats: 5` 而不是只测一次的原因：单次测量会把 0.116 s 这种
> 「缓存最干净」的极端值当成典型值，从而**高估**加速比（用 0.116 算会得到 84×，而 5 轮均值是 34×）。

> 📖 **这正是本章思考题 9 的实证答案**。思考题 9 问：「动态信息（如系统时间戳、工具列表顺序）的变化会破坏 KV Cache 前缀命中……你会如何设计上下文布局来最大化缓存命中率？」
>
> 我的数据给出了一个量化版本：**每轮往上下文里插入一个 unique 前缀（哪怕只有十几个 token），
> 就足以把下一轮的命中 TTFT 从 0.116 s 推到 0.33 s 左右——慢了约 3 倍。**
> 注意插入的只有 `[req-{i}-{ns}] ` 这二十几个字符（约十几个 token），占 4096 token 前缀的不到 0.5 %，
> 但因为它在**最开头**，代价却是全局的。
>
> 这精确对应书稿 `chapter2.md:538` 的原理：「代价取决于改动位置：**变动点越靠前，需要重新计算和计费的 token 越多**」。
> `benchmark.py:203` 把 unique 串插在 system prompt 开头，等于故意制造「最坏情况」：改开头 = 全部 4096 token 重算，
> 所以 miss TTFT 高达 9.77 s。这也解释了书稿为何反复强调「系统提示词一旦定下来就不要改」——
> **改动的位置比改动的字数重要得多**。

推算 prefill 吞吐：4096 tok ÷ 9.765 s ≈ **419 tok/s**。

> ⚠️ **34.28× 这个加速比带有硬件烙印，不要当成 KV Cache 的普适收益。**
> 加速比 = miss ÷ hit，分子几乎全是 prefill 时间。而 419 tok/s 的 prefill 只用到 RX 5700 理论算力的个位数百分比（见 3.5）——
> **软件栈越慢，prefill 越贵，KV Cache 的相对收益看起来就越大。**
> 换到 prefill 更快的环境（Apple M2 走 Metal，或把本机 offload 调对），miss TTFT 会大幅下降，加速比也会随之收窄。
> 真正跨硬件稳定的是**绝对收益**：省下 9.5 秒的首 token 等待。

> 📌 **历史对照（上一轮 9B，数据已被本节取代，仅留作对比与教训）**
>
> 用 `qwen/qwen3.5-9b` 跑同一场景：命中 0.450 s / 未命中 52.100 s / **~116×**，
> 推算 prefill ~80 tok/s、decode ~6.4 tok/s。**但那次只完成 3/5 轮**——
> 我给命令设了 `timeout 850`，而 9B 每轮实际要 3.4–5.7 分钟，进程在第 4 轮途中被杀，
> `--output` 的 JSON 没写出；更糟的是我那句 `echo "exit=$?"` 取的是**前一条空 `echo`** 的退出码，
> 显示 `exit=0`，掩盖了被 timeout 杀掉的事实。
>
> 两条教训：**timeout 要按「单轮耗时 × 轮数 × 2」估**；**`$?` 必须紧接 `timeout`，中间不能插任何命令**。
> 换成 0.6B 后全程只要 163 秒，`timeout 1800` 绰绰有余。原始日志留在 `benchmark_kv_cache.log`。

### 3.3 吞吐与并发

**单流吞吐（`throughput`，5 次）**：

| 次 | TTFT | 总耗时 | 输出 | decode |
|---|---|---|---|---|
| 1 | **0.568 s** | 6.77 s | 512 tok | 82.6 tok/s |
| 2 | 0.135 s | 6.53 s | 512 tok | 80.0 tok/s |
| 3 | 0.074 s | 6.03 s | 512 tok | 86.0 tok/s |
| 4 | 0.071 s | 6.00 s | 512 tok | 86.4 tok/s |
| 5 | 0.068 s | 6.03 s | 512 tok | 85.9 tok/s |
| **均值** | **0.183 s** | 6.27 s | 512 tok | **84.17 tok/s** |

decode 标准差 2.77 tok/s（变异系数 3.3 %）。第 1 次 TTFT 0.568 s 明显高于后四次（0.07–0.14 s），
是 `--scenario all` 的**首个请求**，含连接建立与冷启动开销。

**并发（`batching`，四个层级）**：

| 并发 | 墙钟 | 总 tokens | 聚合 tps | 单请求 tps | 平均 TTFT |
|---|---|---|---|---|---|
| 1 | 6.41 s | 512 | 79.9 | 79.9 | 0.296 s |
| 2 | 7.66 s | 1024 | 133.6 | 66.8 | 0.149 s |
| 4 | 13.64 s | 2025 | **148.5** | 37.1 | 2.937 s |
| 8 | 27.37 s | 4019 | 146.8 | 18.4 | **11.016 s** |

这是一条**吞吐-延迟权衡**的教科书式曲线：

- **聚合吞吐 79.9 → 148.5 tok/s（1.86×），但拐点在并发 4**；并发 8 反而略降到 146.8。
  说明服务端的批处理能力在并发 4 附近已**饱和**，再加并发只是排队，不再换来更多吞吐。
- **单请求体验持续恶化**：单请求 tps 79.9 → 18.4（÷4.3），TTFT 0.296 → 11.016 s（**×37**）。
  并发 8 时用户要等 11 秒才看到第一个字——聚合吞吐漂亮，体验却是灾难。
- 并发 2 的 TTFT（0.149 s）比并发 1（0.296 s）**还低**，因为并发 1 那次含 batching 场景的首请求开销，不是稳态值。看趋势要从并发 2 起算。
- 并发 4/8 的总 tokens 是 2025/4019 而非 2048/4096，说明有请求提前 EOS 结束（`max-tokens 512` 是上限而非定值）。

> 🔑 **按 `claim_policy.throughput` 严格判定：本次仍不能声称复现手稿的 M2 >100 tok/s。**
>
> protocol 原文：「report measured decode throughput; do not claim the manuscript's M2 >100 tok/s observation
> **unless this run exceeds it on identified hardware**」。这个指标的出处是 `chapter2.md:411`——
> 「在本书作者所用的**苹果 M2 芯片**上，模型能够以超过每秒 100 个 token 的速度生成响应」，
> 是作者在一台具体机器上的实测值，所以 protocol 才要求对照方也报出自己的硬件。
>
> 本次两个条件的状态：
>
> 1. **口径不符（不满足）**：单流 decode 实测 **84.17 tok/s < 100**（达参照值 84 %）。并发 4 的聚合 148.5 tok/s 虽超 100，
>    但那是**聚合**吞吐，不是 claim_policy 说的「decode throughput」，两者不能混用。
> 2. **硬件已识别（满足）**：Windows 10 + AMD Radeon RX 5700，见 3.5。
>
> **所以结论是明确的「未复现」，而不是「无法判断」**——而且 3.5 会说明，这个差距不在硬件上
> （RX 5700 的显存带宽是 M2 的 4.5 倍），而在软件栈上。
>
> **能诚实声称的是**：在与 protocol 同一个模型（Qwen3-0.6B）上，实测单流解码 84.2 tok/s、
> 前缀缓存把 TTFT 从 9.77 s 压到 0.29 s（34×）、批处理在并发 4 出现 148.5 tok/s 的聚合吞吐拐点。

### 3.4 同一台服务端：9B vs 0.6B

两次实验打的是同一台 `192.168.1.2`、同一个 LM Studio，只换了模型：

| 指标 | `qwen3.5-9b`（端口 1234） | `qwen3-0.6b`（端口 8234） | 变化 |
|---|---|---|---|
| tool_case 总耗时 | 35 s | ~10 s | 3.5× 更快 |
| KV 命中 TTFT | 0.450 s（3 轮） | 0.285 s（5 轮） | 1.6× 更快 |
| KV 未命中 TTFT | 52.100 s | 9.765 s | 5.3× 更快 |
| **KV 加速比** | **~116×** | **34.28×** | **反而缩小 3.4 倍** |
| 推算 prefill | ~80 tok/s | ~419 tok/s | 5.2× 更快 |
| decode | ~6.4 tok/s | 84.17 tok/s | 13× 更快 |

最反直觉的是**加速比从 116× 掉到 34×**——明明 KV Cache「变快了」。

因为加速比 = miss ÷ hit，是个**比值**，分子分母的缩小幅度不同：

- 9B 时 prefill 是绝对瓶颈（52 s），跳过它收益巨大；hit 的 0.450 s 里固定开销占比很小。
- 0.6B 时 prefill 只要 9.77 s，而 hit 的 0.285 s 里**固定开销（HTTP 往返、调度、首 token 采样）占比大幅上升**。
- 分子缩小 5.3 倍，分母只缩小 1.6 倍 → 比值必然掉下来。

> 🔑 **KV Cache 的相对收益随模型变小而递减，但绝对收益依然显著**（9.77 s → 0.29 s，实打实省 9.5 秒）。
>
> 推论：**模型越小、prefill 越不是瓶颈，缓存优化就越不值得优先做**；
> 反过来在大模型或长上下文场景（4096+ token 前缀）上，前缀缓存几乎是必选项。
> 这也解释了 protocol 为什么特意规定 `approximate_prefix_tokens: 4096`——
> **前缀太短的话 prefill 本身就不贵，根本测不出缓存的价值**。

至于 decode 的 13× 差距：9B 参数量是 0.6B 的 15 倍，若 decode 受内存带宽约束，速度比就该接近 15 倍。实测 13×，量级吻合。
但光看这个比例还不够——**把两个模型的带宽利用率分别算出来后，结论会硬很多**，见下一节。

### 3.5 硬件与软件栈：84 tok/s 不是这块卡的极限

服务端环境（已确认）：**Windows 10 + AMD Radeon RX 5700**。

| 项 | RX 5700 规格 |
|---|---|
| 架构 | RDNA1（Navi 10），7 nm，2019 年 |
| 显存 | 8 GB GDDR6，256-bit |
| **显存带宽** | **448 GB/s** |
| 算力 | FP32 7.94 TFLOPs / FP16 15.9 TFLOPs |
| 可用计算 API | **只有 Vulkan**（无 ROCm，见下） |

**这块卡跑本地推理只能走 Vulkan，没有第二条路**：

- ROCm 的 PyTorch 轮子**不覆盖 RDNA1**——RX 5500/5600/**5700** 这一代被明确排除，torch 在这些卡上只能用 CPU，
  GPU 推理必须改走 Vulkan（unsloth 安装文档的原话：`GGUF chat runs on the GPU through Vulkan`）。
- Windows 上本来也没有完整的 ROCm 栈；LM Studio 在 Windows + AMD 下的默认路径就是
  Settings → Runtime → GGUF Acceleration → **Vulkan**。
- 量级参照（社区实测）：Qwen2.5-14B Q4 在 Windows + Vulkan 下约 16 tok/s，换到 Linux + ROCm 能到 24+ tok/s。

#### 把实测值与理论上限对一下

decode 阶段每生成一个 token 都要把模型权重完整读一遍，所以速度上限约等于 `显存带宽 ÷ 权重大小`。按 Q4 量化估算：

| 模型 | 权重（约） | 理论上限 | 实测 | **带宽利用率** |
|---|---|---|---|---|
| `qwen3-0.6b` | 0.4 GB | 448 / 0.4 ≈ **1120 tok/s** | 84.17 tok/s | **7.5 %** |
| `qwen3.5-9b` | 5.1 GB | 448 / 5.1 ≈ **88 tok/s** | ~6.4 tok/s | **7.3 %** |

**两个模型的利用率几乎完全一致（7.5 % vs 7.3 %）**。这个一致性比「速度差 13× 接近参数量差 15×」更有说服力：
它说明两个模型走的是**同一条推理路径、同一个系统性瓶颈**，而不是「0.6B 用 GPU、9B 用 CPU」。

> ⚠️ 上表是**量级估算**，依赖两个我没能确认的假设：量化格式（按 Q4_K_M 取 ≈0.5 byte/参数）
> 与 GPU offload 层数（按全部层都在 GPU 上算）。若实际是 FP16，0.6B 权重 1.2 GB，
> 理论上限降到 373 tok/s，利用率变成 22.5 %——仍然偏低，但没那么离谱。

#### 7.5 % 的利用率意味着什么

**这块卡的硬件能力远没有被用满**。最可能的三个原因，按排查成本排序：

| # | 检查项 | 在哪看 | 为何会拖慢 |
|---|---|---|---|
| 1 | **GPU Offload Layers 是否拉满** | 加载模型时的 advanced settings | 没拉满就有部分层在 CPU 上跑，每步都要跳 PCIe 搬数据——**最常见的原因** |
| 2 | **Runtime 是不是 Vulkan** | Settings → Runtime → GGUF Acceleration | 若落在 CPU 后端，448 GB/s 的显存带宽完全用不上 |
| 3 | 量化格式与上下文长度 | 模型文件名（Q4_K_M / Q8_0 / FP16） | FP16 会把权重放大 4 倍，直接压低理论上限 |

> 🔑 **这改变了「M2 >100 tok/s」那条判定的性质。**
>
> 之前缺硬件信息，我只能说「无法对照」；现在硬件明确了，结论变成：
> **RX 5700 的显存带宽是 Apple M2（约 100 GB/s）的 4.5 倍，实测速度却只有 M2 参照值的 84 %。**
> 硬件更强反而更慢，说明差距**不在硅片上，在软件栈上**：Apple Silicon 走 Metal（统一内存、无 PCIe 搬运），
> 而这台机器走 Vulkan（且很可能还有 CPU 混合推理）。
>
> 所以按 `claim_policy` 仍然**不能声称复现 M2 >100 tok/s**（84.17 < 100），
> 但现在可以补一句更有价值的话：**这个差距不是硬件造成的**。按理论上限算，
> 只要把 offload 与后端配置调对，这块卡有充分余量超过 100 tok/s——理论值是它的 13 倍。
> 这一条已记入 6.1 作为待验证项。

### 3.6 为什么会访问 `api.open-meteo.com`

`get_current_temperature` 需要真实天气数据，本地算不出来。`tools.py:138-184` 是**两跳**请求：

| 跳 | 端点 | 作用 | 超时 |
|---|---|---|---|
| 1 | `geocoding-api.open-meteo.com/v1/search` | 地名 → 坐标 | 5 s |
| 2 | `api.open-meteo.com/v1/forecast` | 坐标 → 当前天气 | 5 s |

选它的原因写在 `tools.py:141` 的注释里：`No API key required`——免费、免注册、免凭据。
而 protocol 的 `acceptance_gates` 正好有两条对应：

- `"all execution is local except the read-only Open-Meteo weather lookup"` ← **唯一被批准的外部调用**
- `"no credential is sent to or retained by the experiment"` ← Open-Meteo 无凭据，不违反

对比之下 `get_current_time` 是纯本地计算（`datetime` + 时区换算），不出网。
所以整体网络行为是：**模型推理走局域网（不出外网）+ 天气查询走 Open-Meteo（唯一外网调用）**。

> ⚠️ 两跳都只给 5 秒超时。国内访问 Open-Meteo 若变慢，工具会静默返回
> `{"error": "..."}` 而非崩溃——**模型仍会拿着错误结果继续作答**，这是第 1 章提过的「静默降级」形态。
> 复现时若天气数据异常，先单独 curl 这两个端点确认连通性。

---

## 四、⚠️ 最重要的限制：官方验收脚本无法用于本方案

`run_experiment.py` 是 2-1 的官方验收 runner，它在文件开头（第 4-5 行）就写明：

> Unlike an OpenAI-compatible client, this runner **deliberately** uses Ollama's `/api/generate` endpoint with `raw=true`.

它只实现了 Ollama 协议（`:60` 的 `OllamaRawClient`、`:106` 拼 `/api/generate`、`:375` 硬编码 `"provider": "local Ollama"`），
且**靠 Ollama 返回的 model digest 判定验收**（`:406-407`），而 LM Studio 的 `/v1/models` 不提供 digest。
实测也确认服务端未开 Ollama 兼容层（`GET /api/version` → `{"error":"Unexpected endpoint or method."}`）。

### 九条 `acceptance_gates` 逐条对照

| # | 门控 | 本方案 | 说明 |
|---|---|---|---|
| 1 | 服务端报告 `qwen3:0.6b` 且带非空不可变 model digest | ⚠️ 半满足 | **模型已对齐**（`qwen3-0.6b` 即 `Qwen3-0.6B`），但 LM Studio 的 `/v1/models` 不提供 digest，"不可变性"仍无法验证 |
| 2 | 渲染后的 prompt 保留 chat-template 特殊 token 与 tool schema | ❌ **无法验证** | OpenAI 兼容接口下 chat template 由服务端套用，客户端看不到渲染结果；`raw=true` 模式不存在 |
| 3 | 首次响应恰好含两个要求的工具调用 | ✅ | 见 3.1 |
| 4 | 两工具并发执行且结果可审计 | ✅ | 见 3.1 |
| 5 | 第二轮消费工具结果并终止 | ✅ | 见 3.1 |
| 6 | 保留流式分块、请求 prompt、计时、token 数、服务端耗时与哈希 | ⚠️ 部分 | `benchmark_all.json` 已完整产出计时与 token 数；但仍无 prompt 哈希与服务端耗时 |
| 7 | 保留匹配的 hit/miss TTFT 样本，且不要求有利结果 | ✅ | `benchmark_all.json` 保留了 5 对原始 TTFT 样本，并如实记录了 hit 臂 34 % 的波动 |
| 8 | 除只读 Open-Meteo 外全部本地执行 | ⚠️ 实质满足 | 推理在局域网另一台机器，**不出外网**，但严格说非本机 |
| 9 | 不发送或保留任何凭据 | ✅ | `OPENAI_API_KEY="EMPTY"`，LM Studio 不校验 |

**结论**：本方案能达成 2-1 的**学习目标**（工具调用闭环、并行工具执行、KV Cache 对 TTFT 的影响、批处理的吞吐-延迟权衡），
但**不能声称"通过了官方验收"**——gate 2 结构性不满足，gate 1 只满足一半（模型对了、digest 拿不到），gate 6 缺 prompt 哈希。

> 📈 相比上一轮 9B 的运行，换成 0.6B 后 **gate 1 从 ❌ 变 ⚠️、gate 7 从 ⚠️ 变 ✅**：
> 模型对齐了 protocol 的 `reference_model`，5 轮样本也补齐了。
> **剩下的缺口全是"接口形态"造成的，不是"没跑完"造成的**——再跑多少次也补不上，只能换 Ollama（见 6.2）。

---

## 五、protocol 偏离清单

| protocol 规定 | 本次实际 | 影响 |
|---|---|---|
| `server: "Ollama native /api/generate"` | LM Studio `/v1/chat/completions` | 无法 `raw=true`，gate 2 不可验证 |
| `model: "qwen3:0.6b"` | `qwen3-0.6b` | ✅ **同一模型**（只是 id 风格不同），性能数据与官方口径**可比** |
| `raw_mode: true` | 走 chat 接口，模板由服务端套用 | 看不到渲染后的 prompt |
| `temperature: 0` | ✅ 已按要求传 `--temperature 0` | — |
| `num_predict: 512` | ✅ `--max-tokens 512` | — |
| `approximate_prefix_tokens: 4096` | ✅ `--prefix-tokens 4096` | — |
| `matched_repeats: 5` | ✅ 5 轮全部完成，JSON 已落盘 | — |
| `warmups: 2` | ❌ **只预热 1 次**（`benchmark.py:194`） | 代码写死，无命令行开关；实测影响很小（见下） |
| `required_location: "Vancouver, Canada"` | ❌ 0.6B 只填 `'Vancouver'` | 工具靠 geocoding 补全，结果仍正确，但填参不达标 |
| 全本地执行 | 推理在 `192.168.1.2` | 不出外网，但非本机 |

> `warmups` 这条偏离是**代码层面的**，不是我没传参数：`benchmark.py:192-194` 硬编码只发一次预热请求，
> 命令行没有 `--warmups` 选项。要满足 protocol 得改代码（把 194 行再发一次，或包个循环）。
> 实际影响很小——从 hit 臂数据看，第 2 轮起就已稳定在 0.29–0.35 s，说明一次预热足够让缓存进入稳态；
> 真正的扰动来自每轮的 unique miss 请求（见 3.2），而不是预热次数不够。

---

## 六、待补项

### 6.1 真正要补的

| 项 | 说明 | 成本 |
|---|---|---|
| **查 LM Studio 的 GPU offload 与后端** | 实测带宽利用率仅 7.5 %（见 3.5），硬件远未用满。确认 GPU Offload Layers 是否拉满、Runtime 是否为 Vulkan，调对后重跑 throughput——**这是唯一有机会达到 M2 >100 tok/s 参照线的路径** | 服务端点几下 + 重跑 163 秒 |
| `warmups` 补到 2 次 | 改 `benchmark.py:194`（硬编码，无命令行开关） | 2 行代码；实测影响很小，可选 |
| 清理 `.env` 冗余 | `VLLM_HOST` / `VLLM_PORT` 命名已名不副实（实际连的是 LM Studio） | 可选 |

> ✅ 已完成并关闭：「KV Cache 补满 5 轮」、「throughput / batching 场景」、
> 「确认服务端硬件」（Windows 10 + AMD RX 5700，分析见 3.5）。

### 6.2 主动暂缓 / 结构性不可达

| 项 | 缺什么 | 备注 |
|---|---|---|
| `run_experiment.py` 官方验收 | 需 Ollama native `/api/generate` + model digest | 除非在 `192.168.1.2` 或本机装 Ollama 并 `ollama pull qwen3:0.6b` |
| gate 2（渲染后 prompt 校验） | 需 `raw=true` 的原始补全接口 | OpenAI 兼容接口结构上拿不到，**再跑多少次也补不上** |
| gate 1 的 digest 半条 | 需服务端提供不可变摘要 | LM Studio 的 `/v1/models` 没有这个字段 |

> 💡 **若想让 2-1 真正通过官方验收**，最省事的路径是在 `192.168.1.2` 上装 Ollama 并 `ollama pull qwen3:0.6b`
> （那台机器现在就在服务 `qwen3-0.6b`，拉同一个模型毫无压力），然后：
> `python run_experiment.py --base-url http://192.168.1.2:11434 --model qwen3:0.6b`。
> 这样 gate 1（digest）、gate 2（raw prompt）都能满足，只剩 gate 8 的"非本机"这一项偏离。
> 0.6B 的 Q4 权重约 520 MB，本机装 Ollama 也放得下（磁盘 35 GB、可用内存 8.8 GiB），
> 代价是纯 CPU 推理会更慢。

---

## 七、文件清单

```
chapter2/
├── README.md                       ← 本文件
└── local_llm_serving/              2-1
    ├── result.log                  工具调用完整运行记录（09-19 23:12，0.6B，约 10 秒）
    ├── benchmark_all.log           三场景基准的终端输出与汇总表（23:12–23:15，163 秒，exit_code=0）
    ├── benchmark_all.json          ★ 结构化结果：throughput 5 次 + kv-cache 5 对 + batching 4 级
    └── benchmark_kv_cache.log      历史数据：9B 的 KV Cache，⚠️ 仅 3/5 轮（保留作对比与教训）
```

**当前所有结论均以 0.6B 为准**（`qwen3-0.6b` @ `192.168.1.2:8234`）。`benchmark_kv_cache.log` 是上一轮 9B 的遗留，
仅用于 3.2 的历史对照与 timeout 教训，**不要再当作有效结果引用**。9B 那次**未产出** `benchmark_kv_cache.json`（进程被杀，未执行到写文件）。

**原仓库中被本次修改的文件**（不在本 note 库内，记录在此以便追溯）：

| 文件 | 改动 |
|---|---|
| `ai-agent-book/chapter2/local_llm_serving/agent.py` | 第 13 行导入追加 `MODEL_NAME`；第 248、379 行 `model="Qwen/Qwen3-0.6B"` → `model=MODEL_NAME` |
| `ai-agent-book/chapter2/local_llm_serving/.env` | `MODEL_NAME` → `qwen3-0.6b`；`VLLM_PORT` → `8234` |
| `ai-agent-book/.venv` | 装入 `openai` `python-dotenv` `requests` `PyPDF2` 四个包（原为 0 包空壳） |
