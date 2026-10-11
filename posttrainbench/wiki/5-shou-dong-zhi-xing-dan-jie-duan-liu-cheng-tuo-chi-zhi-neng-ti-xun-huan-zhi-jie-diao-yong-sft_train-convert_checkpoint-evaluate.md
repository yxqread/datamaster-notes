AutoSFT 的常规形态是一个由元智能体驱动的闭环：它自动选择数据动作、物化配方、提交训练、转换检查点、提交评测、再分析反馈并迭代。但在调试某个单独阶段、复现历史轮次、或以传统方式做对照实验时，你需要**跳过整个智能体循环，直接调用三个命令行入口**——`sft_train.py`（提交一轮 SFT 训练）、`convert_checkpoint.py`（将 Megatron 检查点转换为 HF 格式）、`evaluate.py`（提交基准评测）。本页面向中级开发者，说明手动模式的适用场景、三个入口的完整调用语法与参数语义、手动/守卫两种模式的差异、串联转换+评测的包装脚本，以及每一步在磁盘上留下的产物与幂等行为。官方 README 将这一能力明确列于 "Manual / By-Hand Operation" 一节：*To run a single stage outside the agent loop, use the same entrypoints the skills document*——手动模式不是旁门左道，而是与智能体技能共用同一批入口的一等公民接口。
Sources: [README.md](README.md#L378-L388)

> **📖 小白导读**
> - 这篇讲什么：跳过智能体循环、像传统脚本那样手工调用三个命令行入口——`sft_train.py`（训练）、`convert_checkpoint.py`（格式转换）、`evaluate.py`（评测）。
> - 读之前先知道：**CLI（命令行接口）**＝在终端里用参数驱动的程序入口；**检查点（checkpoint）**＝训练到某一步时模型权重的存档文件；**配方（recipe）**＝用 YAML 格式写的"数据配比单"（YAML＝用缩进表达层级的配置文本格式）。
> - 读完你能明白：三个命令各自吃什么参数、产出什么；单阶段复现历史轮次时怎么组装它们；手动模式下策略守卫对你是不是也生效（答案：生效）。
> - 提醒：日常无人值守运行不需要本页；它是调试与对照实验的"手术刀"。

## 为什么需要手动模式：固定接口设计下的研究变量分离

AutoSFT 的设计哲学是**研究变量只允许是数据（配方），训练与评测接口全部固定**。智能体循环的十一步流水线（AGENTS.md 第 13-25 行）中，第 5 步（提交 SFT）、第 7 步（转换检查点）、第 8 步（提交评测）都是通过这三个入口完成的；智能体本身并不直接调 `clusterctl pytorchjob create`，硬性规则明确禁止这一行为——*All SFT must go through a recipe YAML plus `sft_train.py`*。这意味着手动调用与智能体调用走的是完全相同的代码路径：同样的配方物化、同样的 SSTable 转换（SSTable＝训练引擎消费的按键排序数据文件格式）、同样的提交注册表幂等检查（幂等＝同一操作重复执行，不会重复提交，结果不变）、同样的台账记录。因此手动模式的可信度与智能体模式一致，产物也写入同样的目录结构。

手动模式的典型场景有三类。第一类是**分阶段调试**：你怀疑某一轮的转换环节有问题，只想重跑 `convert_checkpoint.py` 而不想重新发起完整训练。第二类是**手动/遗留的检查点契约**：`--resume-from`（断点续训）和 `--load-ckpt`（从指定权重初始化）这两个标志在守卫模式下被策略守卫拒绝（退出码 3），但它们被刻意保留用于手动操作——AGENTS.md 明确写道 *The `sft-checkpoint-resume-control` skill and the underlying explicit `--resume-from` / `--load-ckpt` flags remain available only for manual or legacy operation; the normal AutoSFT pipeline does not read or call that skill*。第三类是**独立评测某个历史检查点**：每个轮次的最终检查点会保留在磁盘上，你可以在任何时候转换并评测其中任意一个。
Sources: [AGENTS.md](AGENTS.md#L302-L311)
Sources: [AGENTS.md](AGENTS.md#L297-L300)

理解手动模式与守卫模式的关键差异在于一个环境变量。`policy_guard.py` 中的 `policy_enabled()` 仅在 `AUTO_SFT_POLICY_GUARD == "1"` 时激活；智能体运行由 `commit.sh` 启动时会设置此变量，而你在终端手动执行时不设置它，则 `guarded=False`。这个布尔值贯穿三个入口的行为分叉：守卫模式下 SFT 输出目录必须严格等于 `AUTO_SFT_RUN_DIR`（不匹配即记为 `sft_run_dir_mismatch` 违规并退出）、`--load-ckpt` / `--resume-from` 被拒绝、训练超参环境变量被封锁；手动模式下你可以自由指定 `--run-dir`、使用检查点标志，但**所有固定训练配置仍然生效**——三 epoch 调度、自动 train_iters 计算不可用 `--train-iters` 覆盖（该标志在任何模式下都会报错，见 `sft_train.py` 第 336-342 行）。
Sources: [policy_guard.py](src/control_plane/policy_guard.py#L192-L193)
Sources: [sft_train.py](sft_train.py#L336-L342)

## 手动三阶段总览：从配方到分数的完整链路

手动执行一轮完整实验的三阶段流程如下。注意转换是纯本地 CPU 工作，不消耗任何 Cluster 资源；训练和评测是异步提交到 Cluster 集群的作业，提交后需轮询状态。

```mermaid
flowchart LR
    A["recipe YAML<br/>(数据配方)"] -->|sft_train.py| B["prepare_recipe<br/>物化+转SSTable"]
    B --> C["submit_cluster_sft.sh<br/>→ 8-GPU训练作业"]
    C -->|Succeeded +<br/>checkpoint_complete| D["Megatron检查点<br/>iter_N/"]
    D -->|convert_checkpoint.py<br/>(本地CPU,幂等)| E["HF/vLLM模型目录<br/>+ conversion_manifest.json"]
    E -->|evaluate.py| F["每benchmark一个<br/>Cluster评测作业"]
    F -->|Succeeded| G["metrics.json 等<br/>评测产物"]
```

三条命令的核心形态（与 README 手动章节一致）：

```bash
python sft_train.py --recipe <recipe> --run-name iter_001                        # 阶段1：提交训练
python convert_checkpoint.py --checkpoint <run>/sft/checkpoints --ckpt-step <N> \
  --output <run>/hf_model --hf-template /path/to/Qwen3-1.7B-Base                  # 阶段2：本地转换
python evaluate.py --model-path <run>/hf_model --benchmark ifeval,ifbench \
  --run-name iter_001_eval                                                        # 阶段3：提交评测
```

每条命令都以 JSON 形式向 stdout 打印清单（manifest），这是判断操作是否真正成功的**权威依据**——例如训练提交的 `run_manifest.json` 中包含 `sft_task`（Cluster 任务 ID），若为 `null` 则没有任务被创建。
Sources: [README.md](README.md#L383-L388)
Sources: [sft_train.py](sft_train.py#L688-L715)

## 阶段一：sft_train.py —— 配方物化与训练作业提交

`python sft_train.py --recipe <recipe> --run-name <name>` 一次调用内完成六个动作：校验配方、物化数据、转换 SSTable、计算步数、提交作业、记录台账。其内部流程为：`main()` 先解析参数并检查守卫状态，随后（非续训模式下）调用 `prepare_recipe()` 物化配方——该函数将配方复制到 `run_dir/data-recipe.yaml`、写 `data_manifest.json`（含 `total_tokens` 与可选的 `token_accounting`）、并通过独立 Python 环境调用 SSTable 转换器生成 Megatron 格式数据集。接着 `resolve_train_iters()` 按公式 `train_iters = 3 * ceil(total_tokens / (global_batch_size * seq_length))` 计算总步数，`check_token_budget()` 在任何集群提交前执行预算一致性校验。最后通过 `submission_lock()`（基于 `fcntl.flock` 的文件锁）串行化"检查-创建"操作：若 `sft_submissions.jsonl` 中已存在同名 `run_name` 的记录，调用 `find_sft_submission()` 比对四元组（output_dir、dataset_path、train_iters、seq_length），完全一致则返回 `existing_task_found` 并复用任务，任何一项不同则**拒绝关闭**（fail closed）。
Sources: [sft_train.py](sft_train.py#L613-L647)
Sources: [submission_registry.py](src/control_plane/submission_registry.py#L44-L101)
Sources: [recipe_data.py](src/control_plane/recipe_data.py#L876-L929)

### 参数表

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--recipe` | 无（必填）* | 配方 YAML 路径，描述 HF 数据集与权重（见 `agent_execute/data-recipe.yaml` 示例）。*与 `--resume-from` 互斥，二选一 |
| `--run-name` | `sft_<时间戳>` | 轮次名。决定输出目录 `<run_dir>/sft/<run_name>/` 与幂等身份 |
| `--run-dir` | `agent_execute/runs/<run_name>` | 输出根目录。守卫模式下必须精确匹配 `AUTO_SFT_RUN_DIR` |
| `--datasets-dir` | `AUTO_SFT_DATASETS_DIR` 环境变量 | 本地已下载数据集目录 |
| `--seq-length` | `32768` | 序列长度（自动步数计算的输入之一；守卫模式禁止显式传递） |
| `--train-iters` | — | **任何模式下均不可设置**；传值即报错，步数总是自动计算 |
| `--resume-from <dir>` | — | **仅手动/遗留**：从已有运行目录续训，恢复原计划的 `train_iters`，忽略 `--recipe` |
| `--load-ckpt <dir>` | — | **仅手动/遗留**：从指定完整检查点目录初始化新轮次（重置优化器/LR/迭代计数） |
| `--dry-run` | 关 | 只做校验与打印，不创建任务、不写台账 |
| `--skip-data-prepare` | 关 | 跳过物化，要求 `AUTO_SFT_SFT_DATASET_PATH` 已设置且包含已知 token 总量，否则无法计算步数 |

Sources: [sft_train.py](sft_train.py#L386-L432)

### 手动续训与检查点加载的两种模式

`skills/sft-checkpoint-resume-control/SKILL.md` 定义了手动检查点契约的两种模式。**真续训**（`--resume-from`）保持迭代计数、Adam 优化器状态、LR 调度器和数据迭代器全部不变，训练延续到原运行的计划总步数（从 `run_manifest.json` 恢复），不要传 `--train-iters` 也不要重新乘三。**加载精确检查点**（`--load-ckpt`）则重置以上全部状态，用当前配方按标准三 epoch 公式重新训练，台账以 `branched_from` 记录显式来源。`--load-ckpt` 接受的目录必须包含 `latest_checkpointed_iteration.txt`（完整性标记），而不是某个 `iter_<N>` 子目录。注意 `sft_train.py` 在 `main()` 入口处会将 `--load-ckpt` 立即 `resolve()` 为具体路径，防止后续 `run_dir/sft/checkpoints` 符号链接重指后路径漂移。
Sources: [sft-checkpoint-resume-control/SKILL.md](skills/sft-checkpoint-resume-control/SKILL.md#L35-L73)
Sources: [sft_train.py](sft_train.py#L443-L452)

### 产物与幂等语义

一次成功提交在 `run_dir` 下产生：`data_manifest.json`（数据物化清单）、`run_manifest.json`（轮次清单，含任务 ID、步数、初始化来源）、`sft_submissions.jsonl`（提交注册表，追加写）、`iterations.jsonl` 台账记录、`data-recipe.yaml` 配方快照，以及 Megatron 检查点目录 `<run_dir>/sft/<run_name>/checkpoints/`。提交脚本 `submit_cluster_sft.sh` 将 `save-interval` 默认设为整个轮次长度（`DEFAULT_SAVE_INTERVAL="${TRAIN_ITERS}"`），因此每轮**只保存最终检查点**（单个 `iter_<final>`）。`sft_train.py` 还维护两个磁盘整理函数：`prune_prior_round_checkpoints()` 清理历史轮次中早于 `latest_checkpointed_iteration.txt` 的中间 `iter_*` 目录（当前轮次不受影响），`link_current_round_checkpoints()` 将 `run_dir/sft/checkpoints` 保持为指向当前活跃轮次检查点的符号链接——这使 README 手动命令中 `--checkpoint <run>/sft/checkpoints` 的写法始终有效。
Sources: [submit_cluster_sft.sh](skills/cluster-sft-job-submission/scripts/submit_cluster_sft.sh#L41-L53)
Sources: [sft_train.py](sft_train.py#L129-L193)

### 一个容易踩坑的前置条件

无论手动还是智能体模式，`prepare_recipe` 依赖 `AUTO_SFT_SFT_TOKEN_BUDGET` 环境变量：`src/commit_utils/set_env_vars.sh` 在脚本被 source 时检查该变量，若非正整数会直接报错退出（*must be supplied by the launcher as a positive integer*）。`resolve_token_budget()` 的语义是预算为**运行级别的硬上限**——配方中可以复述相同的 cap 供阅读，但若配方写了一个不同的值会被拒绝而非静默采纳。手动执行时你需要显式 `export AUTO_SFT_SFT_TOKEN_BUDGET=<正整数>`（或通过仓库根 `.env` 文件），这与智能体启动器自动注入的行为对齐。
Sources: [set_env_vars.sh](src/commit_utils/set_env_vars.sh#L52-L67)
Sources: [recipe_data.py](src/control_plane/recipe_data.py#L773-L814)

## 阶段二：convert_checkpoint.py —— Megatron 到 HF 的幂等本地转换

转换是三个阶段中最轻量的一个：它是纯本地 CPU 操作，不提交任何 Cluster 任务，通常在几分钟内完成。调用语法要求四个必填参数：

```bash
python convert_checkpoint.py \
  --checkpoint "<checkpoint_root>" \
  --ckpt-step "<step>" \
  --output "<output_dir>" \
  --hf-template "${AUTO_SFT_BASE_MODEL}"
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `--checkpoint` | 是 | Megatron 检查点根目录（含 `latest_checkpointed_iteration.txt` 的目录，如 `<run>/sft/<run_name>/checkpoints`） |
| `--ckpt-step` | 是 | 检查点步号，接受普通或前导零格式（`2` 或 `0000002` 等价，内部做规范化） |
| `--output` | 是 | HF 模型输出目录 |
| `--hf-template` | 是 | 基础 HF 模型/tokenizer 模板目录，提供 config/tokenizer 文件 |
| `--megatron-dir` | 否 | Megatron 转换器目录，默认 `tools/megatron_qwen3`（可被 `AUTO_SFT_MEGATRON_DIR` 覆盖） |
| `--python` | 否 | 转换用 Python 解释器，默认为独立 NumPy 1.x 环境（`AUTO_SFT_CONVERT_PYTHON` 可覆盖） |
| `--dry-run` | 否 | 只打印将执行的命令 |
| `--force` | 否 | 当输出目录的 manifest 记录的是不同输入时，强制替换 |

Sources: [convert_checkpoint.py](convert_checkpoint.py#L19-L33)

其内部实现有两点值得注意。**架构超参硬编码**：命令行固定传入 Qwen3-1.7B 的形状参数（`--num-layers 28 --hidden-size 2048 --ffn-hidden-size 6144 --num-attention-heads 16 --num-query-groups 8 --kv-channels 128`），即转换器只服务于这一特定模型。**环境隔离**：转换脚本通过 `subprocess.run()` 在 `cwd=megatron_dir` 且 `PYTHONPATH` 追加 megatron 目录的环境下用独立的 `--python` 解释器执行，避免 Megatron 依赖污染控制面环境。

幂等与安全语义由 `conversion_manifest.json` 驱动，分四种情形：若输出目录已存在 manifest 且三元组（规范化后的 checkpoint 路径、规范化步号、hf_template 路径）**完全一致**，且必需文件（`config.json`、`tokenizer_config.json` 加任一权重文件 `*.safetensors` 或 `*.bin`）齐全，则直接返回 `existing_conversion_found` 而不重跑转换器；若三元组**不一致**且未传 `--force`，则**拒绝关闭**并提示 *conversion output belongs to different inputs*；`--force` 保留给刻意的手动替换。转换完成后，脚本从模板复制七个 tokenizer/config 文件到输出目录，并最终写入新的 manifest（记录完整命令、时间戳、dry_run 标记）。
Sources: [convert_checkpoint.py](convert_checkpoint.py#L75-L98)
Sources: [convert_checkpoint.py](convert_checkpoint.py#L112-L180)
Sources: [megatron-hf-checkpoint-conversion/SKILL.md](skills/megatron-hf-checkpoint-conversion/SKILL.md#L29-L53)

**检查点完整性的判断准则**：可信标记是 `<checkpoint_root>/latest_checkpointed_iteration.txt`，读取它并确保请求步号不大于其值；**绝不要**通过枚举 `iter_*` 目录推断完整性——目录可能存在但写入是部分的（中途写入）。这是转换技能明确强调的安全准则。
Sources: [megatron-hf-checkpoint-conversion/SKILL.md](skills/megatron-hf-checkpoint-conversion/SKILL.md#L26-L31)

## 阶段三：evaluate.py —— 基准评测作业提交

`python evaluate.py --model-path <converted_model> --benchmark ifeval,ifbench --run-name <name>` 接收一个**已转换的 HF 模型目录**（即阶段二的输出），对每个基准提交一个独立的 Cluster 评测任务。benchmark 列表支持逗号或空格分隔并自动去重，默认 `ifeval,ifbench`。

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--model-path` | 无（必填） | 已转换 HF 模型目录（含权重与 config） |
| `--benchmark` | `AUTO_SFT_EVAL_BENCHMARK` 或 `ifeval,ifbench` | 基准列表，逗号/空格分隔；每个基准一个 Cluster 任务 |
| `--run-name` | `eval_<bench>_<时间戳>` | 评测运行名 |
| `--run-dir` | `agent_execute/runs/<run_name>`（守卫下为 `<RUN_DIR>/eval_runs/<run_name>`） | 输出根目录 |
| `--limit` | `AUTO_SFT_EVAL_LIMIT` 或 `-1`（不限） | 样本数上限 |
| `--eval-backend` | `inspect` | `inspect`（本地 Inspect AI/vLLM 路径）或 `opencompass`（在 PyTorchJob 内自托管 vLLM 并经 OpenAI 兼容 API 评测） |
| `--dry-run` | 关 | 传递给提交脚本，不真正创建任务 |

Sources: [evaluate.py](evaluate.py#L40-L67)

其核心执行逻辑围绕**幂等身份**展开。对每个 benchmark，`evaluate.py` 持运行级提交锁（`submission_lock`），以 `model_identity + benchmark + eval_backend + limit` 四元组为身份在 `AUTO_SFT_RUN_DIR/eval_runs/` 下所有规范 manifest 中查找既有提交——`model_identity` 不是可复用的路径字符串，而是从 `conversion_manifest.json` 读出的**检查点来源（checkpoint + step）**，这意味着即使你把模型目录复制到新路径，同一来源的模型仍被识别为同一身份。命中则返回 `existing_task_found` 与源 manifest 路径，即使新执行者换了评测运行名也不会创建重复任务；未命中则以 `bench_run_name`（多基准时为 `<run_name>_<benchmark>`）调用 `skills/cluster-eval-job-submission/scripts/submit_cluster_eval.sh` 创建任务。
Sources: [evaluate.py](evaluate.py#L118-L161)
Sources: [submission_registry.py](src/control_plane/submission_registry.py#L104-L130)

**部分失败容忍**：若某个 benchmark 提交失败（如预校验错误），其他 benchmark 的提交不受影响——失败的以 `submit_failed` 状态记入 manifest，成功的照常记录，最终命令以首个失败码退出。这是刻意设计：评测请求按基准独立报告。**后端切换规则**：`--eval-backend` 可为单次手动请求指定 `opencompass`，或用 `AUTO_SFT_EVAL_BACKEND` 环境变量为整个运行固定后端；但**绝不要在同一运行的不同轮次间切换后端**，因为 prompt、生成与打分方式的变化会使优化历史失去可比性。
Sources: [evaluate.py](evaluate.py#L162-L187)
Sources: [cluster-eval-job-submission/SKILL.md](skills/cluster-eval-job-submission/SKILL.md#L30-L58)

每次调用写 `eval_request.json`（含全部提交记录、模型身份、后端、limit），评测产物落在 `<run_dir>/eval/<benchmark>/` 下。评测本身是异步的：提交后立即返回，需要用生命周期脚本轮询（见下文"收尾"一节）。
Sources: [evaluate.py](evaluate.py#L244-L265)

## 一键包装：eval_round_ckpts.sh 串联转换与评测

如果你手里是一个**已完成的训练轮次**的检查点目录，不想分两次手动调用阶段二和阶段三，仓库提供了包装脚本 `skills/cluster-eval-job-submission/scripts/eval_round_ckpts.sh`：

```bash
bash skills/cluster-eval-job-submission/scripts/eval_round_ckpts.sh \
  "<round-checkpoint-root>"
```

它的执行逻辑是两遍式：第一遍读取 `latest_checkpointed_iteration.txt` 确定最终步号，枚举所有 `iter_<N>`（N ≤ 最终步号）目录——通常只有一个最终检查点——并对每步调用 `convert_checkpoint.py` 转到 `${AUTO_SFT_RUN_DIR}/ckpt_eval/<round>/step_<N>/hf_model`（本地 CPU、顺序执行）；第二遍对每个已转换模型调用 `evaluate.py`，因为提交是异步的，所有 (步号 × 基准) 任务会被紧凑地连续创建并在集群上并行运行。默认每个评测任务 1 GPU，通常的 1 步 × 2 基准 = 2 个任务并行。输出落在 `<RUN_DIR>/eval_runs/${RUN_PREFIX}_step_<N>/`。它支持 `AUTO_SFT_CLUSTER_DRY_RUN=1` 干跑。
Sources: [eval_round_ckpts.sh](skills/cluster-eval-job-submission/scripts/eval_round_ckpts.sh#L101-L145)

**关键前置约束**：只在 SFT 任务达到终止状态 `Succeeded` 之后运行——绝不在训练仍在进行时启动评测，评测任务不能与 8-GPU 训练任务抢卡。这条规则同样适用于手动模式。
Sources: [cluster-eval-job-submission/SKILL.md](skills/cluster-eval-job-submission/SKILL.md#L60-L78)

## 提交之后的收尾：轮询与收集（生命周期脚本）

三个入口只负责**提交**（阶段二除外），作业在 Cluster 上的完成状态需要用 `skills/cluster-job-lifecycle/` 下的脚本跟踪。手动模式下这些脚本同样可用：

```bash
# 轮询 SFT 状态（向 cluster_events.jsonl 追加增量记录）
AUTO_SFT_SFT_MANIFEST=<run>/sft_submissions.jsonl \
  bash skills/cluster-job-lifecycle/scripts/cluster_sft_status.sh

# 收集 SFT 结果（写 sft_final_status.json 与 loss 产物）
AUTO_SFT_SFT_MANIFEST=<run>/sft_submissions.jsonl \
  bash skills/cluster-job-lifecycle/scripts/cluster_sft_collect.sh

# 轮询/收集评测（eval 侧对应脚本）
AUTO_SFT_EVAL_MANIFEST=<eval_run>/eval_submissions.jsonl \
  bash skills/cluster-job-lifecycle/scripts/cluster_eval_status.sh
AUTO_SFT_EVAL_MANIFEST=<eval_run>/eval_submissions.jsonl \
  bash skills/cluster-job-lifecycle/scripts/cluster_eval_collect.sh
```

成功判据要**信任真实的 `cluster_status` 而非文件存在性**：SFT 成功要求 `cluster_status == "Succeeded"` 且 `checkpoint_complete == true`；评测成功要求每个请求的 benchmark 任务都 `Succeeded` 且 `artifacts_complete == true`。失败或停止的任务可能留下部分输出，绝不要仅凭检查点或分数文件的存在就报告成功。manifest 驱动的轮询会跳过最新事件已终止的任务，重复调用是安全的。
Sources: [cluster-job-lifecycle/SKILL.md](skills/cluster-job-lifecycle/SKILL.md#L43-L104)

## 手动模式与守卫模式行为对照

| 维度 | 手动模式（无 `AUTO_SFT_POLICY_GUARD`） | 守卫模式（智能体运行） |
|------|----------------------------------------|------------------------|
| `--load-ckpt` / `--resume-from` | 可用（经 `sft-checkpoint-resume-control` 技能契约） | 直接拒绝，退出码 3 |
| 输出目录 | 可自由指定 `--run-dir`，默认 `agent_execute/runs/<name>` | 必须精确等于 `AUTO_SFT_RUN_DIR`，不匹配记 `sft_run_dir_mismatch` / `eval_run_dir_mismatch` |
| `--train-iters` | **仍然禁止**（自动计算） | 同样禁止 |
| `--seq-length` / `AUTO_SFT_SFT_*` 环境变量 | 可传（但不推荐偏离默认） | 策略守卫拦截 `training_param_override` |
| 台账记录 | 照常写入 `iterations.jsonl` | 照常写入 |
| 提交幂等 | 同一套 `submission_lock` + 注册表检查 | 同一套 |
| 评测后端 | 可每次指定 | 运行级固定，禁止轮次间切换 |

需要强调：**手动不等于无约束**。训练配置的固定性（三 epoch、自动步数、默认超参）是接口契约而非仅守卫规则——`--train-iters` 在源码层面对任何调用者报错；转换器的 Qwen3-1.7B 形状参数同样硬编码。手动模式放开的是**检查点契约**与**输出位置**，放开的是操作编排方式，而不是研究变量范围。
Sources: [AGENTS.md](AGENTS.md#L302-L320)
Sources: [sft_train.py](sft_train.py#L336-L342)

## 常见问题排查

| 症状 | 原因 | 处置 |
|------|------|------|
| `--train-iters` 传值即报错 | 设计如此：步数自动按 `3 * ceil(total_tokens / (global_batch_size * seq_length))` 计算 | 省略该参数；若想训练更久，增加配方数据量 |
| 报 `Cannot auto-compute --train-iters: tokenized data size is unknown` | `--skip-data-prepare` 模式下 manifest 缺 `total_tokens` | 提供带 `total_tokens` 的路径，或不用 `--skip-data-prepare` |
| `sft_train.py` 报 `--recipe is required unless --resume-from is provided` | 两个数据来源标志都没给 | 传 `--recipe`（新轮次）或 `--resume-from`（续训），二选一 |
| 训练提交报 `AUTO_SFT_SFT_TOKEN_BUDGET must be supplied by the launcher` | `set_env_vars.sh` 未检测到正整数预算 | 手动 `export AUTO_SFT_SFT_TOKEN_BUDGET=<正整数>` 或配置 `.env` |
| 转换报 `conversion output belongs to different inputs` | 输出目录已有 manifest 且三元组（checkpoint/step/template）不匹配 | 换 `--output` 目录，或确认确实要替换时传 `--force` |
| `evaluate.py` 返回 `existing_task_found` | 幂等身份（模型来源+基准+后端+limit）已存在任务 | 这是正常复用；轮询既有任务即可，勿重复提交 |
| `submit_cluster_sft.sh exited with code N` 且无 manifest 记录 | 预校验失败，**没有任务被创建** | 读上方 ERROR 行定位；重试前先检查 manifest 确认无歧义 |
| 评测任务与训练抢卡 | 违反"评测必须在 SFT 终止后"规则 | 停止评测任务，等 SFT `Succeeded` 后重提 |

Sources: [sft_train.py](sft_train.py#L196-L212)
Sources: [sft_train.py](sft_train.py#L508-L509)
Sources: [convert_checkpoint.py](convert_checkpoint.py#L147-L151)
Sources: [cluster-sft-job-submission/SKILL.md](skills/cluster-sft-job-submission/SKILL.md#L65-L74)

## 手动产物的后续消费

手动模式写入的产物与智能体模式完全同构，因此可以互相消费。手动提交的 SFT 轮次照样进入实验台账 `iterations.jsonl`，可以用 `python -m src.control_plane.ledger summarize` 回顾；评测产出的 `metrics.json` / `scores.json` / `failure_breakdown.json` 可用台账的 `best-scores` 命令查询最佳分数。一个实用的混合工作流是：让智能体跑出若干轮后，手动对某个历史检查点做补充评测或用 `--load-ckpt` 做对照分支实验——所有轮次的最终检查点都保留在 `<run_dir>/sft/<run_name>/checkpoints/` 供随时转换评测，也可用于最终化时挑选最佳历史模型。

**后续阅读**：想了解提交后在集群侧的完整作业治理（有界重试、事件日志、停止语义），参见 [Cluster 作业生命周期：提交、轮询、停止、收集与 clusterctl 有界重试基础设施](22-cluster-zuo-ye-sheng-ming-zhou-qi-ti-jiao-lun-xun-ting-zhi-shou-ji-yu-clusterctl-you-jie-zhong-shi-ji-chu-she-shi)；检查点转换的底层机制（分布式 Megatron 格式细节、Python 环境隔离原因）见 [检查点转换：Megatron 分布式到 HF/vLLM 格式的幂等转换与 Python 环境隔离](20-jian-cha-dian-zhuan-huan-megatron-fen-bu-shi-dao-hf-vllm-ge-shi-de-mi-deng-zhuan-huan-yu-python-huan-jing-ge-chi)；评测双后端架构见 [评测后端双轨：Inspect AI 默认路径与 OpenCompass vLLM 自托管 API 桥接](21-ping-ce-hou-duan-shuang-gui-inspect-ai-mo-ren-lu-jing-yu-opencompass-vllm-zi-tuo-guan-api-qiao-jie)；手动产物在 `agent_execute/runs` 下的完整目录布局见 [运行产物目录全解：agent_execute/runs 下的台账、报告、检查点与分类数据布局](6-yun-xing-chan-wu-mu-lu-quan-jie-agent_execute-runs-xia-de-tai-zhang-bao-gao-jian-cha-dian-yu-fen-lei-shu-ju-bu-ju)。