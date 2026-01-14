# mani_longvideo 任务清单（v6）深度审计与扩展建议

本文档面向 `ECCV/chat/mani_longvideo_tasks_plan_final_v6.md` 中的 **Task_01 ~ Task_37**，结合 three-stage 数据产物的字段 Schema，对当前任务体系做一次“可落地、可评分、低噪声”的深度复盘：

1) 逐任务分析其训练价值、潜在噪声来源、与其他任务的重复关系，并给出“保留/降权/合并/删除”的建议；
2) 基于现有 Schema 字段，提出若干**新增且有价值**的任务类型（尽量客观题化），补齐当前任务集的覆盖空白。

---

## 1. 我们能用的 Schema 信号（任务设计的“上限”）

以 `<ITEM_DIR>/causal_plan_with_keyframes.json` 为核心，关键可用字段包括：

- 顶层：`high_level_goal`、`steps`
- Step：`step_goal`、`rationale`、`causal_chain`、`counterfactual_challenge_question/expected_challenge_outcome`、`failure_reflecting.reason/recovery_strategy`、`critical_frames`（2 张）
- `causal_chain`：
  - 实体槽位：`agent`、`action`、`patient`
  - 约束与后果：
    - `causal_precondition_on_spatial`: `relation/objects/truth`
    - `causal_precondition_on_affordance`: `object_name/affordance_types/reasons`
    - `causal_effect_on_spatial`: `relation/objects/truth`
    - `causal_effect_on_affordance`: `object_name/affordance_types/reasons`
- `critical_frames[*]`：
  - `action_state_change_description`
  - `interaction.tools/materials`
  - `interaction.hotspot.description/affordance_type/mechanism`

> 重要边界：这些字段本质上来自上游标注模型（或其后处理），并不等价于“绝对真值”。因此任务设计要优先选择：**可由视觉证据直接核验**、或可用**可控的合成负样本**来强化鲁棒性的形式。

---

## 2. 审计评价维度（用来判断“是否无意义/是否重复/是否该删”）

对每个任务建议从以下维度评估：

1. **可评分性（Objective）**：能否稳定自动评分（A/B/C/D、三态、三分类、序列精确匹配）？自由文本越长越难评。
2. **可核验性（Groundable）**：答案是否能从给定证据中看出来？大量“不可观测命题”会导致噪声堆积。
3. **抗泄漏性（Non-leaky）**：是否会被文件名/目录名/ts 等旁路信息击穿？（工程侧需严格屏蔽）
4. **负样本可控性（Negatives）**：是否能用“单一扰动”生成高质量负样本（只替换一个对象/翻转一个关系/打乱一个顺序）？
5. **边际增益（Marginal Value）**：是否已经被其它任务覆盖（同字段、同证据、同输出形态）？重复信号过多会浪费配比。

---

## 3. 任务集的总体结构与潜在空洞

### 3.1 当前覆盖强项

- **强可评分任务**：MCQ（Task_24/26/27/28/29/33/34）、三态核验（Task_14/15/20/31/32）、序列任务（Task_22/23）、一致性/缺陷定位（Task_16/21/30）。
- **长时序能力**：prefix → next（Task_18）、prefix → progress（Task_35）、prefix → reorder/infill（Task_22/23）。

### 3.2 当前容易“重复/低增益”的区域

- **空间关系**：Task_02（复述）与 Task_20（真假核验）高度同域。
- **热点可供性**：Task_03（自由文本）与 Task_26/27（MCQ）信号重叠。
- **反事实/失败反思**：Task_12/13（自由文本）与 Task_33/34（客观题）重叠。
- **后置效果**：Task_09（自由文本）与 Task_31/32（三态核验）重叠。
- **下一步预测**：Task_11（弱监督/计划）与 Task_18（prefix 视觉预测）同目标不同证据，训练主线通常只需其一。

### 3.3 当前可能的“能力空洞”

在不新增字段的前提下，仍有几个“用现有字段能做、但 v6 里缺少/偏弱”的方向：

1) **对称检索任务**：目前有 “图 → step_goal” (Task_24)，缺少 “step_goal → 图” 的检索/匹配客观题（非常有助于对齐与检索评测）。
2) **理由（reasons）字段的客观化使用不足**：`reasons` 目前主要以自由文本存在，没有做“理由一致性/对齐核验”的客观任务。
3) **跨步因果对齐的客观化不足**：Task_10 是自由文本解释，缺少“effect ↔ precondition 的配对/选择题”版本（更稳定可评测）。
4) **坏计划的“修复”**：Task_21 只定位缺陷，不要求输出修复后的正确计划；缺少“纠错→重排/替换”的可评分任务。

---

## 4. 逐任务深度审计（Task_01 ~ Task_37）

下面每个任务给出：价值、噪声点、重复关系、建议动作。

### Task_01_Macro_Anchor_Extraction

- **价值**：提供“场景锚点对象词表”，可作为 MCQ 候选池/负样本池；对整体 grounding 有帮助。
- **主要噪声**：锚点列表来自上游标注，可能漏标/错标；开放式抽取难精确评分。
- **重复关系**：与 Task_29（patient 识别）不冲突；更像“候选池构建”。
- **建议**：保留，但建议做一个可评分变体（见新增任务建议：Anchor Yes/No 或 MCQ），并降低其在评测中的比重。

### Task_02_Transient_Geometric_Verification

- **价值**：训练空间关系的自然语言表达（paraphrase）。
- **主要噪声**：自由文本难评分；容易退化为模板句；与真假核验任务重复。
- **重复关系**：与 Task_20（关系核验）高度重叠。
- **建议**：强烈建议删去或极低比例保留（仅用于语言多样性），主线保留 Task_20 即可。

### Task_03_Micro_Affordance_Visual_Semantics

- **价值**：训练“热点—可供性—机制”的解释能力，是物理理解的重要表达形态。
- **主要噪声**：自由文本机制描述可能泛化/口号化；机制句子存在同义改写难评分。
- **重复关系**：与 Task_26/27（affordance_type/mechanism 的 MCQ）信息重叠。
- **建议**：保留但降权；作为补充的“解释型”数据。若以评测为主，优先用 Task_26/27。

### Task_04_Entity_Role_Identification

- **价值**：训练“工具 vs 材料”的角色区分；可做 Yes/No 客观题，适合自动评分。
- **主要噪声**：tools/materials 的标注可能主观；某些对象在不同步里角色会变化。
- **重复关系**：与 Task_01（对象集合）相辅相成；不算重复。
- **建议**：保留；推荐以 Yes/No 版本为主，少量自由文本。

### Task_05_State_Evolution_Description

- **价值**：训练关键帧“事件描述”和“状态变化”抽取；为 Task_19 的事件文本提供同风格能力。
- **主要噪声**：与 Task_06（因果链解释）存在表达层重复；自由文本难评。
- **重复关系**：与 Task_06/19 在语义上相邻，但粒度更小（事件级）。
- **建议**：若资源有限可降权；若要保留，建议增加一个客观化变体（见新增任务：Event Caption MCQ）。

### Task_06_Holistic_Causal_Chain_Analysis

- **价值**：把空间+可供性+机制+后效串成闭环，最贴合“物理因果理解”的长答训练。
- **主要噪声**：长自由文本难自动评分；容易输出泛泛解释；上游 causal_chain 噪声会被放大。
- **重复关系**：其核心信息已被 Task_20/26/27/28/29/31/32 等多个客观任务“拆分覆盖”。
- **建议**：作为解释能力的补充可保留少量；若目标是可评分训练/评测，建议降权或改为结构化 JSON 输出（更可控）。

### Task_07_Strategic_Rationale_Justification

- **价值**：训练“为什么做这一步”的因果动机表达，有助于规划可解释性。
- **主要噪声**：rationale 往往模板化（如“enable subsequent steps”）；难评分。
- **重复关系**：与 Task_10（跨步依赖）存在“因果解释”重复。
- **建议**：可删或降权；若保留，建议将答案长度限制在 1 句并要求引用具体对象/状态（减少空话）。

### Task_08_Step_Execution_Statement

- **价值**：训练把视频片段总结成可观测“微动作列表”，可提升动作分解能力。
- **主要噪声**：片段对齐若不准会引入跨步动作；“看不清”比例可能很高；自由文本难评。
- **重复关系**：与 Task_16（对齐判别）在证据层类似，但输出更开放。
- **建议**：若追求稳定训练/评测，建议删或显著降权；用 Task_16 + Task_24 替代其主要价值。

### Task_09_Expected_Physical_Effects

- **价值**：学习“动作完成后应该发生什么”。
- **主要噪声**：很多 effect/affordance 在单帧不可观测，容易口头化；自由文本难评。
- **重复关系**：与 Task_31/32（后置三态核验）高度重叠。
- **建议**：建议删去或仅保留极低比例（写作训练）；主线用 Task_31/32。

### Task_10_Inter_Step_Dependency_Analysis

- **价值**：训练跨步因果桥接（effect → precondition），是规划学习的关键。
- **主要噪声**：自由文本解释可能牵强；当 overlap 不强时会产生“强行解释”。
- **重复关系**：与 Task_07 同属因果解释；但 Task_10 更结构化（依赖 overlap 才生成）。
- **建议**：保留，但建议新增一个“配对/选择题”版本（见新增任务：Effect↔Precondition Matching），将其客观化。

### Task_11_Next_Action_Prediction

- **价值**：下一步 step_goal 预测的“弱监督/计划基线”。
- **主要噪声**：不如 prefix 视觉版本 grounded；容易退化为“语言模型按常识续写”。
- **重复关系**：与 Task_18（prefix 视觉预测）目标重合。
- **建议**：若任务主线是多模态长视频，建议删或仅保留极低比例作为 baseline；主线保留 Task_18。

### Task_12_Counterfactual_Prediction

- **价值**：训练 what-if 下的物理后果推理与应对策略（解释能力强）。
- **主要噪声**：counterfactual 与 outcome 本身是“假设性标注”，不一定与视觉强绑定；自由文本难评。
- **重复关系**：与 Task_33（counterfactual outcome MCQ）重叠。
- **建议**：作为解释型数据可保留少量；若目标可评分训练/评测，建议用 Task_33 为主。

### Task_13_Failure_Recovery_Protocol

- **价值**：训练失败原因与恢复策略的语言表达，为失败反思提供自然语言能力。
- **主要噪声**：failure_reflecting 是假设性标注；自由文本难评且容易模板化。
- **重复关系**：与 Task_34（recovery strategy MCQ）重叠。
- **建议**：降权；评测主线用 Task_34。

### Task_14_Physical_Feasibility_Verification

- **价值**：把“前置条件是否满足”上升为“此刻是否可行”的三态判别；可通过合成负样本增强鲁棒性。
- **主要噪声**：如果仅用真实样本，它几乎总是 feasible（因为视频里确实做了）；必须依赖高质量负样本。
- **重复关系**：与 Task_15（prefix 前置核验）部分重叠，但证据不同（关键帧 vs prefix）。
- **建议**：保留，但务必把负样本作为主比例（例如 50%），否则训练会退化。

### Task_15_Precondition_Check_or_MCQ

- **价值**：长时序规划的关键：基于 prefix 判断某步前置条件是否已满足/不可观测。
- **主要噪声**：很多 affordance/内部状态不可观测，若不严格三态会引入噪声；prefix 对齐不准也会噪声。
- **重复关系**：与 Task_14 同域不同证据。
- **建议**：保留；建议优先做“单条核验三态”，并对“不可观测”做高比例采样（让模型学会说不知道）。

### Task_16_Plan_Execution_Alignment

- **价值**：对齐判别（match/partial/mismatch）是失败反思与数据质量控制的核心；负样本构造简单且可控。
- **主要噪声**：clip 对齐不准会导致 partial/mismatch 混淆；需在生成器中尽量对齐 step。
- **重复关系**：与 Task_08（微动作描述）在证据层相近，但本任务可评分性更强。
- **建议**：强烈建议保留并作为核心任务之一；同时建议把 partial/mismatch 的判据写得更硬（例如必须指出可见证据）。

### Task_17_Goal_Recognition_From_Full_Video

- **价值**：学习从长视频提炼 high_level_goal，帮助 goal-conditioned planning。
- **主要噪声**：自由文本很难自动评分；也可能偏模板化。
- **重复关系**：与序列规划任务（Task_22/23）相邻但不重复。
- **建议**：建议增加一个客观化版本（见新增任务：Goal Retrieval MCQ），自由文本保留少量即可。

### Task_18_Next_Step_Goal_Prediction_From_Prefix

- **价值**：多模态长视频规划的核心任务，输出严格可评分（exact match）。
- **主要噪声**：prefix 证据不足时会变成常识猜测；需用 curriculum（prefix 从短到长）。
- **重复关系**：与 Task_11 同目标；本任务更 grounded。
- **建议**：保留并作为主线；建议同时保留“候选列表排序/四选一”变体（可进一步提升可评分性）。

### Task_19_Temporal_Order_Check_AB

- **价值**：训练跨步时间顺序理解，输出 A/B 可评分；可构造大量样本。
- **主要噪声**：需要在工程侧用 ts 判定 label，但 ts 不能进入 prompt；短视频/重复帧会导致 ts 相同需过滤。
- **重复关系**：与 Task_25（同一步 initiation/completion）互补：一个跨步、一个步内阶段。
- **建议**：保留；注意过滤重复帧与 ts 相等样本。

### Task_20_Visual_Spatial_Relation_Check

- **价值**：把空间关系变成三态可评分监督；负样本可控（翻转 truth/替换 objects）。
- **主要噪声**：某些关系在单帧难判断（接触/inside 的遮挡），需要 not observable。
- **重复关系**：覆盖了 Task_02 的主要价值。
- **建议**：保留；建议删 Task_02 或只留极低比例。

### Task_21_Failed_Planning_Flaw_Pointing

- **价值**：失败反思的关键：从坏计划中定位缺陷步骤与缺陷类型；单一扰动可控、可评分。
- **主要噪声**：Reason 是自由文本，自动评分时应只评 flaw_step/flaw_type；Reason 可选。
- **重复关系**：与 Task_22/23 同属规划，但更偏“诊断”。
- **建议**：保留；建议新增“修复计划”任务（见新增任务：Plan Repair）。

### Task_22_Next_K_Steps_Reordering_From_Prefix

- **价值**：多步规划的核心：在候选集合内重排序，强可评分。
- **主要噪声**：候选若语义过近会造成歧义；建议用跨步依赖/工具依赖筛选 harder cases。
- **重复关系**：与 Task_23（infill）互补：reorder 是“给候选”，infill 是“开放补全”。
- **建议**：保留并作为核心；适当提高其占比（比 infill 更稳）。

### Task_23_Middle_Steps_Infill_From_Head_Tail

- **价值**：长时序补全能力（head+tail → middle steps），对真实长视频规划非常重要。
- **主要噪声**：head/tail 证据可能不足导致猜测；开放输出更难评（但仍可用 exact match）。
- **重复关系**：与 Task_22 同类但难度更高。
- **建议**：保留但相对降权；优先保证 Task_22 的覆盖，再做 Task_23 curriculum。

### Task_24_Keyframe_to_StepGoal_Matching_MC

- **价值**：强监督对齐：图像 → step_goal 四选一，极易自动评分；可做为检索/对齐基准任务。
- **主要噪声**：干扰项若过于容易会导致任务过简单；需要 hard negatives（同类动作/相同物体）。
- **重复关系**：与 Task_08（微动作）部分同目标，但本任务更稳。
- **建议**：强烈建议保留并提高占比；同时建议补一个对称任务（step_goal → 图，见新增任务）。

### Task_25_Init_vs_Complete_Keyframe_Order

- **价值**：步内阶段理解（起始 vs 完成），对“状态变化”学习有帮助。
- **主要噪声**：某些步骤两张关键帧差异可能很小，导致歧义；需过滤“几乎一样”的对。
- **重复关系**：与 Task_19 互补；与 Task_05/06 相关但不重复。
- **建议**：保留但注意过滤“无变化”样本。

### Task_26_Hotspot_AffordanceType_MCQ

- **价值**：把 affordance_type 学习变成可评分 MCQ；与物理交互理解高度相关。
- **主要噪声**：affordance_type 标签的粒度与一致性要控制（同义/近义标签会使 MCQ 变难评）。
- **重复关系**：覆盖 Task_03 的部分信息。
- **建议**：保留；建议维护 affordance_type 的小词表/规范化映射（工程侧）。

### Task_27_Hotspot_Mechanism_MCQ

- **价值**：训练机制选择（force_transfer/friction/leverage/...），对因果理解很有价值。
- **主要噪声**：mechanism 往往是自由文本长句，若不规范化会导致候选答案不稳定、难构造好干扰项。
- **重复关系**：与 Task_03 重叠较强。
- **建议**：保留但建议先做“机制标签归一化”（小 taxonomy），否则训练/评测都容易漂。

### Task_28_Action_Phrase_MCQ

- **价值**：动作短语选择（action），可评分且对规划语言对齐有用。
- **主要噪声**：action 也可能有同义改写；需要候选池规范化或 hard negatives 设计。
- **重复关系**：与 Task_24（step_goal）不同槽位，不算重复。
- **建议**：保留；建议把 action 也做轻量规范化（短语模板）。

### Task_29_Patient_Identification_MCQ

- **价值**：学习“被作用对象（patient）”定位，是因果链的核心槽位之一；易评分。
- **主要噪声**：patient 有时可能是复合实体或命名不稳定；候选池需去重与同义合并。
- **重复关系**：与 Task_01（对象集合）互补。
- **建议**：保留；建议限制为单选场景或在多选时明确评分规则。

### Task_30_Step_vs_Keyframe_CausalChain_Consistency

- **价值**：一致性检测 + 反事实负样本识别，非常适合做“失败反思/纠错”训练。
- **主要噪声**：step-level chain 是“整步”，keyframe chain 是“瞬间”，天然会有细节差；不应把细节差当 inconsistency。
- **重复关系**：与 Task_21（坏计划诊断）互补：一个诊断结构一致性，一个诊断计划错误。
- **建议**：保留；负样本要“强替换”（替换 patient/action），避免弱差异造成误标。

### Task_31_Spatial_Postcondition_Check

- **价值**：把“因果后果（空间关系）”变成三态核验，强可评分，直接服务因果规划。
- **主要噪声**：遮挡/视角导致不可观测；需允许 not observable 并采样可观测关系优先。
- **重复关系**：可替代 Task_09 的大部分目标。
- **建议**：强烈建议保留并作为核心；同时建议构造 hard negatives（对象替换/关系替换）。

### Task_32_Affordance_Postcondition_Check

- **价值**：把“动作→可供性改变”作为可评分核验/选择题，是可执行规划的关键。
- **主要噪声**：很多 affordance 不直接可见；若采样不当会被 not observable 淹没。
- **重复关系**：可替代 Task_09（affordance 部分）。
- **建议**：保留；采样时优先选择“可见状态类 affordance”（open/closed/inserted/holding 等）。

### Task_33_Counterfactual_Outcome_MCQ

- **价值**：把反事实结果客观化（四选一），可评分且能训练物理后果推理。
- **主要噪声**：outcome 是假设性标注；若干扰项设计差会变成“读关键词”。
- **重复关系**：覆盖 Task_12 的可评分核心。
- **建议**：保留，但建议提高干扰项质量（同对象/同工具的 hard negatives）。

### Task_34_Recovery_Strategy_MCQ

- **价值**：把恢复策略客观化（四选一），可评分，适合 failure reflecting 的稳定指标。
- **主要噪声**：recovery_strategy 假设性强；若候选过容易会变成模板选择。
- **重复关系**：覆盖 Task_13 的可评分核心。
- **建议**：保留；干扰项建议来自“同失败类型但不同具体对象”的策略，提升区分度。

### Task_35_Prefix_Completed_Steps_MultiSelect

- **价值**：长时序“进度理解”可评分化（推荐输出最大完成 step_id），对 prefix 规划很重要。
- **主要噪声**：若 prefix 不是严格切到 step 尾帧会引入偏差；需要工程侧保证 prefix 定义一致。
- **重复关系**：与 Task_18/22/23 互补（一个是进度、一个是下一步、一个是未来序列）。
- **建议**：保留；优先做“最大 step_id”版本（最稳定）。

### Task_36_Stage2_Temporal_Localization_Check（可选）

- **价值**：利用 Stage2 的 start/end indices 做更严格的时间定位监督；可作为强评测指标。
- **主要噪声/门槛**：依赖 Stage2 产物存在且可读；抽帧过稀会导致边界误差变大。
- **重复关系**：与 Task_19（事件先后）互补：一个是全局边界，一个是相对顺序。
- **建议**：保留为可选；若没有稳定 Stage2 产物，可暂不启用。

### Task_37_Next_Step_After_Recovery

- **价值**：形成失败反思闭环：失败 → 恢复 → 下一步选择（重规划雏形）。
- **主要噪声**：label 若固定为 `steps[i+1].step_goal`，在真实情形下不一定成立（很多恢复需要“重做当前 step”或插入额外动作）。
- **重复关系**：与 Task_34（选 recovery_strategy）相关；但本任务意图更强。
- **建议**：
  - 若不扩展 schema：建议降权或只在“恢复明显是前置修复且不改变 step 本身”的场景使用；
  - 更推荐把它改成“恢复后是 **重试本步** 还是 **继续下一步** 的二分类”，并将 `continue/redo` 作为 label（见新增任务建议）。

---

## 5. 建议删除/合并清单（面向“去重与提纯”）

如果目标是构建一个更“可评分、低噪声、训练效率高”的任务集，建议按以下优先级裁剪：

### 5.1 强烈建议删除或极低比例保留

- `Task_02`：与 `Task_20` 高重复，且自由文本难评分。
- `Task_08`：微动作列表对齐噪声较大，且与 `Task_16/24` 的边际增益有限。
- `Task_09`：与 `Task_31/32` 重复，且不可观测命题会导致大量噪声。
- `Task_11`：与 `Task_18` 同目标，作为 baseline 可留极少量，否则可删。

### 5.2 建议降权（保留少量“解释能力”）

- `Task_03/05/06/07/12/13/17`：主要价值是解释与生成能力，但评分弱、易模板化；建议作为辅助数据而非核心评测指标。

### 5.3 强烈建议保留为核心（客观题/核验/序列）

- 对齐与核验：`Task_14/15/16/20/31/32`
- 长时序规划：`Task_18/22/23/35`（再配合 `Task_19` 做时序能力）
- 纠错与一致性：`Task_21/30`
- 关键槽位对齐：`Task_24/26/27/28/29`
- 失败反思客观化：`Task_33/34`

---

## 6. 基于 Schema 的新增高价值任务（建议新增）

以下新增任务均 **不需要新增字段**，只基于现有 schema 与可控负样本构造；优先客观题化。

> 命名建议从 Task_38 起，仅作提案；是否落入正式任务清单由你决定。

### Task_38_StepGoal_to_Keyframe_Matching_MC（step_goal → 关键帧四选一）

- **动机**：补齐 Task_24 的对称检索能力（文本检索图像），对检索/对齐评测非常有用。
- **字段来源**：`steps[s].step_goal`（query），正例图来自 `steps[s].critical_frames[0]`；负例图来自其他 steps 的关键帧。
- **证据**：4 张 keyframe 图（A/B/C/D）
- **输出**：`A/B/C/D`
- **负样本**：hard negatives 优先选“同工具/同材料/相近动作”的 step 的关键帧。

### Task_39_Affordance_Precondition_MCQ（前置可供性四选一）

- **动机**：目前 Task_15 是 prefix 核验；新增一个“从动作/证据推断必须具备的 affordance”任务，强化可执行规划。
- **字段来源**：`steps[i].causal_chain.causal_precondition_on_affordance[*].object_name/affordance_types`
- **证据**：step i 的 initiation 关键帧（`critical_frames[0]`）
- **输出**：给定 object_name，四选一选择正确 affordance_type（或 Yes/No）
- **负样本**：从其他 step 的 affordance_types 抽取干扰项（优先语义相近）。

### Task_40_Effect_to_Precondition_Matching（跨步 effect ↔ precondition 配对题）

- **动机**：把 Task_10 的自由文本解释客观化：让模型“选对依赖边”，而不是写空话。
- **字段来源**：
  - 左侧候选：`steps[i].causal_chain.causal_effect_on_*`（抽 2–4 条）
  - 右侧候选：`steps[i+1].causal_chain.causal_precondition_on_*`（抽 2–4 条）
- **证据**：可选 keyframe_single（step i 尾关键帧）或 text_only（如果你只做结构监督）
- **输出**：配对结果（例如 `P1<-E2; P2<-E1` 的固定格式）或单选（“哪个 effect 最支持这个 precondition？”）
- **负样本**：打乱配对关系，或插入来自其他 step 的 effect/precondition。

### Task_41_Affordance_Reason_Consistency_Check（reasons 一致性三分类）

- **动机**：充分利用 `reasons` 字段：让模型学会判断解释是否真的与证据/对象/affordance 对齐。
- **字段来源**：`causal_precondition_on_affordance[*].reasons` 或 `causal_effect_on_affordance[*].reasons`
- **证据**：对应关键帧（precondition 用 initiation，effect 用 completion）
- **输出**：`consistent / inconsistent / not directly observable`
- **负样本**：把 reasons 与不同对象/不同 affordance_types 交叉配对。

### Task_42_Plan_Repair_From_Flaw（坏计划修复：输出纠正后的计划）

- **动机**：Task_21 只“定位缺陷”，缺少“给出修复后 plan”的训练；修复任务更贴近真实 failure reflecting。
- **字段来源**：Task_21 的 `bad_plan_steps` 与 gold `steps[i+1:i+K].step_goal`
- **证据**：`video_prefix`（同 Task_21）
- **输出**：纠正后的 K 步序列（严格匹配 gold 顺序；或先输出修复后的候选集合再排序）
- **评分**：exact match（序列）

### Task_43_Recovery_then_Retry_or_Continue（二分类：恢复后重试本步/继续下一步）

- **动机**：替代/修正 Task_37 的潜在弱标问题，避免把 label 绑死在 i+1。
- **字段来源**：`failure_reflecting.reason/recovery_strategy` + `steps[i].step_goal` + `steps[i+1].step_goal`
- **证据**：`video_prefix` 或 keyframe_single
- **输出**：`retry_current_step` / `continue_next_step`
- **弱监督标签建议**（无需新增字段）：
  - 如果 recovery_strategy 明显是“让本步可完成”的修复（如“稳住/重新抓取/增加摩擦”），倾向 `retry_current_step`；
  - 如果 recovery_strategy 本身等价于“本步完成的最后动作”，倾向 `continue_next_step`。
- **备注**：这条任务的 label 仍可能是弱监督，但比“永远选 i+1”更合理。

---

## 7. 工程落地注意事项（直接影响任务质量）

1) **彻底屏蔽泄漏源**：不要把 `step_slug`、文件名中的 `ts_XX.XXs`、`frame_###` 字符串写入模型可见 prompt；只在 `meta.evidence_files` 记录。
2) **三态是刚需**：对 precondition/effect/affordance 的核验任务，默认应该允许 `not observable`，否则会把不可观测命题硬判为 Yes/No，噪声极大。
3) **负样本“单一扰动”**：每条样本只注入一个错误（替换一个对象/翻转一个 truth/交换一个顺序），不要一次注入多错，避免无法归因。
4) **过滤低信息样本**：关键帧对几乎相同、遮挡严重、ts 相同、clip 无进展的样本应丢弃，否则任务变成猜谜。

