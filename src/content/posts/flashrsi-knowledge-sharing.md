---
title: "Agent 开始自己改 Harness：从 Harness RSI 原理、FlashRSI 框架到 RSI 算法"
published: 2026-11-23
description: "系统梳理 Harness RSI 核心原理、六大组件与 L0/L1/L2 循环，深入解析 FlashRSI 框架设计与 14 种 RSI 算法。"
tags: [RSI, Agent, Harness Engineering, FlashRSI]
category: "RSI"
draft: false
---
## 一、Harness RSI

**RSI:** 泛指系统能在反馈闭环里不靠人工持续变好；按改进作用的对象划分，当前 RSI 实践主要分为三个方向：

| 演化方向                           | 改进对象             | 典型做法                                                                          | 特点与边界                                   |
| ---------------------------------- | -------------------- | --------------------------------------------------------------------------------- | -------------------------------------------- |
| **Model 层（模型权重）**     | 底层神经网络参数     | 自我训练（Self-training）、自我对弈（Self-play）、可验证奖励强化学习              | 能力上限高，但训练成本与遗忘风险大           |
| **Harness 层（运行外壳）**   | 包裹模型的外围系统   | 冻结模型权重，迭代优化 Prompt、技能、工具、上下文管理与控制流                     | 迭代快、性价比高，是当前的主要研究方向       |
| **Artifacts 层（交付产物）** | 具体任务的产出物本身 | 通过“生成 → 测试验证 → 修正”闭环优化算法代码、算子或方案（如*AlphaEvolve*） | 直接产生可验收成果，但优化结果多绑定具体任务 |

本文讲述 直接决定 Agent 表现的 **Harness 层**。

### 为什么先做 Harness RSI

Lilian Weng（翁荔）在《*Harness Engineering for Self-Improvement*》中说：近期 RSI 不太可能从模型直接改写自身权重开始，更现实的路径是先在 **Harness 层** 跑出规模化收益。

* **优化 Harness 本身能显著降本**：几轮 Harness 迭代就能将推理成本最多降低 60%，靠把上下文管理得更合理、让多 Agent 协作更流畅等，而不是靠拉长模型的推理链条。
* **对大规模部署的中间档模型受益**：Lin 等（2026）将“提出有效改动的能力（*harness-updating*）”与“用好新 Harness 的能力（*harness-benefit*）”拆开评测，发现从 Qwen3-32B 到 Claude Opus 4.6，**不同规模模型提出改动建议的能力几乎持平**、波动很小；但“能否把新 Harness 用出收益”呈非单调分布——弱模型容易卡在未加载或执行出错，强模型已逼近任务上限，反而是 GPT-OSS-120B、Qwen3-235B 这类中间档位模型获益最大。

Harness RSI 为什么划算：**负责提修改建议的优化器不必依赖最昂贵的旗舰模型，可以使用中等主力模型。**

### Harness RSI 具体改什么

**模型**提供推理与生成能力；**Agent Harness** 决定模型看到什么上下文、能调什么工具、出错怎么重试、何时算做完。

传统靠人工读日志、手工改 Prompt 或控制流，不仅耗时，也很难规模化。

**Harness RSI**，就是让系统基于真实执行结果，自己诊断、修改并验证这套由文本、配置和代码构成的 Harness。

当模型权重保持冻结，Harness RSI 修改的内容主要落在以下五类资源上：

| 可改资源                   | 典型问题                               | 对应的改进方式                           |
| -------------------------- | -------------------------------------- | ---------------------------------------- |
| **指令（Prompt）**   | 目标含糊，输出格式或业务约束被忽略     | 重写规则、补齐步骤、收紧约束             |
| **技能（Skill）**    | 某类子任务反复踩同一个坑               | 沉淀可复用的操作流程、核验方法与适用条件 |
| **工具（Tool）**     | 动作接口难用、说明不清，或缺少必要能力 | 修改工具实现、工具描述或调用约定         |
| **执行代码（Code）** | 上下文截断、重试策略或终止判断不合理   | 改写运行循环、观察压缩、状态保留与控制流 |
| **配置（Config）**   | 运行参数、钩子或扩展设置与任务不匹配   | 调整允许修改的执行参数、Hook 与插件配置  |

---

## 二、Harness RSI 组件

一套完整的 Harness RSI 系统，通常由以下六个角色组成：

|               组件               | 一句话职责                                         | 经典论文中的典型形态                                                                                                            |
| :------------------------------: | -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **执行 Agent（Executor）** | 挂载当前 Harness 跑任务，产出结果与执行轨迹        | *OPRO* 的 Task LLM、*Harness²* / *RRSI* 中权重冻结的策略模型、*DGM* 中被测的编程 Agent                                 |
|  **诊断器（Diagnoser）**  | 把分数和长轨迹翻译成“错在哪、为什么错”的归因信号 | *Reflexion* 的语言反思、*TextGrad* 的文本梯度反馈、*GEPA* 的轨迹反思、*AHE* 的逐题 Debugger、*RRSI* 的 Critic         |
|   **提案器（Proposer）**   | 基于诊断或历史经验，产出下一版候选修改             | *OPRO* 的 Optimizer LLM、*AlphaEvolve* 的模型集成代码变异器、*DGM* 的自修改引擎、*Meta-Harness* 的 Agentic 编程提案器   |
|  **评估器（Evaluator）**  | 在真实或沙箱环境中客观度量候选版本的表现           | *AlphaEvolve* 的自动化评测脚本与性能分析器、*DGM* 的 SWE-bench 沙箱、*Harness²* 不依赖标准答案的行为对照 Probe           |
|   **选择器 (Selector)**   | 决定谁能留下来、谁能成为下一轮修改起点             | *EvoPrompt* 的种群淘汰、*GEPA* 的逐题 Pareto 前沿、*DarwinX* 的能力保留契约、*RRSI* 兼顾噪声带与 Token 成本的 Pruner    |
|     **记忆（Memory）**     | 跨轮保存历史版本、实测表现、失败教训与修改理由     | *ADAS* / *DGM* 的开放式设计档案、Google *CRSI* 提出的自改进记忆（SIM）、*Meta-Harness* 的文件化历史、*Arbor* 的假设树 |

**诊断器的作用**：如果只有“评估器打一个总分 + 提案器盲改”，系统很快就会在离散搜索空间里原地打转——这也是早期黑盒提示优化的瓶颈。*TextGrad*、*GEPA*、*AHE* 引入轨迹级诊断，本质是把“标量信号”升级成“可定位到具体步骤的语义信号”。

**Memory的作用：** 如果只保留“当前最好版本的代码”，不记录“之前为什么改、哪些尝试失败了、在什么条件下生效”，系统不仅会反复提出已经证伪的补丁，换一个数据集或换一个基座模型时还根本不知道旧改动该留还是该删。Google 在 *CRSI* 论文中提出的 **Self-Improvement Memory（SIM）**，专门针对这个问题：它把“问题归类、修改理由、预期与实测结果、评估上下文”记录下来，给予selector&Proposer查阅，而不是每次都失忆重来。

---

## 三、Harness RSI 循环

循环: “系统到底在进化什么”。Harness RSI 可以分成 **L0、L1、L2** 三个层级：

| 层级                         | 最终目标                                | 更新对象                             | 常见问题                                       |
| ---------------------------- | --------------------------------------- | ------------------------------------ | ---------------------------------------------- |
| **L0：单题解演化**     | 帮**当前这一道题**找到最优解      | 这道题的session、候选代码或解题方案  | 成果绑在单道题上，换题就得重来，受随机性影响大 |
| **L1：任务求解器演化** | 帮**一类任务**练出更强的 Agent    | 指令、技能、工具、执行代码及搜索历史 | 容易对少数题过拟合，新改动破坏已有能力         |
| **L2：改进器自身演化** | 让**“负责改进的机制”** 自己变强 | 诊断方式、提案策略、自改进记忆等     | 设计复杂，没有好门禁时很难稳定出收益           |

* **L0（为单题找最优解）**：代表如 *AlphaEvolve*、*EvoX*，哪怕过程中也在调整搜索策略，只要目标仍是当前这一道科学或算法题的最佳答案，就属于 L0。
* **L1（为同类任务造更强的 Agent）**：是目前落地的主流——从早期改指令（*OPRO*、*EvoPrompt*）、改工作流结构（*ADAS*），发展到轨迹反思与代码自修改（*GEPA*、*ACE*、*DGM*），再到 2026 年的完整 Harness 演化（*Meta-Harness*、*AHE*、*ModularRSI*、*RRSI*、*Arbor*），目标都是产出能跨题目甚至跨 Benchmark 复用的求解器。
* **L2（让改进器自己变强，同时避免无限套娃）**：
  * **独立改进器升级**：求解器与改进器分开，靠跨任务经验和验证门禁升级改进器（如 *Harness²*、*Self-Harness*、*VERSE*、*CRSI*，以及 *SkillOpt* 的 `meta-skill`）。
  * **避免套娃**：如果顺着 L2 继续往上叠“改改进器的改进器（L3）”，只会陷入无限套娃。*Hyperagents*（2026）直接把任务 Agent $A$ 和元改进器 $M$ 写进同一个可编辑程序 $H = (A, M)$，每轮通过 $H_{t+1} = M_t(H_t, \text{traces})$ 同时改进解题能力和自身的改进流程，用自指解决套娃问题。

### L1 的标准运行骨架

```text
1. 选起点（Select Base）   → 从当前版本、历史最佳、候选池或档案中挑出本轮修改基线
2. 跑任务（Execute）       → 执行 Agent 挂载该版本跑样本，收集答案、分数与完整轨迹
3. 做诊断（Diagnose）      → 诊断器分析失败案例、同题成败差异或历史记录，提炼病因
4. 提修改（Propose）       → 提案器产出新指令、结构化补丁或多文件代码改动
5. 过验证（Evaluate/Gate） → 评估器在验证集或对照任务上测量候选，核对门禁
6. 留状态（Select/Record） → 选择器决定接纳、拒绝或归档，把新版本与教训写回记忆
```

落地时主要有两种转法：

* **离线迭代搜索（Offline Evolution）**：在固定训练集和验证集上反复试错筛选，最后冻结出一个最优版本上线（如 *OPRO*、*GEPA*、*ADAS*、*DarwinX*、*RRSI*）。
* **在线流式编译（Online Stream）**：任务按顺序进来，每道题只跑一次，把失败教训边跑边编译进 Playbook 或技能库，直接服务下一批任务（如 *ACE* 的 online 模式、*Evo-Harness*）。

---

## 四、RSI 框架

### 4.1 背景：为什么 RSI 结果很难直接比

打开两篇 RSI 论文，A 说自己在 SWE-bench 上涨了 12 个点，B 说自己在多跳问答上涨了 15 个点。这两个数字几乎没法横向对比，它们可能同时换了**基座模型、Agent 框架、任务数据集**。最后看到的涨分，到底来自算法，更强的 Worker 模型、还是多跑了 Rollout，根本说不清。

**FlashRSI** 要解决的就是这件事。它把一次 RSI 实验拆成三个可以独立替换的模块：

```text
一次实验 = RSI 算法（Algorithm） × Agent 框架（Harness） × 评测任务（Benchmark）
```

只要把三者接到同一套接口上，就能固定 Agent 框架和评测任务，专门对比不同算法在相同起点的真实表现。

### 4.2 角色

|  FlashRSI 核心对象  |     承接的 RSI 角色     | 代码中的实际职责                                                                                                     |
| :-----------------: | :----------------------: | -------------------------------------------------------------------------------------------------------------------- |
|  **Target**  |      任务执行 Agent      | 挂载指定 `Harness Snapshot` 运行任务，通过 `rollout()` 返回 `Trajectory`；只有调用 `update()` 才切换默认版本 |
|  **Worker**  |     诊断器 + 提案器     | 通过 `ask()` 输出文本/JSON 提案，或通过 `edit()` 直接修改检出目录；模型与凭据完全独立于 Target                   |
| **Algorithm** | 搜索策略 + 选择器 + 记忆 | 声明读写范围，在 `evolve()` 里全权负责组织证据、调用 Worker、管理档案、执行准入并返回最终版本                      |
| **Snapshot** |      不可变版本载体      | 封装 Harness 声明的完整资源树、组件规则、SHA-256 内容标识与父子谱系                                                  |
| **Benchmark** |     任务源 + 评估器     | 通过 `load()` 提供 `train / dev / test` 划分，通过 `grade()` 给轨迹补齐主指标并封装为 `Results`              |
|  **Runner**  |    实验总控与审计记录    | 解析 YAML/CLI 配置、组装角色、绑定任务环境、落盘记录，并在搜索结束后统一在 `test` 上评估选定版本                   |

### 4.3 Snapshot

做 RSI 实验必须能回答：**这个评测结果，到底是哪一份提示词、技能和代码（哪个版本的Harness）跑出来的？** 如果只存零散的 diff 或改写建议，无法复现当时的真实行为。

FlashRSI 使用 **Snapshot **。它是一份**不可变、内容寻址的单文件 JSON。**

* Snapshot 内容：通过 `components.yaml` 声明纳入版本的全部文件（Base64 编码字节、组件标签、可执行位）、`component_rules`、`agent_type`，以及 `root_id` 与 `parent_id`。

每个 Harness 先用 `components.yaml` 声明自己愿意暴露哪些资源。

```yaml
# MiniAgent：干净开源，直接暴露运行时代码
code:   [runtime/**, middleware/**, sub_agents/**]
skill:  [skills/**]
prompt: [prompts/**]
tool:   [tools/**, tool_descriptions/**]
config: [agent.yaml]
```

```yaml
# Claude Code：闭源 SDK，只暴露稳定扩展接口（没有 code 组件）
prompt: [CLAUDE.md, output-styles/**]
skill:  [skills/**, commands/**]
agent:  [agents/**]
hook:   [hooks/**]
tool:   [.mcp.json, .lsp.json, servers/**]
config: [.claude-plugin/plugin.json, settings.json]
```

然后，每个 RSI 算法再声明自己的读写范围：比如 SkillAdam 只声明 `{"skill"}`，AHE 声明 `{"*"}`（表示当前 Target 暴露了哪些它就都能改)。

* **修改时（局部放行）**：算法调用 `snapshot.check_out()` 给 Worker 准备临时工作目录时，放出被允许看和改的那几个文件，防止误改。
* **评测时（完整执行）**：Worker 改完后，框架将改动与未改文件合并生成新的 Snapshot；Target 跑任务时，始终把新 Snapshot 的全部文件还原出来执行，测出整体真实表现。

### 4.4 Harness 接入

|              如何接入              |                Harness                | 具体做法                                                                                                                                     |                                                                       修改范围/注意事项                                                                       |
| :--------------------------------: | :-----------------------------------: | -------------------------------------------------------------------------------------------------------------------------------------------- | :-----------------------------------------------------------------------------------------------------------------------------------------------------------: |
|      **直接用原版代码**      |        MiniAgent、JiuwenSwarm        | 本身就是纯 Python，直接把原文件收进 Snapshot 打上标签，跑的时候直接加载（JiuwenSwarm 只做 Target）                                           |                                                                代码、提示词、技能、工具都能改                                                                |
|        **接官方 SDK**        |              Claude Code              | 通过官方 Agent SDK 调起，闭源主循环留在官方那边，Snapshot 只管官方开放出来的外围配置                                                         |                                 只能改 `CLAUDE.md`、skills、hooks、MCP/LSP 和 settings，**改不了闭源产品内部的行为**                                 |
| **用 Python 自己重写核心版** | Codex、Pi、Hermes、OpenClaw、OpenCode | 很多原版依赖重或不是 Python 写的（如 Rust / Node / Bun CLI），FlashRSI 锁死某个上游版本，自己用纯 Python 重写它的无界面核心循环装进 Snapshot | 重写后的核心代码和资源都能做版本管理并演化，但**只是精简版本**（如 Codex 没覆盖 WebSocket、HTTP/OAuth MCP 和完整执行策略；Pi 只对齐 v1.4.0 无界面循环） |

### 4.5 流程(需要改)

![FlashRSI 主要架构：装配、评估、生成候选](../../assets/images/flashrsi-architecture.svg)

把 14 种算法放在 `flashrsi/core/algorithm.py` 的公共基类上拆解，一次演化共用六步基础操作：

|     步骤     | 做什么                                                            | 谁做决定             |
| :-----------: | ----------------------------------------------------------------- | -------------------- |
|   1. 选起点   | 当前版本、历史最佳、候选池成员，或每次回到初始 `seed`           | Algorithm            |
|  2. 组织反馈  | 分数、失败轨迹、源码文件、历史经验或上一轮判决                    | Algorithm            |
|  3. 产生修改  | `worker.ask()` 返回文本/补丁，或 `worker.edit()` 改目录       | Worker               |
|  4. 提交候选  | 校验路径、组件权限与凭据，生成不可变 Snapshot                     | Snapshot + Algorithm |
| 5. 执行与评分 | 在指定任务上运行完整候选，经 `Benchmark.grade()` 得 `Results` | Target + Benchmark   |
|  6. 更新状态  | 接纳、拒绝、入档、写记忆，或调整下一轮预算/策略                   | Algorithm            |

FlashRSI 在框架层的三条规则：

1. **Harness 资源与 Benchmark 环境彻底隔离。** `runner.py` 绑定完 Benchmark 集成后，会强制校验 `target.snapshot.id == selected`——如果某个任务集成偷偷替换了 Harness 版本，启动直接报错。Harness 从 `files_dir` 读自己的版本，通过 `working_dir` 操作任务，两者绝不混在一起。
2. **容器隔离与宿主机评分。** 已接入的纯文本任务（SearchQA、HotpotQA context、MATH 等）通过 `supplied_context` 在每题独立的 Docker 容器里跑 Target，只挂载公开题目和选中 Snapshot，容器网络只放行模型端点；原始标注、评分器、Judge 和用户模拟器全部留在宿主机，防止 Agent 偷看答案或历史目录。
3. **评测集划分 `train / dev / test` 。**
   * `train` 产生训练轨迹与诊断证据（OPRO、AHE、DarwinX、ModularRSI、RRSI 甚至直接用 train 评分选优）；
   * `dev` 用于筛选和比较候选（一旦反复参与选择，它就已是搜索集，不再是独立泛化证明）；
   * `test` 留给 `Runner` 在搜索结束后统一评一次。`flashrsi run` 结束时只评最终选中的版本，不会自动补评初始版本。

### 4.6 示例：一道实体混淆题，怎样变成可复用技能

看第一道题（`demo-01`）：材料说“Aurora 由 Mira Chen 协调，Mira Chen 在 Bern 工作；Noah Reed 在 Oslo 协调 Project Harbor”，问题问 Aurora 协调员在哪个城市工作。

假设初始版本 `S0` 受干扰实体影响，错答了 `Oslo`。用教学视角走一遍 SkillOpt 的流程：

1. **定基线**：初始指令、工具和技能组成 `S0`，先在 3 道 `dev` 题上测出基准分。
2. **跑训练**：用 `S0` 跑本批 `train` 题，拿到轨迹与得分（`demo-01` 失败）。
3. **做诊断**：Worker 看到轨迹，归因是“多跳实体关系追踪不完整，误用了邻近干扰人物的城市”。
4. **提有界补丁**：Worker 按 JSON 协议提出一条针对 `skill` 组件的修改，写入通用核验规则：
   ```text
   多跳实体核验：先锁定问题主体，逐条追踪关联关系；
   引用最终属性前，确认该属性明确归属于已锁定实体，排除邻近干扰项。
   ```
5. **完整物化测候选**：提交新技能形成 `S1`（其余 Prompt、工具和代码不变），在完整 `dev` 上重测。
6. **过门禁**：SkillOpt 要求 `dev` 均分严格提升才调用 `target.update(S1)`；同分或下降就保留 `S0`，并把被拒补丁记进拒绝缓冲，提醒下一轮别重踩。
7. **考未见题**：搜索结束，Runner 用最终选定的版本跑 3 道 `test` 题，检验规则是真学会了“防实体混淆”，还是只背下了 `Bern`。

## 五、FlashRSI适配的部分 RSI 算法拆解

rsi算法主要差别：**改什么资源、给 Worker 看什么证据、用什么数据结构记历史、按什么规则留下新版本。**

| 分组                         | 算法                   | 一句话核心思想                                          | FlashRSI 实际改什么     | 跨轮维护的核心状态                                         | 保留与最终输出规则                                                        |
| ---------------------------- | ---------------------- | ------------------------------------------------------- | ----------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------- |
| 指令                         | **OPRO**         | 把历史“指令—分数”曲线喂给模型做黑盒提案              | 单个指令文件            | 全部已测指令与 train 分数、`_leader`                     | 已测全入历史；最终取 train 最高分                                         |
| 指令                         | **EvoPrompt**    | LLM 做自然语言交叉/变异，GA 或 DE 管淘汰                | 单个指令文件            | 固定大小种群、跨代全局 `best`                            | GA 整代选幸存者，DE 逐位严格替换；最终取跨代 dev 最佳                     |
| 指令                         | **GEPA**         | 轨迹反思改指令，逐题 Pareto 前沿保留互补探索方向        | 单个指令文件            | 持续追加的候选池、逐题 dev 分数、父索引                    | train 小批次过筛后测完整 dev 即入池；最终取池中 dev 均分最高              |
| 指令                         | **ACE**          | 用增量 Playbook 条目避免整段重写丢经验                  | 指令文件的受管后缀      | 条目、helpful/harmful 计数、拒绝记录                       | 合法更新立即采用；offline 取测得 dev 最佳，online 返回末版                |
| 技能与经验                   | **SkillOpt**     | 有界补丁 + 拒绝反馈 + 跨 epoch 慢整理与元技能           | `skill` 组件文本文件  | 当前技能、拒绝补丁缓冲、上轮 epoch 版本、`meta-skill`    | 快慢资源候选均须严格提升 dev；返回最后接受版本                            |
| 技能与经验                   | **SkillAdam**    | 问题记忆稳方向，逐题分差波动调编辑预算                  | 单个已有 `.md` 技能   | `issue memory`、逐题分差方差 EMA、dev 基线               | 严格提升 dev 才更新技能；被拒尝试仍更新记忆与方差                         |
| 技能与经验                   | **Evo-Harness**  | 单次任务流失败经验，编译成两级原生技能目录              | 原生 `skills/` 目录   | 当前技能库、本批失败经验                                   | Topic / General 合法即采用，无 dev 门槛；返回流结束末版                   |
| 完整 Harness（迭代/档案）    | **ADAS**         | 用设计说明与分数组成的档案，探索 Agent 结构             | 全部已声明组件（`*`） | 设计档案（`name/thought/fitness`）、`best`             | 从 `best` 检出并评前复查，全入档；最终取 dev 最佳                       |
| 完整 Harness（迭代/档案）    | **Meta-Harness** | 把完整历史源码与轨迹放进文件系统，让提案 Agent 自己找   | 全部已声明组件（`*`） | `seed`、历代源码与轨迹、`best`                         | 每次从 `seed` 检出，Worker 自由复用历史；最终取 dev 最佳                |
| 完整 Harness（迭代/档案）    | **AHE**          | 每次改动必须附带可核对的效果预测清单                    | 全部已声明组件（`*`） | `current`、上轮判决、回滚原文、拒绝记录                  | 合法且测完的候选全部留作 `current`（含降分）；返回最后保留版            |
| 完整 Harness（约束/模块/树） | **DarwinX**      | 保住已有能力再扩展，并从落选分支中重组互补能力          | 全部已声明组件（`*`） | 全量档案、已准入父池、能力保留集 `invariant`             | 小样本筛选 + 重复全量确认决定准入；最终取已准入者最高存档 train 分        |
| 完整 Harness（约束/模块/树） | **ModularRSI**   | 按功能模块限定文件，用同题成败对照独立演化再集成        | `modules` 映射文件    | 各模块独立分支、同题成功记忆、改动史                       | 文件/语法/完整 diff review/执行门禁决定前进（无涨分门槛）；按回退规则输出 |
| 完整 Harness（约束/模块/树） | **RRSI**         | 提案侧收缩机制预算，选择侧卡噪声带与 Token 成本         | 全部已声明组件（`*`） | `incumbent`、历史最高分 `S*`、噪声带 `delta`、编辑史 | 先判准入，再在合格兄弟中选分数最高者；返回最后接纳版                      |
| 完整 Harness（约束/模块/树） | **Arbor**        | 用假设树管研究方向，实验从 `trunk` 出发，经验向上综合 | 全部已声明组件（`*`） | 假设树（含 `insight`）、当前 `trunk`                   | train 过筛且 dev 严格提升才更新 `trunk`；返回最终 `trunk`             |

---

### 5.2 第一类：指令与 Playbook 演化（改“怎么说”）

这一组四套方法都作用于 `prompt` 组件的单个指令文件，只需要支持 `ask()` 的纯 LLM Worker（也可以换 Agent Worker）。

#### OPRO：把“历史指令与分数”变成下一轮的输入

**要解决什么问题。** 离散的自然语言指令没法算数值梯度。Google DeepMind 的 OPRO 思路极简：既然大模型擅长从规律里做续写，那就把过去试过的“指令 + 实测分数”按从低到高排好队塞进提示词，让它顺着上升趋势猜下一条更好的指令。

**在 FlashRSI 里怎么转。**

* **编辑范围与数据**：只替换 Target 声明的 `instruction_file`；**全程使用 `train` 评分与选优**，不设 `dev` 门槛。
* **一轮流程**：
  1. 在完整 `train` 上评 `S0`，作为第 0 条记录放进 `_history`。
  2. 每轮挑历史 `train` 最高分记录作 `parent`（同分取较早者）。
  3. 拼装 Worker 提示：过滤掉原始分数低于 `score_threshold`（默认 `0.1`）的记录，取其中最高分的至多 `history_size`（默认 `20`）条，分数乘 100 取整后按升序展示；再从当前领跑者 `_leader` 的轨迹里随机抽 `few_shot_size`（默认 `3`）道“问题 + 输出 + 指标”作样例。
  4. 用同一份提示调 Worker `num_candidates`（默认 `8`）次，提取最后一对 `<INS>...</INS>`（无标签则取整段回复）。空回复和重复指令直接丢弃，不补采。**本轮所有提案都生成完，才统一开始评估。**
  5. 每个新指令都基于本轮同一个 `parent` 构造候选，跑完整 `train`。无论高低全部写入 `_history`，严格破纪录才更新 `_leader`。
  6. 跑完 `max_steps`（默认 `200`）轮，一次性 `_promote` 历史 `train` 最高分版本（同分取较早者）。

```text
H = [评估(S0, train)]
重复 max_steps 轮：
    parent = H 中 train 最高分（同分取早）
    texts = Worker.ask(升序高分历史, _leader 任务样例) × num_candidates
    对本轮非空且未见过的指令：
        candidate = 替换 parent 的 instruction_file
        H.append(评估(candidate, 完整 train))
返回 H 中 train 最高分 Snapshot
```

**边界与成本。** `score_threshold` 只管“展示给 Worker 看哪些”，不会删历史，也不限制最终选优。设 `train` 有 $N$ 题、轮数 $S$、每轮 $C$ 候选，最多要跑 $(1 + S \times C) \times N$ 次 Target 任务——默认 `200 × 8` 规模惊人，实操时务必按预算调小。与原论文把指令插在题目不同位置相比，FlashRSI 收敛为重写同一个原生指令文件。

---

#### EvoPrompt：LLM 当自然语言算子，进化算法决定谁活下来

**要解决什么问题。** 指令是强语义文本，像传统遗传算法那样按字词随机切开交叉，拼出来的多半是乱码。EvoPrompt 保留经典进化算法的种群和淘汰骨架，把“交叉”和“变异”外包给 LLM 在语义层面完成，提供 **GA（遗传算法）** 和 **DE（差分演化）** 两种变体。

**在 FlashRSI 里怎么转。**

* **编辑范围与数据**：只改单个 `instruction_file`；**只用 `dev` 评适应度**，不读 `train` 轨迹，也不做失败归因。
* **初始化**：先评 `S0` 的完整 `dev` 作为第 1 个成员，再让 Worker 把原始指令改写 $P - 1$ 次、逐个测 `dev` 填满种群（默认 `population_size = 10`，GA 至少 2，DE 至少 4）。空回复或换行规范化后与父指令相同的提案不重测，直接沿用父版本与分数（记 `carried: true`），但仍占据一个种群位置。
* **两种演化算子的核心差别**：
  * **GA（整代提出，统一选优）**：按 `selection`（默认 `wheel` 轮盘赌，可选 `random` / `tournament`）抽父对，让 Worker 语义交叉并变异出 $P$ 个子代，全测完完整 `dev` 后统一选幸存者。默认 `survivor_selection: topk` 把父代和子代混在一起按分挑前 $P$（优先不同指令，不够再用高分补齐）；`standard` 则直接用全部子代换掉旧种群。
  * **DE（逐位竞争，只换目标位置）**：冻结本代初始种群，对每个位置 $i$ 的目标指令 $p_i$，从其他位置抽 3 个不同 donor，让 Worker 比较两条 donor 的差异、变异差异、结合基底指令（默认本代最佳，`donor_random: true` 才用第三个 donor）再与 $p_i$ 交叉。新生成的试验候选只在 `dev` 分数**严格大于** $p_i$ 时才替换位置 $i$，同分保留原成员，且不影响其他位置。

```text
GA (默认 topk): P_next = topk(P_old + children)
DE:            P_next[i] = trial_i  if score(trial_i) > score(P_old[i]) else P_old[i]
```

**最容易看错的细节。** 无论 GA 还是 DE，实现里都**单独维护一个跨代全局 `best`**（仅严格提分时更新）。哪怕 `standard` GA 在最后一代不小心把曾经的冠军淘汰出了活动种群，搜索结束 `_promote` 时交出来的依然是那个历史最高分。设种群 $P$、代数 $G$（默认 10）、dev 大小 $D$，最多执行 $P \times (1 + G) \times D$ 次 Target 任务；除与父指令完全一致外没有全局去重缓存，重复提案可能重复花钱。

---

#### GEPA：轨迹反思提修改，逐题 Pareto 前沿保留互补能力

**要解决什么问题。** 只看一个总分，优化器不知道具体错在哪里；如果始终只从均分最高的版本继续迭代，又容易过早淘汰那些“整体均分略低，但在特定题目上表现突出”的候选策略。GEPA（Reflective Prompt Evolution）同时解决这两个问题：用**执行轨迹反思**生成更有针对性的修改，用**逐题 Pareto 前沿**为具备局部优势的候选保留继续作为父版本的机会。

在 FlashRSI 的 14 种算法里，GEPA 是把“入池、抽样、最终选优”三件事切得最清楚的一个，值得完整拆透。

**1. 改什么、给 Worker 看什么。**
同样只重写单个 `instruction_file`，但同时用 `train` 和 `dev`（且要求 `dev` 任务 ID 唯一）。Worker 收到的不是泛泛的“请优化”，而是父指令加上本轮训练小批次里每道题的**问题、父版本输出、指标、参考答案（若有）、状态/错误以及完整执行消息 `trajectory`**。

**2. 候选究竟怎么进池子？**
初始化先在完整 `dev` 上评 `S0`，建立只含 1 个成员的 `_population`。注意，这个池子是**只追加、不删成员**的列表，每个 `Member` 记录 `index`、`snapshot`、`text`、逐题分数 `per_task`、完整 dev 均分 `score` 以及 `parents`。每轮严格按七步走：

1. **选父版本**：按 `candidate_selection_strategy`（默认 `pareto`，可改 `current_best`）从池里挑 `parent`。
2. **跑 `train` 小批次**：用 `parent` 跑 `reflection_minibatch_size`（默认 3）道题，得到基线 `before`。
3. **筛反思证据**：默认 `skip_perfect_score: true`，过滤掉主指标恰为 `1.0` 的轨迹；如果全满分没有可反思的题，本轮直接以 `perfect_minibatch` 结束（注意：这只过滤给 Worker 看的证据，后面父子对比仍比整个小批次）。
4. **反思生成新指令**：空回复、与父相同、或换行规范化后已在池中出现过的指令直接拒绝。
5. **同批复测**：把新指令写入 `parent` 构成候选，在**同一个小批次**上跑出 `after`。
6. **小批次快筛**：默认 `strict_improvement` 要求 `after.score > before.score`（改为 `improvement_or_equal` 允许同分）。过不了就不测 `dev`、不入池。
7. **测完整 `dev` 并无条件入池**：过了快筛，就跑完整 `dev`，把新 `Member` 追加进 `_population`。**关键点来了：这里没有“dev 必须胜过父版本或历史最佳”的第二道门槛！** 哪怕一个候选 `dev` 均分只有 0.25，只要刚在小批次赢了父版本，它照样合法入池。

```text
P = [Member(S0, 评估(S0, dev))]
while 预算与 epoch 允许：
    parent = 从 P 的逐题前沿抽样
    M = 一个 train 小批次
    before = 评估(parent, M)
    evidence = 过滤满分后的 before 轨迹；若空则跳过
    text = Worker.ask(parent.text, evidence)；空/未变/重复则跳过
    candidate = 替换 parent 的指令文件
    after = 评估(candidate, M)；未通过小批次筛选则跳过
    dev_res = 评估(candidate, 完整 dev)
    P.append(Member(candidate, dev_res, parents=(parent.index,)))
返回 P 中完整 dev 均分最高者（同分取更早入池者）
```

**3. 逐题前沿怎么决定“谁还有机会当父版本”？**
设池里有成员 $S_0 \dots S_k$，dev 有题目 $x_1 \dots x_D$。`frontier()` 先对每道题找出最高实测分的并列赢家集合：

$$
F[j] = \{i \mid \text{score}(S_i, x_j) = \max_k \text{score}(S_k, x_j)\}
$$

接着 `prune()` 做**保持覆盖的冗余裁剪**：按 `dev` 均分从低到高、索引从早到晚检查每个成员——如果删掉它后，它覆盖的每道最优题仍有别的活跃成员覆盖，就把它从本次抽样前沿里剔除（注意：只从抽样关系里剔除，绝不从 `_population` 删除）。最后把每道题裁剪后的集合直接摊平为一个列表，均匀随机抽一个位置：

```text
weighted = [i for j in dev_tasks for i in pruned_F[j]]
parent = P[random.choice(weighted)]
```

也就是说，**成员 $i$ 赢过（或并列赢过）的题越多，被抽中当父版本的概率就越高**。

**4. 用四个候选彻底把“入池、被抽中、最终获胜”拆开。**
假设经过几轮小批次快筛，池里有了 A、B、C、D 四个成员：

| 成员      | dev 题 1    | dev 题 2    | dev 题 3    | dev 题 4    | dev 均分       | 逐题角色                      |
| --------- | ----------- | ----------- | ----------- | ----------- | -------------- | ----------------------------- |
| A（种子） | **1** | **1** | 0           | 0           | 0.50           | 唯一拿下题 1；与人并列题 2    |
| B         | 0           | **1** | **1** | **1** | **0.75** | 唯一拿下题 3、4；并列题 2     |
| C         | 0.5         | 0.5         | 0.5         | 0.5         | 0.50           | 已合法入池，但没赢过任何一题  |
| D         | 0           | **1** | 0           | 0           | 0.25           | 只并列题 2，完全可被 A/B 替代 |

原始前沿是 `F1={A}, F2={A,B,D}, F3={B}, F4={B}`。C 未进入前沿；D 的覆盖被 A/B 完全覆盖，被 `prune()` 裁掉。剩下摊平为 `[A, A, B, B, B]`，于是下轮抽父版本的概率是 **A 40%、B 60%、C 0%、D 0%**。

* A 均分不如 B，却因为独家会做题 1，保住了 40% 的探索机会；
* C 和 D 虽然抽样概率为 0，依然完整留在 `_population` 里；
* 如果此时预算耗尽，最终 `_promote` 交出的是 **B**（按完整 `dev` 均分取最高，同分取更早入池者）。

**预算与论文差异。** `max_metric_calls` 统计的是 **Target 任务评估次数**（初始 $D$ 次，完整一轮 $2B + D$ 次，不含 Worker 调用和最终 test），每轮开始前按完整开销检查剩余预算；未指定时自动计算下限。要特别留意与原论文的边界：**FlashRSI 本地实现只重写一个指令文件，不支持多模块轮换，也没有实现论文的 system-aware merge（设 `use_merge: true` 会在启动时直接报错）。**

---

#### ACE：用增量 Playbook 避免整段重写导致的“经验坍塌”

**要解决什么问题。** 每轮让模型把整份系统提示重写一遍，模型天生喜欢“概括精简”，改着改着就会把之前辛辛苦苦攒下来的细粒度避坑规则压扁、写丢。ACE（Agentic Context Engineering）把上下文当成一本只增不乱改的 **Playbook**，用 `Generator → Reflector → Curator` 三段分工按条目增量累积。

**在 FlashRSI 里怎么转。**

* **编辑范围**：保留原指令前缀不动，只管理 `<!-- flashrsi:ace-playbook -->` 标记之后的后缀区域。每条 bullet 带有自动分配的分节编号、正文和 `helpful / harmful` 计数。三种角色靠提示词切换，不需要配三套模型。
* **一轮流程**：
  1. **Generator**：Target 用 `current` 跑一批 `train`（`batch_size` 默认 1）。若为 `online` 模式，在任何反思、重试或更新之前，先把这首次尝试成绩计入 `online_score`。
  2. **Reflector**：Worker 读取当前 Playbook 以及本批题目的输入、输出、参考答案（若有）、分数、状态和错误（注意：当前实现不把完整工具调用消息逐条塞给 Reflector），产出根因、经验和已有 bullet 的 `helpful / harmful` 标签。如果任务未完成或低于 `success_threshold`（默认 `1.0`）且还有反思轮数（`max_reflection_rounds` 默认 3），就把最新诊断拼在原问题前面让 Target **用同一个 Snapshot 重试**，直到达标或轮数耗尽（注意：带 `Task.env` 的环境任务目前强制要求 `max_reflection_rounds: 1`）。
  3. **记账立即生效**：把 `helpful / harmful` 标签累加到对应 bullet 计数，只要有变化就立刻采用为新 `current`。
  4. **Curator 增量策展**：每 `curator_frequency`（默认 1）个 batch，让 Worker 基于最后一次诊断输出结构化操作。**本地目前仅支持 `ADD` 操作**（校验去重前的 `edit_budget` 默认 4、近似大小上限 `playbook_token_budget = ceil(字符数 / 4)`、以及空白与大小写规范化去重）。合法新增同样**立即采用，不设当场涨分门槛**；最近 4 次非法拒绝会留作后续策展警示。

**Offline 与 Online 的本质区别。**

* `offline` 按 `seed + epoch` 打乱训练，每 `eval_steps`（默认 100）个 batch 测一次完整 `dev`（严格更高分才更新 `best`），结束时若 `current` 变过就补评一次 `dev`，再恢复测得的 `best`；
* `online` 保持原顺序消费、不碰 `dev`、不回滚，直接把最后攒出的 `current` 交给 `test`。

**容易混淆的边界。** `online_score` 衡量的是“边跑边学过程中每批更新前的首次尝试表现”，和最后拿冻结末版去跑 `test` 得到的 `test_score` 是两码事。另外，本地实现两种模式都会把可用 `ground_truth` 交给 Reflector，因此本地 `online` 不能直接等同于论文里的无标签在线设置；原论文的语义去重与精简压缩在本地也尚未实现。

---

### 5.3 第二类：技能与经验编译（沉淀“怎么做”）

当指令文件越写越长，把具体操作流程抽出来放进独立技能文件，就进入了第二类方法。

#### SkillOpt：有界快更新 + 跨 epoch 慢整理 + 优化器元技能

**要解决什么问题。** 直接把技能文档丢给模型自由改写，很容易删掉已有有效规则、引入自相矛盾，甚至在不同批次里把同一个已经被证明没用的点子反复提出来。SkillOpt 把技能当成冻结 Agent 的外部可训练参数，用**结构化精确定位补丁、编辑预算、拒绝记忆和快慢双时间尺度**把技能优化管起来。

**在 FlashRSI 里怎么转。**

* **编辑范围与角色**：`evolve_scope = {"skill"}`，可改 `skill` 组件内的多个文本文件；Worker 只需 `ask()` 输出 JSON 补丁，不需要文件编辑权限。同时要求 `train` 与 `dev` 非空。
* **快更新（每个 train batch）**：
  1. 先在 `dev` 上评 `S0` 存为基线。
  2. 每个 epoch 打乱 `train` 切 batch（默认 `batch_size = 40`），用当前版本跑完后，**先切反思小批次**（`reflection_minibatch_size = 8`），**再在每个小批次内划成败**（成功要求 `finished` 且主指标**严格大于** `success_threshold = 0.0`）。
  3. 成功组提炼该推广的做法，失败组提炼纠错规则；先同类合并，再按“失败优先”合并两类提案，超过 `edit_budget`（默认 4）时再由 Worker 排序截断。
  4. 应用 JSON 补丁，支持 `append / insert_after / replace / delete`——除 `append` 外，定位锚点必须在目标文件里**恰好出现一次**；只有 `append` 可以建新文件。
  5. 有实际改动才测完整 `dev`。**必须 `candidate_score > incumbent_score` 才调用 `target.update()` 接受**；同分或降分保留旧版，并把最近最多 5 个被 `dev` 拒绝的快补丁放进拒绝缓冲，供下轮反思避坑。
* **慢更新与 `meta-skill`（从第 2 个完整 epoch 结束开始）**：
  只要一个 epoch 完整跑完（中途耗尽 `max_steps` 不触发），算法就抽 `slow_update_samples`（默认 20）道 `train` 题，把上一 epoch 末版本和当前版本同台重跑一遍，开启两条独立开关：
  * `use_slow_update`：提炼跨 epoch 的长期指导，写入技能文件的受保护区块 `<!-- SLOW_UPDATE_START --> ... <!-- SLOW_UPDATE_END -->`（快补丁严禁触碰该区），同样必须严格提升 `dev` 才接受；
  * `use_meta_skill`：提炼**“怎么更好地改技能”的优化器记忆**，只保存在算法内部并喂给下一轮快反思，**不写进 Target 技能文件，也不单独过 `dev` 门禁**。

**细节与成本。** 因为每次接受都要求 `dev` 严格递增，最后返回的已接受版本天然是接受链上的最高 `dev` 分版本。要注意 `max_steps` 数的是 `train` batch 数，不是模型调用次数——一次快更新内部就可能包含多轮反思、合并和排序调用。

---

#### SkillAdam：借用 Adam 思想——记住试过什么，按波动调步子大小

**要解决什么问题。** 只盯着当前这批错题改，优化方向容易左右摇摆；而且同样的修改幅度，在前期大刀阔斧没问题，到了后期一动就容易顾此失彼。SkillAdam 直接把深度学习里 Adam 优化器的一阶动量、二阶自适应步长概念，翻译到了文本技能优化上。

**在 FlashRSI 里怎么转。**

* **编辑范围与前提**：`evolve_scope = {"skill"}`，要求已有单个 Markdown 技能文件（如果有多个 `.md`，必须通过 `skill_path` 明确指定），不自动从零生成技能；Worker 走 `ask()`，同时要求 `train`、`dev` 非空且 `dev` 任务 ID 唯一。
* **三份跨轮状态**：
  * `memory[label]`（一阶方向记忆）：记录每个反复出现的问题标签、`open / resolved` 状态、最新证据，以及过去尝试过的 `approach` 和是否过了 `gate_accepted`；
  * `v_ema`（二阶波动估计）：逐题 `dev` 分差样本方差的指数滑动平均；
  * `baseline`：当前已接受技能在固定 `dev` 上的均分与逐题分数。
* **一轮流程**：
  1. 先用上轮 `v_ema` 算出本轮编辑预算 `budget`（公式见下），再无放回抽 `batch_size`（默认 8）道 `train` 题跑当前技能。
  2. Worker 读当前技能全文、训练轨迹、完整 `issue memory` 和本轮 `budget`，返回包含 `issues` 和 `edits` 的 JSON（每条 edit 关联问题标签，给出 `approach`、文件内唯一精确片段 `before` 与替换文本 `after`）。
  3. 校验条数、标签引用和精确匹配；有实际变动才测完整 `dev`，并与保存的 `baseline` **逐题配对相减**。
  4. **无论本轮候选最终接不接受，只要测出了逐题分差，就立刻更新 `v_ema`；并且把本轮观测与 `gate_accepted` 写回 `memory`！**
  5. 只有 `dev` 均分**严格高于** `baseline` 才 `target.update()` 更新技能和基线。

编辑预算的自适应公式非常直观（默认 `base_budget=4, min_budget=1, beta=0.9, v_max=0.25`）：

```text
d[i] = candidate_dev[i] - baseline_dev[i]
sigma_sq = variance(d)                   # 样本方差；仅 1 题时为 0
v_ema = beta * v_ema + (1 - beta) * sigma_sq
ratio = min(v_ema / v_max, 1.0) if v_max else 0.0
next_budget = max(min_budget, round(base_budget * (1 - ratio)))
```

**一个一眼秒懂的算例。** 假设两次修改在 4 道 `dev` 题上的平均分差都是 0：第一次逐题变化是 `[0, 0, 0, 0]`，方差为 0；第二次是 `[+1, -1, +1, -1]`，拆东墙补西墙，样本方差高达 `4/3`。两者都会因均分没涨而被拒绝，但第二次会把 `v_ema` 拉到约 `0.1333`，直接把下一轮编辑预算从 4 条压到 `round(4 × (1 - 0.1333/0.25)) = 2` 条。**预算盯住的是“题间效果撕裂程度”，而不只看均分涨跌。**

**与原论文的差别。** 原论文的 EIT 模块会在验证后额外调一次模型结合验证反馈更新问题记忆；FlashRSI 把修改和问题观测合并在同一次 Worker 提案里，再追加真实门禁结果，用固定 `dev` 配对分差驱动方差更新。`issue memory` 属于优化器私有状态，不会导出给 Target。

---

#### Evo-Harness：单次任务流，把失败直接编译成两级原生技能

**要解决什么问题。** 真实线上任务往往像水流一样只经过一次，没有反复重跑多轮 epoch 的奢侈条件；而且原始上下文里混着一次性任务细节和可复用套路。Evo-Harness 走的是**无重放、流式经验编译**路线——把单次失败先提炼成结构化教训，再沉淀进 Harness 原生目录的两级技能库，让下一批任务自然加载使用。

**在 FlashRSI 里怎么转。**

* **编辑范围与前提**：`evolve_scope = {"skill"}`、`observation_scope = {"skill", "config"}`。**必须用具备 `edit()` 能力的文件编辑 Agent Worker**，且 Target 必须支持原生目录技能发现（例如 MiniAgent 的 workspace loader + `read_skill`，或 Claude Code 的原生 `Skill` 工具）。要求 `train` 非空且 ID 唯一；**不做 `dev` 选优、不跑多 epoch、不重试当前题**。
* **每个 batch 的编译流**：
  1. 整批任务（默认 `batch_size = 4`）用同一个固定 `current` 各跑一次，先记录更新前分数。
  2. 只挑选未完成或主指标低于 `success_threshold`（默认 `1.0`）的失败任务反思，Worker 返回 `{lesson, trigger, evidence, scope_hint}` 或 `{"skip": true}`。
  3. 只要有合法经验，先调 **Topic Curator**（`worker.edit()`），在原生目录如 `skills/<name>/SKILL.md` 下增删改主题级技能（带 YAML frontmatter：`name`、`description` 和 `flashrsi: {algorithm, level, source_task_ids}`）。校验权限、层级、来源必须引用本批合法经验、单技能不超 `max_skill_chars`（2000）且总库不超 `max_library_chars`（48000）——**合法即直接采用为新 `current`，不测 dev、不要求涨分**。
  4. 紧接着从刚采纳的 Topic 结果出发，调 **General Curator** 提炼跨主题通用技能。新增或修订的通用技能必须引用至少 `general_min_tasks`（默认 3）个本批不同来源任务 ID（所以若 `batch_size < general_min_tasks`，永远编不出新通用技能）。General 层校验失败不会回滚已经采纳的 Topic 层。

**和 ACE 放一起看。** ACE 把经验按条目追加在指令文件末尾，每次推理全量塞进上下文；Evo-Harness 把经验编译成带 `name/description` 的原生目录技能文件，靠 Harness 原生发现与按需加载在推理时调用，无需额外优化器。不过要注意：结构校验只能证明来源 ID 齐全，不能保证技能语义一定通用；判断技能到底有没有起作用，得看后续轨迹里 Target 是不是真的加载了技能正文，而不只是扫了一眼描述。

---

### 5.4 第三类：迭代与档案型完整 Harness 搜索（改“整个系统”）

从这里开始，算法不再局限于提示词或技能，全部声明 `evolve_scope = {"*"}`，面向 Target 暴露的全部资源组件（开源 Harness 可改运行代码，闭源 Harness 改暴露出的指令、技能、Hook、MCP 与配置），并且**统一要求具备文件编辑能力（`edit()`）的 Agent Worker**。

#### ADAS：用设计档案探索 Agent 的程序结构

**要解决什么问题。** 只改 Prompt 终究被困在写死的运行循环里。ADAS（Automated Design of Agentic Systems）把视角抬升到“让 Meta Agent 直接发明和改写 Agent 程序结构”，同时用一份不断变厚的设计档案记录“过去试过什么架构、分别考了多少分”，引导下一代设计。

**在 FlashRSI 里怎么转。**

* **数据与状态**：**只用 `dev` 评估**（不使用 `train`）。维护一份评分档案，每条记录是 `(name, thought, fitness)`（设计名、设计思路说明、框架实测 `dev` 均分），外加当前最佳 `best`。
* **一轮流程**：
  1. 完整 `dev` 评 `S0`，以 `name: initial` 入档并设为 `best`。
  2. **每次都检出当前 `best`**，把评分档案与组件规则交给 Worker，请它编辑文件并用 `<NAME>` 和 `<DESIGN>` 写清设计思路。
  3. 默认在**同一个 checkout 目录里再做 `num_reflections = 2` 次评估前复查**——每次都看同一份档案和最新设计说明，直接继续改文件，中间不触发框架 `dev` 评估。
  4. 清理运行时产物后只提交一次。空编辑或非法变更记为拒绝、不进评分档案，但仍消耗一步。
  5. 合法候选测完整 `dev`：无论涨跌都写入评分档案；只有**严格超过 `best`** 才替换下一轮编辑起点和最终胜者。跑完 `max_steps`（默认 10）后一次性 `_promote(best)`。

```text
archive = [(initial, 评估(S0, dev))]
best = S0
重复 max_steps 轮：
    checkout(best)
    Worker.edit(archive, 组件规则)
    再做 num_reflections 次评估前编辑复查
    提交一次合法候选 → score = 评估(candidate, 完整 dev)
    archive.append((name, thought, score))
    if score > best_score: best = candidate
返回 best
```

**边界与成本。** ADAS 的档案里放的是受 `max_evidence_chars`（默认 30000）截断的设计说明与总分，**不主动把历代完整源码和工具轨迹递给 Worker**。每次正常尝试要跑 `1 + num_reflections` 次 Worker 编辑，Target 最多跑 `(1 + S) × D` 次任务。另外要注意仓库文档里坦诚写明的一点：`ShengranHu/ADAS` 的上游 commit 目前尚未固定，本地实现是用原生 Harness 文件树替代原论文固定框架里的 `forward()` DSL，且从单个 `S0` 起步而非多种子架构库。

---

#### Meta-Harness：把历代源码和完整轨迹塞进文件系统，让提案 Agent 自己找证据

**要解决什么问题。** 把几万字的执行轨迹和整份代码库压缩成“分数 + 三行设计摘要”，几乎必然丢掉真正导致失败的底层细节——比如很早一步工具返回里被误截断的字段。Stanford / MIT 的 Meta-Harness 直接打破上下文压缩瓶颈：**不靠外循环替模型拼摘要，而是把历代完整源码、实测分数、决策记录和轨迹原样放进文件系统，让具备读文件、搜索和查日志能力的 coding Worker 自主检索定位问题。**

**在 FlashRSI 里怎么转（重点看它和 ADAS 的根本差异）。**

* **数据与状态**：**只用 `dev` 评估**（不使用 `train`）。保留 `seed`、全部已测候选 `versions`、轨迹记录和 `best`。
* **一轮流程**：
  1. 完整 `dev` 评 `S0`，设为 `seed` 和 `best`。
  2. **关键设计：每一轮都检出最初的 `seed` 作为可编辑目录**（不是像 ADAS 那样检出 `best`！）。
  3. 框架在这个工作目录里临时准备一个只读参考历史目录：
     * `index.json`：所有已测版本的 Snapshot ID、父 ID 和源码路径；
     * `sources/`：导出历代全部已测版本的完整原生源码（包括降分版本）；
     * `trajectories/`：已有的脱敏评估结果、分数、决策记录和此前 Worker 自己的操作轨迹。
  4. 外循环既不指定父版本，也不规定固定交叉/变异算子——Worker 自己决定读哪几版代码、查哪几条失败日志、要不要把 S2 的检索逻辑和 S3 的重试机制拼进当前工作目录。
  5. 提交前，框架把临时历史目录和运行产物统统删掉再校验提交。合法候选跑完整 `dev`，无论升降全加入历史，严格提分才更新 `best`。跑完 `max_steps`（默认 20）后 `_promote(best)`。

**一个极容易踩的坑。** 因为每轮都是从 `seed` checkout 提交的，哪怕 Worker 实际上把历史第 5 版的代码完整抄过来再改了两行，新候选的 `Snapshot.parent_id` 在账面上依然指向 `seed`。所以复盘 Meta-Harness 时，**不能只看 `parent_id` 判断思路来源，必须结合 Worker 的文件读取轨迹和实际 diff**；而且“把历史目录摆在桌面上”也不代表 Worker 真的去读了，同样要查轨迹确认。此外，FlashRSI 本地适配采用单 seed、每轮单候选和单一主指标，未实现原论文的多初始 Harness 与 accuracy/token 多目标前沿。

---

#### AHE：每次改动先做效果预测，再用下一次执行逐题核对

**要解决什么问题。** 面对一个由多文件组成的完整 Harness，改了三处代码、总分涨了 2 个点，到底哪处改动修好了哪道题、又顺手把哪道原本做对的题改坏了？AHE（Agentic Harness Engineering）把**可观测性**和**事前预测核对**引入了演化循环：每次修改必须附带一份 `Change Manifest`，写明每个文件预期修复哪些题、有风险影响哪些题，下一轮实测立刻逐题核对。

**在 FlashRSI 里怎么转。**

* **数据与状态**：**只用 `train` 诊断和评分**（不使用 `dev`，也不保存最后回选用的 dev-best 档案）。持久维护 `current`、其完整 `train` 测量、上一轮 manifest 判决、改动前文件原文（有长度限制，供回滚参考）以及最近 4 次拒绝原因。
* **一轮流程**：
  1. 完整 `train` 评 `S0`。每轮开始前，若当前均分已经达到 `target_score`（默认 `1.0`），直接停止。
  2. 组织证据。默认开启 `use_debugger: true`：先对至多 `debugger_max_tasks`（默认 90，失败优先）道题逐题诊断，再合成一份跨题总览；同时把上一轮判决和回滚原文递给 Worker。
  3. 检出 `current`，由 Worker 编辑文件并在 `<MANIFEST>...</MANIFEST>` 中返回 JSON：
     ```json
     {
       "failure_evidence": ["t1 工具输出被截断导致丢字段"],
       "root_cause": "观察压缩未保留结构化标识字段",
       "targeted_fix": "压缩逻辑保留关键标识字段",
       "changes": [{"path": "runtime/tool_result.py", "fix_tasks": ["t1"], "risk_tasks": ["t2"]}]
     }
     ```
  4. 提交候选后，**把真实 Snapshot diff 路径和 manifest 声明的 `changes[].path` 严丝合缝地对账**。缺字段、JSON 非法、路径不吻合或空编辑统统拒绝，不测候选但消耗一步。
  5. 合法候选在同一个完整 `train` 上重测，按**零分边界**统计逐题成败翻转（`before <= 0 < after` 算 `fail_to_pass`，`before > 0 >= after` 算 `pass_to_fail`；注意 `0.3 → 0.8` 记数值增益，不算一次修复翻转），给每条文件改动打上判决：
     * `EFFECTIVE`：预测修复非空且全部翻转，且无声明风险发生；
     * `PARTIALLY_EFFECTIVE`：部分预测修复翻转，且无声明风险发生；
     * `MIXED`：既有预测修复发生，又触发了声明风险；
     * `HARMFUL`：声明风险发生，但没修好任何预测题；
     * `INEFFECTIVE`：修复与风险都没发生。
       未被任何改动预料到的回归另记为 `unattributed_regressions`。
  6. **最反直觉的一步**：只要协议合法且跑完评估，**无论涨分、同分还是降分甚至判为 `HARMFUL`，都会立刻保留为下一轮的 `current`**！下一轮 Worker 拿着这份判决和旧文件原文，自己决定接着修还是手动撤销。跑完 `max_steps`（默认 10）或达标后，直接交付最后保留的 `current`。

**务必分清本地代码与原论文。** 原论文 v4 算法 1 写有归因后的自动 `Rollback` 和返回历史最佳 `H_best`；但 FlashRSI 本地实现刻意走的是“全部保留、把判决和原文交给下一轮自主修复”的变体，结尾既不自动回滚，也没有回选最高分的 `promote`。同时也要记住，多处改动一起测一次，manifest 判决是对预测的核对，不能等价于单个文件的严格因果消融。

---

### 5.5 第四类：约束、模块与假设树型完整 Harness 演化（防过拟合与长程探索）

前面三种全量编辑方法虽然灵活，但在长程搜索里很容易踩三个坑：**拆东墙补西墙（能力回退）、改一处牵全身（改动失控）、Token 越跑越贵或原地打转（过拟合与失忆）**。下面四种 2026 年的新方法，正好从四个角度对症下药。

#### DarwinX：先保住已有能力再扩展，并从落选分支中重组互补能力

**要解决什么问题。** 模型做题有随机性，一次侥幸答对可能掩盖了真实退步；而且某条分支虽然总分不够高未能准入，但它可能恰好掌握了主分支始终无法解决的一类题。DarwinX 的核心设计就是两条：**“保已有、扩新增”的重复确认契约（Preserve-and-Extend）**，以及**跨分支互补重组**。

**在 FlashRSI 里怎么转。**

* **数据与前提**：**只用 `train`**（任务 ID 唯一，主指标必须在 `[0, 1]`），不使用 `dev`。
* **必须分清的三个集合**：
  1. **已测档案（`archive`）**：跑过筛选或全量确认的候选全在里面，包括被拒的；
  2. **普通变异父池（`accepted`）**：只有初始版本和**通过全量确认**的版本有资格当单父变异起点，也有资格参与最终选优；
  3. **重组候选对**：只要**双方都跑完过全量确认**（`fidelity == "confirmation"`，**不要求 `accepted == True`！**），且各自至少解决了一道对方没解决的题，就有资格配对重组。
* **一轮流程**：
  1. **初始化能力保留集**：把 `S0` 在完整 `train` 上重复跑 `confirm_samples`（默认 3）次，按 **avg@k**（多次均值，不是 pass@k）算逐题均分；达到 `solved_threshold`（默认 `1.0`，即每次重复都拿满分）的题目构成初始 `invariant`，后代沿主父链继承并不断扩大这个底线集合。
  2. **选操作（普通变异 vs. 周期重组）**：普通轮次从已准入父池挑父版本——一半概率（`exploration_probability = 0.5`）均匀探索，另一半按 `(cumulative_gain, score)` 挑最强者。每到第 `merge_every`（默认 3）步，优先寻找未试过的互补重组对（按已解决题并集大小、再按双方较低分排序；高分者为主版本，并把共同祖先与 donor 的临时 checkout 一并给 Worker 参考）。
  3. **两阶段重测门禁**（空编辑、非法修改或与任意已入档文件完全重复的候选直接跳过）：
     * **小样本筛选**：抽 `screen_size`（默认 3）道 `train` 题，**把候选和所有父版本在同一批题上同条件重跑 `screen_samples`（默认 1）次**；只要至少一道题有正向增量且回退质量在 `screen_regression_tolerance`（默认 `1.0`）内，就晋级确认。
     * **全量确认**：把候选和所有父版本在完整 `train` 上再重跑 `confirm_samples` 次。令 $c[t]$ 为候选逐题均分、$p[t]$ 为主父本次重测均分、$r[t]$ 为所有父版本本次重测的逐题最大值：

       ```text
       gain            = sum_t(c[t] - p[t])
       regression_mass = sum_t(max(r[t] - c[t], 0))

       全量确认通过条件：
           gain > 1e-12
           and regression_mass <= regression_tolerance + 1e-12   # 默认容差为 0！
           and 每道继承 invariant 题均有 c[t] >= solved_threshold
       ```

       看清楚默认容差为 0 的威力：哪怕总分暴涨，只要在一道题上从 `0.5` 掉到 `0.4`，`regression_mass = 0.1 > 0` 直接拒收——**不准用别题的涨分抵消任何一题的退步**。重组时参考上限 $r[t]$ 取双方逐题较强者，保留集也并入双方已解决题。
  4. **入档与提炼经验**：所有跑过评估的候选全部入档，并调用一次 `worker.ask()` 写 `<LEARNINGS>` 总结教训。只有通过全量确认的才进入变异父池。
  5. **最终交付**：在已准入成员中取**存档 `train` 均分最高者**（同分取早），结束时不重测旧档案。

**成本账本。** 严谨是要花钱的：普通变异筛选就要跑父子共 $2 \times s \times q$ 次任务，进确认再加 $2 \times N \times k$ 次；双父重组更是 $3 \times s \times q$ 加 $3 \times N \times k$ 次，外加初始确认 $N \times k$。

---

#### ModularRSI：按功能模块分治演化，靠“同题成败对照”和工程门禁推进

**要解决什么问题。** 完整 Harness 的一次失败，往往是循环、工具、观察压缩、上下文管理和停止判断搅在一起造成的；大而全的补丁很容易互相冲突。ModularRSI 走了两条非常独特的路：**按功能模块拆成独立分支各自演化后再集成**，以及**不用总分涨跌做门禁，而用同题对比诊断 + diff review + 执行健康度门禁推动版本前进**。

**在 FlashRSI 里怎么转。**

* **模块边界**：配置中必须显式给出 `modules` 映射——MiniAgent 映射五个运行时模块（`agent_loop / tool_use / observation_management / context_management / task_completion_detection`），Claude Code 则映射三或四个原生扩展模块（`prompt / skill / hook`，可选 `tool: [.mcp.json]`）。多个模块可以映射到同一个文件（例如都指向 `runtime/agent.py`）：**框架强制卡死的是精确文件路径边界**，文件内部“只改属于自己模块的逻辑”靠提示词与后续完整 diff review 约束（不支持 `file.py::func` 语法，也没有 AST 隔离）。
* **第一阶段：各模块从同一个 `seed` 独立演化**（按配置顺序执行，分支之间互不继承中间修改或历史成功记忆）：
  1. 用当前模块分支把一批 `train`（默认 `batch_size = 10`）重复跑 `num_rollouts`（默认 3）次；`finished` 且主指标达到 `success_threshold`（默认 `1.0`）记为成功。
  2. 对每道题按多次跑的结果分成三组：**同题既有成功又有失败（`contrastive`）**、全成功（`positive`）、全失败（`negative`，若本分支以前做对过该题，附上最近一条历史成功轨迹）。同题成败对照让 Worker 能直接定位导致分叉的关键动作，而不是把两道无关的题目硬做对比。
  3. Worker 只围绕当前模块输出 JSON `findings`，同文件建议合并后按支持任务数排序，截取前 `edit_budget`（默认 1）个优先文件目标。
  4. Worker 基于分支目录编辑一次，依次过四道门禁：**路径范围校验 → 修改后 `.py` 编译检查 → Worker 对完整 diff 的严格 review（diff 太大放不进证据预算直接拒，绝不审截断补丁！）→ 抽 `validation_samples`（默认 2）道当前 batch 题各跑一次，要求全部 `finished`**。
  5. **特别注意：这道执行门禁不要求答案正确、不要求涨分、也不要求不降分！**（如果 Claude Code 改了 `.mcp.json`，还会额外校验真实执行轨迹里的 `MCPStatus`：新增/修改的 MCP 服务必须连接成功、发现工具且至少完成一次非错误调用。）过了就推进分支，否则保留旧版。
* **第二阶段：集成与回退**：
  1. 独立分支全部跑完，Worker 在**全新的 `seed` checkout** 里同时查看各分支末版和已接受改动史，通过普通文件编辑手动解决共享文件冲突，完成初始集成。
  2. 初始集成过文件/语法/review 后，在**完整 `train`** 上跑执行检查——通过则设为 `fallback`，失败则 `fallback` 退回 `seed`。
  3. 之后再跑 `integration_epochs`（默认 1）轮对比式 batch 修订（可改所有模块文件并集，仍过 review 和抽样执行门禁）。
  4. 最终修订版必须再过一次**完整 `train` 执行检查**，通过就交付，失败就退回 `fallback`——全程没有按 `train` 或 `dev` 分数给中间版本排名的步骤。

**本地适配边界。** 原论文设计了运行时可发现函数库和 Task-Aware Function Composer；FlashRSI 用原生 Harness 文件和 Worker 集成编辑替代，同时支持在 Terminal-Bench / SWE-bench 上可选配置外部独立演化池（`evolution_data_path`）。

---

#### RRSI：提案端退火控复杂度，选择端卡噪声带与 Token 成本

**要解决什么问题。** 不加约束的 RSI 很容易走向两个极端：一是每轮往代码里塞过多补丁、把 Harness 改得过于复杂只为迎合训练集；二是用翻倍的 Token 消耗换取微弱涨分，性价比极低。Google Cloud AI Research 的 RRSI（Regularized Recursive Self-Improvement）把正则化同时装在了**提案侧**和**选择侧**。

**在 FlashRSI 里怎么转。**

* **数据与前提**：**只用 `train`**（ID 唯一，主指标在 `[0, 1]`），**强制要求 Target 返回完整 Token 用量**（缺用量直接判测量无效），不使用 `dev`。
* **第一步：基线测量与自动噪声校准**。对 `S0` 的完整 `train` 重复跑 `k`（默认 2）次，算出均分 $S$ 和单题平均 Token 成本 $C$（包含缓存 Token 和已报告的内部子 Agent 用量，不含 Worker 调用）。当 `delta` 设为 `None` 时，算法在每道题内部做 2,000 次 bootstrap，按 $\delta = \text{delta\_z} \times \sqrt{2} \times \text{std}$ 标定分数噪声带（多轮基线校准时仍留第一次基线作 `incumbent`，不挑最高分）。
* **第二步：代表轨迹调查与冻结本轮上下文**。挑低分端最差 trial（至多 `n_fail_traces = 22`）和高分端最佳 trial（至多 `n_success_traces = 6`），由 `analyst` 调度 `digester` 读取脱敏轨迹文件完成调查报告；冻结包含历史最高分 $S^*$、$\delta$、编辑史、本轮机制预算和探索/剪枝指令的 `context`。
* **第三步：提案侧正则（兄弟并行、预算退火、探索与剪枝）**。

  * 本轮 $m$（默认 2）个兄弟候选**全部从同一个 `incumbent` 独立检出、读同一份冻结 `context`**，彼此不偷看结果。
  * 第 $t$ 轮（$t = 0 \dots T - 1$）允许声明的独立机制条数按余弦退火收缩（一个机制可跨多文件，限制的是机制数而非文件数）：
    $$
    b(t) = \left\lceil b_{\min} + (b_{\max} - b_{\min}) \cdot \frac{1 + \cos(\pi t / T)}{2} \right\rceil
    $$
  * 连续 `w`（默认 3）轮涨分不超过 $\delta$ 时，最后 `m_draft`（默认 1）个槽强制探索此前没碰过的语义组件；最近 `n_prune`（默认 4）轮没有正向实测收益的组件会被推入剪枝建议（由模型决定是否删除，同样须走审查与测量）。
  * Worker 编辑并用 `<EDITS>` 逐条声明机制，过 `critic` 完整 diff 审查（有异议可在同目录最多修复 `repair_rounds = 5` 轮），再过全 `.py` 编译和 `smoke_n`（默认 2）道代表题 smoke 检查，才放行进全量测量。
* **第四步：选择侧正则——先判“准不准入”，再在合格者里选“最高分”**。
  设 `incumbent` 为 $(S, C)$，候选为 $(S', C')$，相对成本变化 $dC = (C' - C)/C$，分差 $dS = S' - S$，已接受历史最高分为 $S^*$：

  ```text
  if S' < S* - delta:
      拒绝                                         # 跌破历史底线
  elif dS > delta:
      要求 dC <= beta0 + beta1 * dS                # 明确涨分：成本增幅不能太离谱
  else:
      要求 w_s * dS - w_c * dC + w_n * novelty > 0 # 噪声带内：用分数、省成本和新颖性综合算账
  且所有配置的 guards 指标均须通过
  ```

  其中 `novelty` 只奖励此前**获胜编辑**里从未出现过的 `client_tool / skill / memory / subagent` 语义类别，且要结合真实 diff 校验。

**最关键的区分：效用只管准入，最终选胜按分数！** 看一个默认参数（`w_s=100, w_c=15`）下的例子，设 $S = S^* = 0.70, \delta = 0.02, \text{novelty} = 0$：

| 候选 | 实测分$S'$ | 成本变化$dC$ | 带内效用           | 是否准入 | 本轮结果                                                 |
| ---- | ------------ | -------------- | ------------------ | -------- | -------------------------------------------------------- |
| A    | 0.710        | +50%           | `1 - 7.5 = -6.5` | 否       | 分数最高，但成本过高被拒绝（`REJECTED`）               |
| B    | 0.690        | -20%           | `-1 + 3 = +2.0`  | 是       | 合格，但分数输给 C（`LOST`）                           |
| C    | 0.705        | 0%             | `+0.5`           | 是       | **合格者里分数最高，成为本轮胜者（`ACCEPTED`）** |

看明白了吗？B 的效用（+2.0）比 C（+0.5）高，但只要都在合格名单里，RRSI 在兄弟之间按**分数最高**（同分取早）挑胜者，所以选 C；如果本轮没有 C、只有 B 合格，RRSI 就会接受 B——允许在历史底线内用微小的分数让步换取 20% 的成本下降。最后，跑完 $T$（默认 20）轮，**返回最后接纳的 `incumbent`，不恢复历史最高分 $S^*$ 对应的版本**。

---

#### Arbor：用假设树管理长程研究，让每次失败都沉淀成向上传递的洞察

**要解决什么问题。** 长程自改进很像做科研：需要同时开几条研究方向，每条方向下细化具体实验假设；如果把几十次尝试平铺成一条流水日志，上下文很快被淹没，失败实验的教训也容易流失。Arbor（Hypothesis-Tree Refinement）把搜索组织成一棵持久化**假设树**，并把“研究思路的父子关系”和“代码的实际版本主干”彻底解耦。

**在 FlashRSI 里怎么转。**

* **数据与角色**：要求 `train`、`dev` 非空，且 `train / dev / test` 任务 ID 唯一且互不重合；基线在 `train` 和 `dev` 上必须全部 `finished`，否则直接中止。Coordinator（规划者）、Executor（实现者）、反思者和向上综合者，实际上都是发给**同一个 Agent Worker** 的不同提示角色。
* **两套必须分开的状态**：
  * **假设树（概念层）**：`ROOT`（深度 0）→ 研究方向 `direction`（深度 1）→ 可执行实验叶（深度 $\ge 2$，默认 `max_depth = 3`）。每个节点记录假设、状态（`pending / running / done / merged / needs_retry / pruned`）和文字 `insight`。
  * **当前主干 `trunk`（代码层）**：当前已采纳的完整 Snapshot 及其保存的 `train / dev` 分数。
* **最容易搞混的关键点**：**树上的 `parent_id`（思路父节点）和代码上的 `base_snapshot_id`（实际检出基线）是两回事！** 哪怕一个新实验是挂在某个被拒实验节点下的细化假设，Executor 动手改代码时检出的永远是**当前的 `trunk`**，而不是那个失败实验的 Snapshot。
* **一轮流程**：
  1. **Coordinator 规划**：读取假设树、当前 `trunk` 文件清单与 `train` 轨迹、保存的 `train / dev` 分数以及祖先 `insight`，返回 `<ARBOR>` JSON 执行 `expand`（在 `root / direction / done / merged` 节点下扩出 1–3 个假设）、`run`（派发深度 $\ge 2$ 的 `pending` 叶节点）、`prune`（剪掉非 ROOT 子树，保留记录但不影响已采纳 `trunk`）或 `stop`。每次派发前最多调 Coordinator 12 次。
  2. **Executor 实现**：从当前 `trunk` checkout 写代码并提交（无独立编译/diff review 门禁；空或非法编辑标 `needs_retry`，不自动重试）。
  3. **train / dev 双门禁采纳**：

     ```text
     train_ok = 候选全部 train 轨迹 finished
                and candidate_train - trunk_train >= merge_threshold   # 默认 0.0
     dev_ok   = 候选全部 dev 轨迹 finished
                and candidate_dev > trunk_dev                          # 严格大于！
     if train_ok and dev_ok:
         trunk = candidate   # 节点标 merged（指被采纳，并非 Git 三方合并）
     ```

     未过 `train` 筛选就跳过 `dev`；任一轨迹未完成标 `needs_retry`；`dev` 同分或退步标 `done`，保留旧 `trunk`。
  4. **文字经验向上综合（核心特色）**：**无论本次实验采纳、失败还是非法编辑**，先由 Worker 写下本节点 `insight`（反思看 `train` 轨迹与可能存在的 `dev` 分数，不看原始 `dev` 轨迹），然后**从下往上沿父、祖父一路综合到 `ROOT`**——每层读取该祖先下所有有 `insight` 的直接子节点（包括被拒和被剪枝的），把局部试错提炼成高层研究结论。这里完全是文本综合，没有 MCTS / UCB 的数值回传公式。
  5. 跑完 `max_cycles`（默认 10）次派发或 Coordinator 主动 `stop` 后，直接 `_promote(trunk)`，不做全树重排。

---

## 参考文献

Agrawal, L. A., Tan, S., Soylu, D., Ziems, N., Khare, R., Opsahl-Ong, K., Singhvi, A., Shandilya, H., Ryan, M. J., Jiang, M., Potts, C., Sen, K., Dimakis, A. G., Stoica, I., Klein, D., Zaharia, M., & Khattab, O. (2025). *GEPA: Reflective prompt evolution can outperform reinforcement learning*. arXiv. https://arxiv.org/abs/2507.19457

Guo, Q., Wang, R., Guo, J., Li, B., Song, K., Tan, X., Liu, G., Bian, J., & Yang, Y. (2023). *EvoPrompt: Connecting LLMs with evolutionary algorithms yields powerful prompt optimizers*. arXiv. https://arxiv.org/abs/2309.08532

Hu, S., Lu, C., & Clune, J. (2024). *Automated design of agentic systems*. arXiv. https://arxiv.org/abs/2408.08435

Jin, J., Hu, Y., Qiu, K., Dai, Q., Luo, C., Dong, G., Li, X., Zhao, T., Ma, X., Zhang, G., Wu, Z., Liu, B., Yang, Z., Li, L., Wang, L., Qian, H., Zhu, Y., & Dou, Z. (2026). *Toward generalist autonomous research via hypothesis-tree refinement*. arXiv. https://arxiv.org/abs/2606.11926

Jin, J., Li, Y., Hu, Y., Dong, G., Li, X., Yang, X., Dou, Z., & Zhu, Y. (2026). *FlashRSI: A unified framework for agent self-improvement* (Commit 560f02d) [Computer software]. GitHub. https://github.com/RUC-NLPIR/FlashRSI

Lee, Y., Nair, R., Zhang, Q., Lee, K., Khattab, O., & Finn, C. (2026). *Meta-Harness: End-to-end optimization of model harnesses*. arXiv. https://arxiv.org/abs/2603.28052

Li, G., Fan, M., Liu, Y., Zhang, S., Fan, J., Wang, S., Hou, J., Weng, X., Tian, H., & Li, Z. (2026). *SkillAdam: Stable and efficient skill evolution for agents*. arXiv. https://arxiv.org/abs/2609.08944

Lin, J., Liu, S., Pan, C., Lin, L., Dou, S., Xi, Z., Huang, X., Yan, H., Han, Z., & Gui, T. (2026). *Agentic harness engineering: Observability-driven automatic evolution of coding-agent harnesses*. arXiv. https://arxiv.org/abs/2604.25850

Lin, M., Wu, J., Wang, Z., Shi, Z., Sang, Y., He, B., Liu, Z., Wei, T., Wu, Z., Zhang, Z., Wang, D., Zhang, X., Dumoulin, B., Xie, C., Zhou, Y., Wang, S., & Lu, H. (2026). *Harness updating is not harness benefit: Disentangling evolution capabilities in self-evolving LLM agents*. arXiv. https://arxiv.org/abs/2605.30621

Liu, S., Agarwal, S., Maheswaran, M., Cemri, M., Li, Z., Mang, Q., Naren, A., Boneh, E., Cheng, A., Pan, M. Z., Du, A., Keutzer, K., Cheung, A., Dimakis, A. G., Sen, K., Zaharia, M., & Stoica, I. (2026). *EvoX: Meta-evolution for automated discovery*. arXiv. https://arxiv.org/abs/2602.23413

Novikov, A., Vũ, N., Eisenberger, M., Dupont, E., Huang, P.-S., Wagner, A. Z., Shirobokov, S., Kozlovskii, B., Ruiz, F. J. R., Mehrabian, A., Kumar, M. P., See, A., Chaudhuri, S., Holland, G., Davies, A., Nowozin, S., Kohli, P., & Balog, M. (2025). *AlphaEvolve: A coding agent for scientific and algorithmic discovery*. arXiv. https://arxiv.org/abs/2506.13131

Shinn, N., Cassano, F., Berman, E., Gopinath, A., Narasimhan, K., & Yao, S. (2023). *Reflexion: Language agents with verbal reinforcement learning*. arXiv. https://arxiv.org/abs/2303.11366

Wang, Z., Ge, Y., Wang, Z., Wang, H., Xu, Y., Frandsen, J., Fu, S., Wilson, A. C., & Reddy, C. K. (2026). *VERSE: Verified self-evolving optimizer for agent harnesses*. arXiv. https://arxiv.org/abs/2610.02616

Wei, T., Shi, Z., Lin, M., He, B., Liu, Z., Sang, Y., Bei, Y., Ning, X., Zou, J., Li, T.-W., Lin, X., Zhao, Y., Wang, C., Dumoulin, B., Wang, D., He, J., & Lu, H. (2026). *Evo-Harness: Context-to-harness skill compilation for self-evolving agents*. arXiv. https://arxiv.org/abs/2608.15071

Weng, L. (2026, July 4). *Harness engineering for self-improvement*. Lil'Log. https://lilianweng.github.io/posts/2026-07-04-harness/

Wu, S., Ren, J., Li, Y., Li, H.-S., Yang, C., Ma, X., Zhang, Y., Gu, W., Yang, J., Batista-Navarro, R., Zhang, C., Liu, X., Zhou, M., Dai, B., & Lin, C. (2026). *ModularRSI: Modular and generalizable recursive harness self-improvement*. arXiv. https://arxiv.org/abs/2609.14857

Xia, P., Han, R., Wang, Z., Chen, Y., Zhuang, Y., Lee, Y., Huang, C., Yu, H., CuiZhu, Z., Ming, Y., Yao, H., Gokturk, B., Pfister, T., & Lee, C.-Y. (2026). *RRSI: Regularized recursive self-improvement of agent harnesses*. arXiv. https://arxiv.org/abs/2609.24972

Xu, R., Chen, Y., CuiZhu, Z., Mishra, B. D., Ming, Y., Yu, H., Han, R., Lee, C.-Y., & Pfister, T. (2026). *Harness²: Recursive agent harnessing for an open world*. arXiv. https://arxiv.org/abs/2610.14224

Yang, C., Wang, X., Lu, Y., Liu, H., Le, Q. V., Zhou, D., & Chen, X. (2023). *Large language models as optimizers*. arXiv. https://arxiv.org/abs/2309.03409

Yang, Y., Gong, Z., Huang, W., Yang, Q., Zhou, Z., Huang, Z., Li, Y., Gao, X., Dai, Q., Liu, B., Qiu, K., Yang, Y., Chen, D., Yang, X., & Luo, C. (2026). *SkillOpt: Executive strategy for self-evolving agent skills*. arXiv. https://arxiv.org/abs/2605.23904

Yu, X., Xia, F., Pan, J., Tang, Y., Liu, C., Yu, A., Jiang, S., Chen, Y., Xue, L., & Yu, T. (2026). *Self-improvement memory enables continual recursive self-improvement*. arXiv. https://arxiv.org/abs/2611.18856

Yuksekgonul, M., Bianchi, F., Boen, J., Liu, S., Huang, Z., Guestrin, C., & Zou, J. (2024). *TextGrad: Automatic "differentiation" via text*. arXiv. https://arxiv.org/abs/2406.07496

Zelikman, E., Lorch, E., Mackey, L., & Kalai, A. T. (2023). *Self-Taught Optimizer (STOP): Recursively self-improving code generation*. arXiv. https://arxiv.org/abs/2310.02304

Zhang, H., Zhang, S., Li, K., Zhang, C., Chen, Y., Zhang, Y., Bai, L., & Hu, S. (2026). *Self-Harness: Harnesses that improve themselves*. arXiv. https://arxiv.org/abs/2606.09498

Zhang, J., Hu, S., Lu, C., Lange, R. T., & Clune, J. (2025). *Darwin Gödel Machine: Open-ended evolution of self-improving agents*. arXiv. https://arxiv.org/abs/2505.22954

Zhang, J., Zhao, B., Yang, W., Foerster, J., Jiang, M., Devlin, S., & Shavrina, T. (2026). *Hyperagents: Recursive metacognitive self-improvement*. arXiv. https://arxiv.org/abs/2603.19461

Zhang, Q., Hu, C., Upasani, S., Ma, B., Hong, F., Kamanuru, V., Rainton, J., Wu, C., Ji, M., Li, H., Thakker, U., Zou, J., & Olukotun, K. (2025). *Agentic context engineering: Evolving contexts for self-improving language models*. arXiv. https://arxiv.org/abs/2510.04618

Zhang, Y., Dai, Y., Tan, J., Yang, L., Mullur, R., Hoang, T., Hu, Z., Zhu, J., Mui, P., Savarese, S., Xu, R., & Chen, Z. (2026). *DarwinX: Evolving agent harnesses through natural selection*. arXiv. https://arxiv.org/abs/2608.07545

坤虫. (2026, November 1). *大灌水时代下自进化（RSI）的论文梳理与思考*. 知乎专栏. https://zhuanlan.zhihu.com/p/672502477

周星星. (2026). *什么是 Self-evolving / self-improving / RSI？一篇文章搞懂自进化*. 知乎专栏. https://zhuanlan.zhihu.com/p/2065227313973825752
