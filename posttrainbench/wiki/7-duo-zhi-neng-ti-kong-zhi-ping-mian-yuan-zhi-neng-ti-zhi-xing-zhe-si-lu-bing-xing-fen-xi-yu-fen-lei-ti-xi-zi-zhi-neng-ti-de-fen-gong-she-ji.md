AutoSFT 的控制平面不是单个智能体的上下文堆叠，而是一个**刻意分层的三级委托体系**：唯一的元智能体持有决策权，一个机械化的执行者完成从数据物化到评测收集的完整轮次，四路只读分析者并行产出结构化反馈，而分类体系子智能体内部再嵌套四个治理角色（设计者、验证器作者、红队评审员、降级裁决员）。本文解析这一分工的静态拓扑、动态协作协议，以及支撑它的持久化与恢复基础设施——即"谁拥有什么、为何分离、如何协作"的完整设计逻辑。

> **📖 小白导读**
> - 这篇讲什么：多智能体的"组织架构图"——一个做决策的元智能体、一个干活的执行者、四路只读分析者、外加分类体系内部的四个治理角色，它们如何分工协作。
> - 读之前先知道：**元智能体（meta-agent）**＝顶层决策者，决定"下一步做什么实验"；**执行者（executor）**＝被派活的子智能体，只做机械执行不读分数；**只读分析**＝只能看文件、不能改动任何东西的观察角色；**委托（spawn）**＝父智能体拉起一个子智能体会话的动作。
> - 读完你能明白：为什么要把"决策、执行、观察、治理"四权分立；每个角色的输入输出边界在哪；这套分工如何防止一个智能体"既当运动员又当裁判"。
> - 提醒：本页是理解后续所有页的骨架，建议精读一遍。

## 设计第一性原则：决策、执行、观察、治理的分离

整个控制平面建立在一个核心判断上：**一个同时持有决策权、执行权与解释权的智能体，会系统性地把自己的上下文噪声变成研究结论**。AutoSFT 用三道分离切断这条链：

第一道是**决策与执行分离**。元智能体选择数据动作并写入不可变计划，但从不提交、轮询或收集任何 Cluster 作业；执行者机械物化该计划并跑完 SFT 到评测的固定流水线，但被明确禁止打开或解读 `metrics.json`、`scores.json` 等任何指标文件——它的报告只包含终态与类型化的 `result_artifacts` 路径，不含任何指标数值或数据建议。

第二道是**观察与决策分离**。四路分析智能体是只读的：各自检查失败案例、数据供给、学习动力学与实验历史，产出零到三条可证伪的建议，但"建议是建议而非决策"——元智能体可以拒绝全部建议并记录 `meta-originated`。

第三道是**治理与构建分离**。分类体系子智能体内部，决定"分类体系必须覆盖什么"的设计者被刻意隔离于"实现代价"信息之外——它的 TOML 定义明文写有 "You are not told what anything costs to build, and you do not ask"；`AGENTS.md` 解释了原因：一个同时画树和写验证器的智能体会在画树时就开始裁剪，"nothing in the artifacts records that a choice was made"。

Sources: [AGENTS.md](AGENTS.md#L27-L67)
Sources: [autosft_executor.toml](.codex/agents/autosft_executor.toml#L60-L68)
Sources: [autosft_taxonomy_designer.toml](.codex/agents/autosft_taxonomy_designer.toml#L1-L14)
Sources: [AGENTS.md](AGENTS.md#L163-L170)

## 分层委托拓扑：深度 3、线程 6 的有界图

整个拓扑由 `.codex/config.toml` 定界：`max_threads = 6` 容纳"一个执行者 + 最多四个并行分析线程 + 一个串行分类体系任务"的并发上限；`max_depth = 3` 的存在则**只为分类体系任务服务**——根元智能体（深度 1）派生 `autosft_capability_taxonomy_verifier`（深度 2），后者再派生其四个固定子角色（深度 3）。除这一条链外，任何嵌套派生都被角色契约禁止：执行者与分析智能体不得派生智能体（"Executor and analysis subagents do not spawn agents"），而分类体系子智能体只被允许派生它名下的四个。

```mermaid
flowchart TD
    subgraph D0["深度 0：非 LLM 外层运行器（Python）"]
        RUNNER["agent_task.py 重派循环<br/>分数门禁 · 分类作业接管 · 根会话恢复"]
    end
    subgraph D1["深度 1：根元智能体（Codex exec 根线程）"]
        META["autosft meta-agent<br/>数据决策 · 编排 · 台账记录"]
    end
    subgraph D2["深度 2：直接子智能体"]
        EXEC["autosft_executor<br/>（机械化执行）"]
        FCS["autosft_failure_case_study"]
        DE["autosft_data_examine"]
        EFA["autosft_evaluation_feedback_analysis"]
        HIS["autosft_history"]
        TAX["autosft_capability_taxonomy_verifier<br/>（可写 · 编排发布）"]
    end
    subgraph D3["深度 3：仅分类体系任务可派生"]
        DES["autosft_taxonomy_designer<br/>冻结覆盖契约"]
        AUTH["autosft_verifier_author<br/>构建确定性验证器"]
        RED["autosft_verifier_red_team<br/>只读红队攻击"]
        ADJ["autosft_demotion_adjudicator<br/>逐叶降级裁决"]
    end
    RUNNER -->|"启动/续跑/重派"| META
    META -->|"不可变计划 + executor_handoff.json"| EXEC
    META -->|"并行 · typed result_artifacts"| FCS
    META -->|""| DE
    META -->|""| EFA
    META -->|""| HIS
    META -->|"串行 · job_type=refine"| TAX
    TAX -->|"串行批次"| DES
    TAX -->|"分批构建"| AUTH
    TAX -->|"验证器变更时一次"| RED
    TAX -->|"每个提案一次"| ADJ
    style D0 fill:#eef4fb,stroke:#5b8ac6
    style D3 fill:#f7eef4,stroke:#a05b8a
```

阅读此图需先理解一个前提：**连线表示的是契约授权而非运行时必然**。元智能体在一轮中先派生执行者并等待其终态，随后**并行**启动四个分析者，全部报告持久化后才**串行**启动分类体系任务——"This serial placement keeps `max_threads=6`"这一行注释揭示了 6 个线程的配额由来：1（元）+ 1（执行者已关闭）+ 4（分析）在峰值时刻恰好占满，分类体系任务必须等分析线程关闭后才能启动。

Sources: [config.toml](.codex/config.toml#L1-L16)
Sources: [agent_prompt_online.md](tools/codex_exec/agent_prompt_online.md#L100-L112)
Sources: [AGENTS.md](AGENTS.md#L146-L149)

## 角色所有权矩阵：十一个智能体的职责、可读面与权限边界

| 智能体 | 层级 | 权限 | 可读技能 | 核心产出 | 绝对禁止 |
|---|---|---|---|---|---|
| 根元智能体 | 1 | 决策+编排 | `huggingface-dataset-discovery`、`sft-data-recipe-authoring` | 数据动作计划、台账结构化决策 | 执行任何作业、解读详细损失/指标工件 |
| `autosft_executor` | 2 | 机械执行 | 5 个执行类技能 | 终态 + 类型化 `result_artifacts` + `retry_history` | 派生智能体、读指标文件、改训练配置 |
| `autosft_failure_case_study` | 2 | 只读分析 | `failure-case-study-analysis` | 行为失败成因报告（0–3 建议） | 编辑数据、写配方、建跨轮实验树 |
| `autosft_data_examine` | 2 | 只读分析 | `training-data-examination` | 供给侧质量/多样性/污染审计 | 诊断模型失败、解读损失曲线 |
| `autosft_evaluation_feedback_analysis` | 2 | 只读分析 | `learning-dynamics-interference-analysis` | 难度/饱和/干扰分析 | 诊断个别失败案例 |
| `autosft_history` | 2 | 只读分析 | `history-round-analysis` | 分支保留的假设/动作实验树 | 重新检查原始训练行或单条预测 |
| `autosft_capability_taxonomy_verifier` | 2 | 有限可写 | `taxonomy-verifier-implementation` 系列 | 冻结契约、发布版本、分类请求 | 写配方、选数据、自行裁决降级 |
| `autosft_taxonomy_designer` | 3 | 有限可写 | `taxonomy-design` 系列 | 冻结契约 + 分类树 + 证据溯源 | 读任何实现代价信息、写验证器 |
| `autosft_verifier_author` | 3 | 有限可写 | `taxonomy-verifier-implementation`（实现半） | 验证器模块 + 金样例 + 束 | 重开设计决策、自行发布 |
| `autosft_verifier_red_team` | 3 | 只读 | 无（内嵌指令） | `taxonomy_work/red_team/findings.json` | 编辑验证器、写金样例 |
| `autosft_demotion_adjudicator` | 3 | 只读 | 无（内嵌指令） | 单叶 `approve/reject` 判决 | 写任何文件、批量裁决 |

矩阵中三处设计值得注意。其一，**技能单一所有权是硬约束**：`skills/README.md` 规定"Each skill has exactly one owning agent role"，且技能从不互相交接控制——排序权完全归于根编排提示。其二，两个深度 3 评审角色（红队、裁决员）**不读任何技能文件**，其全部指令内嵌于 TOML 定义，保证了它们与技能库作者之间的信息隔离。其三，执行者的 `web_search = "disabled"` 与根元智能体在线模式下可用的 `--search` 形成对照——检索权限只配给需要发现数据的那一层。

Sources: [README.md](skills/README.md#L1-L5)
Sources: [autosft_executor.toml](.codex/agents/autosft_executor.toml#L1-L33)
Sources: [agent_prompt_online.md](tools/codex_exec/agent_prompt_online.md#L11-L21)

## 元智能体：单一决策点与结构化决策记录

元智能体的权威来自两个约束的乘积。**正向**：它是唯一有权选择 `action_type`、撰写不可变数据动作计划、以及在台账中记录结构化决策的角色。**反向**：它的每个可选输入都被刻意限流——只读两个元技能，不读执行或分析技能，"Do not submit, poll, stop, convert, or evaluate jobs yourself"。

决策协议本身被 `AGENTS.md` 规定为四步：先读全部建议；选定**恰好一个**数据 `action_type`；然后选中一或多个 `executable_now=true` 且 `action_type` 匹配的建议（或全不选、记录 `meta-originated`）；最后通过 `ledger note` 的结构化旗标记录 `selected_recommendation_ids`、主假设、计划动作、预期效果、可证伪条件与偏差。"Prefer changing one causally interpretable data variable"——若多变量必须捆绑，须记录为何不可分离。这条纪律服务于一个下游消费者：历史分析者靠这些 id 区分 tested/untried/supported/falsified/mixed/inconclusive/confounded 分支，使未来决策可以回溯到更早的兄弟分支。

元智能体还被要求在关闭"看似卡住"的线程之前先检查磁盘："Silence is not evidence of no progress; the work area is"。分类体系设计者在返回任何结果之前就已把契约冻结在 `taxonomy_work/` 下——关闭一个安静但已产出工件的线程等于丢弃一个已做出的决定并为此付出双倍代价。

Sources: [AGENTS.md](AGENTS.md#L127-L145)
Sources: [agent_prompt_online.md](tools/codex_exec/agent_prompt_online.md#L86-L99)
Sources: [AGENTS.md](AGENTS.md#L142-L145)

## 执行者：由交接快照驱动的有限状态机

执行者的智能体定义将其定位为"机械物化元撰写的计划"的工人，而其真实的工作模型是一个**外置的有限状态机**。每次派生前，元智能体用 `python -m src.control_plane.executor_handoff refresh` 刷新 `${AUTO_SFT_RUN_DIR}/executor_handoff.json`——该快照由非 LLM 代码从权威运行工件（`sft_submissions.jsonl`、`cluster_events.jsonl`、`eval_final_status.json`、转换清单）推导出当前 `stage`、`next_action`、`skills_required` 与 `do_not_resubmit_tasks`。

`executor_handoff.py` 中的阶段推导是一段纯函数式的 if/elif 链，覆盖九个终态阶段（`data_materialize → sft_submit → sft_poll → sft_collect → checkpoint_conversion → eval_submit → eval_poll → eval_collect → complete`，外加 `blocked`）。执行者契约要求它只消费快照中的 `next_action`、只读列出的 `skills_required`、绝不轮询已终态的任务、绝不重新提交 `do_not_resubmit_tasks` 中的身份，"Do not scan the whole run tree, rebuild the end-to-end plan"。

这带来一个关键的架构性质：**执行者的可替换性**。`AGENTS.md` 规定元智能体在整个轮次内保持同一执行者线程——"a normal polling interval never creates a new session. Only a thread that actually ended or was lost may be replaced, and the replacement starts from the same handoff"。线程身份不重要，快照状态才重要；替换执行者不是重启轮次，而是让新线程从同一交接点继续。

Sources: [executor_handoff.py](src/control_plane/executor_handoff.py#L521-L594)
Sources: [autosft_executor.toml](.codex/agents/autosft_executor.toml#L45-L56)
Sources: [AGENTS.md](AGENTS.md#L107-L115)

## 四路并行分析：非重叠问题域与共享建议语义

四个分析者的分工不是按流水线阶段切分，而是按**互斥的问题域**切分——`AGENTS.md` 原文列举为：behavioral failure cause and repair（失败案例）、training-data supply（数据供给）、learning difficulty/saturation/interference（学习动力学）、branch-preserving historical evidence（分支保留历史）。这种切分的价值在于并行时的证据独立性：四者读同一轮的 `result_artifacts` 但各自盯住不同的工件子集。

| 角色 | 输入焦点 | 视角问题 | 明确不看的 |
|---|---|---|---|
| `failure_case_study` | 聚合指标、完整失败分解、具体预测 | 漏检 vs 执行/格式失败；单约束 vs 组合 | 原始训练行、训练曲线、跨轮实验树 |
| `data_examine` | 物化行、清单、仅提示的基准参考 | 有效多样性、语义重复、正确性、污染、来源主导 | 模型失败案例、损失曲线 |
| `evaluation_feedback_analysis` | 损失摘要/曲线、暴露清单、验证切片、可比分数史 | 太易/难但可学/不可学/饱和；跨类目干扰 | 个别失败案例 |
| `history` | 台账、配方史、全部既往建议报告 | 假设/动作树的分支保留与混杂检测 | 原始训练行、单条预测 |

统一约束：每份报告含零到三条建议，"zero is correct when the evidence is insufficient"；建议 `action_type` 使用六种规范定义（`filter/remove/reweight/rewrite/synthesize/replay`），其中 `synthesize` 在当前无合成数据政策下**必须** `executable_now=false`——这个约束被固化到 `analysis_reports.py` 的验证逻辑中，违反即拒绝持久化。

每个分析者在返回前必须自行持久化报告（`analysis_reports persist --angle ...`），且该命令做**原子写**：临时文件 + `os.replace`，因为"the four analysis subagents persist in parallel, so a half-written report must never be observable to the meta-agent's completeness check"。元智能体则用 `analysis_reports status` 校验完整性而非代为持久化——校验逻辑扫描目录中每个 `round_*` 文件夹，发现"有部分报告但不足四份"的轮次即报 `missing_angles`。

Sources: [AGENTS.md](AGENTS.md#L54-L67)
Sources: [AGENTS.md](AGENTS.md#L69-L91)
Sources: [analysis_reports.py](src/control_plane/analysis_reports.py#L18-L65)
Sources: [analysis_reports.py](src/control_plane/analysis_reports.py#L419-L497)
Sources: [autosft_data_examine.toml](.codex/agents/autosft_data_examine.toml#L22-L36)

## 分类体系子智能体：深度 3 的内部治理结构

`autosft_capability_taxonomy_verifier` 是唯一可写且可派生的子智能体，可写范围限定于 `${AUTO_SFT_RUN_DIR}/taxonomy/`、`taxonomy_work/` 与分类作业目录。它的四个子角色构成一条治理流水线：

**设计者**先行。它写三份协调工件（`taxonomy_contract.json`、`taxonomy_tree.json`、`source_evidence.json`），契约中每个 scope unit 冻结一个 `include/exclude/merge` 决定。关键隔离在于：父任务派生设计者时**不传任何实现代价信息**——"no verifier module contract, package list, validator/preflight output, or account of laborious leaves"。理由写在 `AGENTS.md`：如果 `args_schema` 在实现期才写，删掉一个枚举值就等于悄悄收窄了分类体系本身，而"nothing anywhere records it"。

**验证器作者**按批次构建。父任务把计划活跃的叶子按 id 最深公共前缀分组、每批 10–15 叶串行派生——"Do not order or size batches by apparent implementation difficulty"防止批次划分本身泄漏代价信号。构建批次只跑逐文件门（`validate-verifier` + 金样例），禁跑全束门，因为全束门"say nothing true about a subset"。

**红队评审员**独立攻击。当验证器代码变更时，父任务在收尾批次前派生恰好一个只读评审员，向每个变更过的验证器双向进攻（false accepts 与 false rejects 并重），产出 `taxonomy_work/red_team/findings.json`。发布门禁机械校验该文件的 `attacked_verifier_ids` 覆盖全部变更集——"a review that lives only in your final message does not exist as far as publication is concerned"。

**降级裁决员**逐叶裁决。作者可以提出"某计划活跃叶子无法保持活跃"的提案（含精确类目/单元、障碍、失败 id、证据指纹），但提案本身不是许可——只有裁决员对该**单个**提案独立返回 `approve`，且五个字段完全匹配时，父任务才把判决原文交还作者机械执行。单元级批量提案、作者单方批准、被拒决定，均无效。守卫运行中发布还额外校验携带精确 `proposal_id` 的机器写入 `spawn_agent` 事件，防止仅凭 JSON 身份字段冒充裁决证据。

Sources: [autosft_capability_taxonomy_verifier.toml](.codex/agents/autosft_capability_taxonomy_verifier.toml#L5-L56)
Sources: [autosft_taxonomy_designer.toml](.codex/agents/autosft_taxonomy_designer.toml#L15-L34)
Sources: [autosft_verifier_author.toml](.codex/agents/autosft_verifier_author.toml#L18-L34)
Sources: [autosft_verifier_red_team.toml](.codex/agents/autosft_verifier_red_team.toml#L16-L44)
Sources: [AGENTS.md](AGENTS.md#L172-L187)

## 持久化契约：每个智能体为自己的产物负责

控制平面的容错基石是一条简单的所有权规则：**谁产出，谁持久化，谁的字段名，谁的位置**。四个分析者各自把报告写入 `analysis_reports/round_<id>/<angle>.json`（四个角名由 `EXPECTED_ANGLES` 集合硬编码）；执行者把自己的报告写入 `executor_reports/round_<id>.json`——这个独立目录的存在理由记录在 docstring 中：`analysis_reports/` 对另外两个消费者（完整性门与台账分类谱系记录）意味着"四个分析角"，执行者报告既非角也非谱系输入，"so it gets its own directory rather than widening either"。

持久化命令同时是**验证门**：`validate_report` 检查角名与轮 id 一致、summary 非空、`key_findings` 带 confidence、建议 id 必须以 `<round_id>:<angle>:` 开头且不重复、`action_type` 属于合法集、`synthesize` 不可即时执行、`outcome` 与 `actual_effect` 的配对约束（`untested` 必须为 null，非 `untested` 必须非空）。历史角色还有专属的 `experiment_tree` 结构校验：节点 id 唯一、父引用必须指向已知节点、`active_branch`/`untried_branches` 只能引用存在的节点。

Sources: [analysis_reports.py](src/control_plane/analysis_reports.py#L301-L349)
Sources: [analysis_reports.py](src/control_plane/analysis_reports.py#L84-L183)
Sources: [analysis_reports.py](src/control_plane/analysis_reports.py#L186-L298)
Sources: [autosft_executor.toml](.codex/agents/autosft_executor.toml#L138-L147)

## 外层非 LLM 运行器：控制平面之上的仲裁层

整个多智能体体系运行在一个**确定性 Python 循环**（`src/control_plane/agent_task.py`）之内，它补充了 LLM 智能体无法可靠完成的三件事：

**分数门禁与重派**。启动链（`commit.sh` → `agent_runner.sh` → `agent_task.py`）从 `AUTO_SFT_TARGET_SCORES` 解析停止条件，运行器在每次尝试结束后用 `ledger best-scores` 对照阈值——目标未达且未 `finalize` 就重派，退出码 3（策略守卫违规）不视为完成。四类重试延迟被区分对待：分类作业结束立即重派（0 秒）、目标已达进入 finalize（短延迟）、策略违规、以及"跑了较久后中断"的瞬态故障（45 秒而非完整退避），并配有 10 连续快速失败（<5 秒）的永久性卡死检测。

**长分类作业的接管**。在本地动态模式下，一旦检测到持久分类请求被提交，运行器**主动终止当前 Codex 进程组**（"A durable classify_pool request was submitted; ending the Codex attempt so the non-LLM runner can take ownership"），在进程组之外以独立会话运行分类 worker，完成后启动新的元智能体尝试并注入结果通知。这解决了"长作业与 LLM 会话超时互斥"的问题——Codex 从不运行、轮询或等待长构建器。

**跨尝试的根会话恢复与孤儿检测**。每个运行持有独立的 `codex_home`（从机器级 home 种子化认证与配置），运行器用 `_latest_root_session_id` 从 rollout 文件中挑选**无 `source.subagent` 标记的最新根会话**来续跑——注释解释了为何不用 `resume --last`：子智能体线程有自己的 rollout，且 taxonomy 任务的设计者往往更新，`--last` 会续错对话。同时运行器在每次尝试结束时检测两类孤儿状态：四角不全的分析轮（提示下一次尝试修复而非绕过），以及"设计已冻结但发布未完成"的孤儿契约——后者会被注入续跑提示，要求基于已有冻结契约构建，"rather than decide coverage again"。

Sources: [agent_task.py](src/control_plane/agent_task.py#L962-L987)
Sources: [agent_task.py](src/control_plane/agent_task.py#L781-L793)
Sources: [agent_task.py](src/control_plane/agent_task.py#L596-L654)
Sources: [codex_home.py](src/control_plane/codex_home.py#L19-L44)
Sources: [commit.sh](src/commit_utils/commit.sh#L91-L128)

## 启动链与信任引导

多智能体拓扑不是运行时协商的结果，而是**启动前静态校验**的结果。`run_codex.sh` 在启动 Codex 前逐一验证：`multi_agent` 特性已启用、`.codex/config.toml` 存在、**十个必需的智能体 TOML 定义全部就位**（缺失任何一个直接退出），并通过 `PreToolUse` 钩子把每次 shell/edit 类工具调用导入 `policy_hook.py` 做策略拦截。这保证了角色边界不依赖智能体的自律，而依赖外层基础设施的强制。

元智能体的提示词本身（`agent_prompt_online.md`，在线模式；`agent_prompt_local.md`，本地模式）由 `default_prompt()` 按数据模式选择，外层运行器在其后追加动态的续跑段——包含分数状态、执行者交接快照路径、分类作业结果、未完成分析轮与孤儿契约的处置指令。因此元智能体的每一轮认知都由"静态角色契约 + 动态磁盘真值"两部分构成，后者始终以权威文件为准。

Sources: [run_codex.sh](tools/codex_exec/run_codex.sh#L39-L87)
Sources: [agent_task.py](src/control_plane/agent_task.py#L333-L390)

## 延伸阅读

本文聚焦控制平面的**静态分工**；各子系统的深度实现与下游消费方由以下页面承接：

- 执行者交接快照的恢复语义与防重复提交注册表：[执行者交接与容错恢复：executor_handoff 紧凑状态、断点续跑与防重复提交注册表](25-zhi-xing-zhe-jiao-jie-yu-rong-cuo-hui-fu-executor_handoff-jin-cou-zhuang-tai-duan-dian-xu-pao-yu-fang-zhong-fu-ti-jiao-zhu-ce-biao)
- 降级裁决双记录门与红队评审的机器验证细节：[降级裁决与红队评审：双记录门、独立评审员与机器写入 spawn_agent 事件验证](18-jiang-ji-cai-jue-yu-hong-dui-ping-shen-shuang-ji-lu-men-du-li-ping-shen-yuan-yu-ji-qi-xie-ru-spawn_agent-shi-jian-yan-zheng)
- 四路报告的共享动作语义与分支树结构：[四维分析报告体系：失败案例、数据供给、学习动力学与分支保留历史的共享动作语义](24-si-wei-fen-xi-bao-gao-ti-xi-shi-bai-an-li-shu-ju-gong-gei-xue-xi-dong-li-xue-yu-fen-zhi-bao-liu-li-shi-de-gong-xiang-dong-zuo-yu-yi)
- 分类体系设计/实现分离原则与冻结契约：[分类体系设计契约：taxonomy_contract 冻结、类目树语义与设计/实现分离原则](15-fen-lei-ti-xi-she-ji-qi-yue-taxonomy_contract-dong-jie-lei-mu-shu-yu-yi-yu-she-ji-shi-xian-fen-chi-yuan-ze)
- 契约化的十一步流水线与硬性规则：[智能体契约与硬性规则：AGENTS.md 的十一步流水线、角色所有权与停止条件](8-zhi-neng-ti-qi-yue-yu-ying-xing-gui-ze-agents-md-de-shi-bu-liu-shui-xian-jiao-se-suo-you-quan-yu-ting-zhi-tiao-jian)
- 技能库的前置元数据与单一所有权规范：[技能库设计规范：Agent Skills 前置元数据、单一所有权与渐进式披露结构](10-ji-neng-ku-she-ji-gui-fan-agent-skills-qian-zhi-yuan-shu-ju-dan-suo-you-quan-yu-jian-jin-shi-pi-lu-jie-gou)