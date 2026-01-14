最终核心严格落在：

- **因果规划（causal planning）**：显式状态/约束（spatial+affordance）→ 可行性 → 因果后果 → 跨步依赖 → 长时序规划（prefix/reorder/infill）
- **失败反思（failure reflecting）**：不一致检测 → 缺陷类型定位（flaw type）→ 恢复策略选择 → 失败驱动重规划（replanning）

### 0.1 四类证据（最终统一口径）

为兼容主规范的 `evidence_type` 枚举与训练落地的可控性，最终建议把证据形态压缩为 4 类（其余如 `images_uniform_clip` 视作工程实现细节）：

1) `keyframe_single`：单张关键帧
2) `images_uniform_scene`：全局视频均匀抽帧的多图
3) `video_clip`：两个step之间的局部视频片段
4) `video_prefix`：累积前缀片段

优先级最高的重复/低增益点：

- `Task_08` 与 `Task_17` 重叠：`Task_17` 已覆盖 Why/How（含机制/因果），建议将 `Task_08` 合并为 `Task_17` 子变体或显著降权。我的想法是删除task17，另外再看有没有别的任何使用跟how相关（mechanism）

- `Task_07` 与 `Task_23` 功能相近：有 `video_prefix` 时优先 `Task_23`；`Task_07` 作为无视频 fallback。删除任务7，修改任务23为直接给完整视频，推测high_level_goal。

- `Task_11` 与 `Task_19` 都围绕“效果”：若强调证据闭环与失败反思，优先 `Task_19`；`Task_11` 更偏计划字段复述，不宜做核心指标。删除任务19。

- `Task_20/21/25` 更偏辅助能力（边界/关键帧理由/进度总结），不建议进入核心指标或占比过高。删除任务20，21，25

### 6.1 新增任务 A：Spatial Postcondition Check（空间后置状态核验，Yes/No）

- 字段来源：`steps[i].spatial_postconditions_detail[*].relation/objects/truth`
- 证据：step i 尾关键帧（优先）或 step clip 抽帧
- 输出：`Yes/No`（exact match）
- 价值：把“动作→空间状态成立/不成立”的因果后果变成可评分监督（强于自由文本 `expected_effects`）

### 6.2 新增任务 B：Affordance Postcondition Check（可供性后置状态核验，Yes/No 或 MCQ）

- 字段来源：`steps[i].affordance_postconditions_detail[*].object_name/affordance_types/reasons`
- 证据：step i 尾关键帧
- 输出：
  - Yes/No：给定 object+affordance_type 判别是否成立
  - 或 ABCD：给定 object 选正确 affordance_type
- 价值：直接监督“动作使对象获得/保持某种可操作性”（可执行规划的核心）
### 6.3 新增任务 C：Failure-Driven Replanning / Recovery Insertion（失败驱动重规划，建议拆成两段客观题）

仅复述 `recovery_strategy` 不足以形成 failure reflecting 闭环；建议拆成两段可评分任务：

1) **Recovery Strategy Selection（ABCD）**  
   - 输入：失败描述 + 证据  
   - 输出：A/B/C/D（选择正确 `failure_handling.recovery_strategy`）

2) **Next Step After Recovery（ABCD 或排序）**  
   - 输入：已选 recovery + 证据（或仅失败描述）  
   - 输出：下一步 step_goal（或候选集合排序）  

### 6.4 Task_14/15 的推荐客观化变体（使其成为“可回归”的失败反思指标）

- `Task_14`：Counterfactual Outcome MCQ（ABCD）  
  - gold：`expected_challenge_outcome`
  - 干扰：来自其他 step/item 的 outcome
- `Task_15`：Recovery Strategy MCQ（ABCD）  
  - gold：`failure_handling.recovery_strategy`
  - 干扰：来自其他 step/item 的 recovery

task28可以用原始的多个step的plan，随机修改其中的一个step plan为一个错误的sub-plan

### 12.5 Task_10 / SS05（step_goal 匹配，ABCD，强监督）

- 证据（keyframe_single）：  
  `ECCV/causal_spafa_plan_dataset_long/P01_01_part1/03_wash_the_cucumber_and_carrot_in_the_sink_under_running_water/frame_020_ts_68.39s.jpg`
- 正确项：该 step 的 `step_goal`
- 干扰项：同 item 其他 3 个 `step_goal`
- 输出：只输出 `A/B/C/D`

### SS03_Temporal_Order_Check_AB（两事件先后判别；对应 Task_26 的更严格可评分版本）

- **证据**：`keyframe_pair`
- **字段来源**：
  - 时间戳：两张关键帧文件名中的 `ts_XX.XXs`
  - 事件文本（用于写题面）：`action_description` / `state_change_description`
- **标签**：`A`（A 更早）或 `B`（B 更早）
- **样本构造（推荐）**：
  1) 选两张关键帧图片（建议来自不同 step，或来自同 step 的 `critical_frames[0]` 与 `critical_frames[-1]`）。
  2) 解析 `ts_a`、`ts_b`，并保证 `ts_a != ts_b`；若相等则丢弃/重采样（重复帧常见于短视频或抽帧失败回填）。
  3) 为每张图生成一个短事件描述（A/B），并随机打乱呈现顺序；label 仅来自 `ts` 比较。
- **问答模板（示例）**：
  - Evidence A: `<IMAGE_PATH_A>`
  - Evidence B: `<IMAGE_PATH_B>`
  - Q: Which event happens earlier in the video, A or B?
  - A (label): `A` / `B`

### SS04_Tool_vs_Material_Check（工具/材料角色判别；对应 Task_04 的二分类变体）

- **证据**：`video_clip`（step clip；优先覆盖该 step 执行过程）
- **字段来源（JSONPath）**：
  - `steps[s].tool_and_material_usage.tools`
  - `steps[s].tool_and_material_usage.materials`
- **标签**：`Yes/No`
- **样本构造（推荐）**：
  1) 采样一个 step。
  2) 取该 step 的一个工具/材料候选名词 `x`，并尽量让问题“依赖看视频”：
     - 正样本：`x` 取自 `tools`（问 tool）或 `materials`（问 material）
     - 弱负样本：`x` 取自同 item 的其他 step（或交换 tool/material 身份），并写 `meta.neg_sample=true`
  3) 问法随机二选一：
     - “Is x a tool used by the agent in this clip?” → label = `x ∈ tools`
     - “Is x a material being acted on in this clip?” → label = `x ∈ materials`
- **问答模板（示例）**：
  - Evidence: `<VIDEO_CLIP_PATH>`
  - Q: In this clip, is `<x>` a tool used by the agent?
  - A (label): `Yes` / `No`
- **注意**：
  - 当 `tools` 为空时，建议在数据构造时补上 `hands`（与主规范一致），否则该任务会退化。

  ### SS05_Step_Goal_Matching_MC（关键帧对应 step_goal 四选一；对应 Task_10 的严格可评分变体）

- **证据**：`keyframe_single`（优先 `critical_frames[0]`）
- **字段来源（JSONPath）**：
  - 正确项：`steps[s].step_goal`
  - 干扰项：同一 item 内其他 step 的 `step_goal`
- **标签**：四选一 `A/B/C/D`
- **样本构造（推荐）**：
  1) 采样一个 step s，取其 `step_goal` 为正确项。
  2) 从同 item 的其他步骤中抽 3 条不同的 `step_goal` 作为干扰项（避免重复或高度同义）。
  3) 打乱选项顺序并写入 `meta.options`，label 为正确选项字母。
- **问答模板（示例）**：
  - Evidence: `<IMAGE_PATH>`
  - Q: Which step goal best matches what is happening in this image?
    - A) `<goal_a>`
    - B) `<goal_b>`
    - C) `<goal_c>`
    - D) `<goal_d>`
  - A (label): `A` / `B` / `C` / `D`

#### C) Task_09 与 Task_18

- **现象**：都聚焦 precondition，但 Task_18 是视觉核验（三态）。
- **问题**：Task_09 容易变成“复述 JSON 列表”，不要求证据闭环。
- **建议**：
  - 主线：**保留 Task_18**（三态核验更 grounded）。
  - Task_09：降权/删除；若保留，务必要求 “仅输出可观测的前置条件 + 不可观测必须标注 not directly observable”。
  我觉得应该将两个任务都删除。然后执行step的action的spatial和affordance的precondition然后多选题进行选择

  #### D) Task_11 与 Task_19

- **现象**：都围绕 effect/postcondition。
- **问题**：Task_11 偏“期望复述”，而 effect 往往不可见（clean/ready 等），噪声大。
- **建议**：
  - **保留 Task_19**（证据核验），Task_11 降权/删除。
  - 若要保留 Task_11：建议仅保留 spatial 类 effect（如 on_top_of/inside/open/closed），减少 affordance/属性类不可观测 effect。

  #### Task_06（Holistic Causal Chain）和 #### Task_16（Physical Feasibility）需要重新组织设置一下

### Task_31_Keyframe_to_StepGoal_Matching（关键帧→步骤匹配，MCQ/分类）

- **动机**：把“关键帧语义”与 “step_goal” 强绑定，且天然可做强负样本。
- **字段来源（JSONPath）**：
  - 正例：`steps[i].step_goal`
  - 候选池：同 item 的 `steps[*].step_goal`（或跨 item 扩展）
- **证据**：`keyframe_single`（选 `steps[i].critical_frames[0]` 或 `[-1]`）
- **构造规则**：
  - 输入：一张关键帧图 + K 个候选 step_goal（K=4/6）。
  - 输出：选项字母或 `step_id`（固定格式便于评分）。
- **负样本**：
  - 同 item 的其它 step_goal（hard negatives），或同场景同类动作的跨 item step_goal（harder

### Task_32_Init_vs_Complete_Keyframe_Order（同一步两关键帧阶段判别）

- **动机**：利用 Stage3 “两关键帧递增顺序” 的结构化监督，形成强时序/状态变化学习信号。
- **字段来源**：`steps[i].critical_frames[0]` 与 `[1]` 的图像（标签：initiation vs completion）。
- **证据**：`keyframe_pair`（两张关键帧图，顺序可打乱）
- **输出**：`A_is_initiation` / `B_is_initiation`（二分类）
- **注意**：不要暴露 `frame_index`。

### Task_33_Hotspot_AffordanceType_MCQ（热点可供性类别选择题）

- **字段来源**：`steps[i].critical_frames[j].interaction.hotspot.affordance_type`
- **证据**：`keyframe_single`
- **输出**：从 4 选 1 的 `affordance_type` 里选正确项
- **负样本构造**：
  - 来自其它关键帧的 affordance_type；或从同关键帧 hotspot.mechanism 语义近但类别不同的项做 hard negative。

### Task_34_Hotspot_Mechanism_MCQ（热点物理机制选择题）

- **字段来源**：`steps[i].critical_frames[j].interaction.hotspot.mechanism`
- **证据**：`keyframe_single`
- **输出**：4 选 1（机制句子，建议做短句标准化）
- **工程建议**：先把 mechanism 做轻量模板化（如 “force_transfer/friction/leverage/fluid_flow/heat_transfer/…” + 一句解释），否则自由文本难当 label。

### Task_35_Action_Phrase_MCQ（`causal_chain.action` 选择题）

- **字段来源**：
  - keyframe-level：`steps[i].critical_frames[j].causal_chain.action`
  - step-level：`steps[i].causal_chain.action`
- **证据**：`keyframe_single`（更 grounded）
- **输出**：4 选 1（动作短语）
- **价值**：补齐当前任务集中对 `action` 字段利用不足的问题。

### Task_36_Patient_Identification_MCQ（受事对象选择题）

- **字段来源**：`steps[i].critical_frames[j].causal_chain.patient`
- **证据**：`keyframe_single`
- **候选池**：来自 Task_01 的对象锚点集合（同 item 去重后的 objects/tools/materials）
- **输出**：多选或单选（取决于 patient 是否单实体）

### Task_37_Step_vs_Keyframe_CausalChain_Consistency（步级↔关键帧因果链一致性判别）

- **动机**：直接训练 “同一步的 step-level 因果链” 与 “关键帧级因果链” 的一致性，构造强负样本。
- **字段来源**：
  - 正例：`steps[i].causal_chain` 与 `steps[i].critical_frames[j].causal_chain`
  - 负例：把 `critical_frames[j].causal_chain` 替换为其它 step 的 causal_chain（或只替换 patient/action）
- **证据**：`keyframe_single` +（可选）step_goal
- **输出**：`consistent / inconsistent / not directly observable`
- **评分**：自动评分；负样本可控。

### Task_38_Spatial_Postcondition_Check（空间后置状态核验，Yes/No/Uncertain）

- **字段来源**：`steps[i].causal_chain.causal_effect_on_spatial[*].relation/objects/truth`
- **证据**：step i 的最后关键帧（或 step clip 抽帧）
- **输出**：对每条 effect 输出 `supported / contradicted / not observable`
- **价值**：比 Task_11 的“复述效果”更 grounded、更可评测。

### Task_39_Affordance_Postcondition_Check（可供性后置状态核验）

- **字段来源**：`steps[i].causal_chain.causal_effect_on_affordance[*].object_name/affordance_types`
- **证据**：step-end 关键帧（注意很多 affordance 不可见）
- **输出**：同 Task_38 三态
- **建议**：优先选择“可见状态类 affordance”（如 open/closed/inserted/holding），避免纯功能性不可见标签。

### Task_40_Counterfactual_Outcome_MCQ（反事实结果选择题）

- **字段来源**：`steps[i].counterfactual_challenge_question` 与 `expected_challenge_outcome`
- **证据**：关键帧（或 step clip 抽帧）
- **输出**：4 选 1 outcome
- **负样本**：来自其它 step 的 outcome，或相同工具/材料但机制不同的 outcome（hard negative）。

### Task_41_Recovery_Strategy_MCQ（失败恢复策略选择题）

- **字段来源**：`steps[i].failure_reflecting.recovery_strategy`
- **证据**：关键帧（或 prefix 到该步）
- **输出**：4 选 1 strategy
- **价值**：把失败反思从自由文本变成客观题，更可评测、更稳定。

### Task_42_Prefix_Completed_Steps_MultiSelect（前缀已完成步骤多选）

- **动机**：替代 Task_25 的开放式总结，用可评分的多选形式衡量“长时序进度理解”。
- **字段来源**：`steps[0..i].step_goal`（作为 gold；不要塞进输入）
- **证据**：`video_prefix`（到 step i 的尾帧）
- **输出**：从候选 step_goal 列表中选出“已完成”的集合（或输出最大已完成 step_id）。
- **负样本**：候选中混入未来 step_goal。

### Task_43_Stage2_Temporal_Localization_Check（可选：基于 Stage2 的客观时间定位）

仅当 item 内存在可读的 Stage2 产物（例如 stage2 的 JSON/manifest 可取）时启用：

- **字段来源**：`stage2` 预测的 `start_frame_index/end_frame_index`
- **证据**：全视频 `sampled_frames/`（或其抽帧子集）
- **输出**：预测/核验某一步的边界索引（更严格、更客观）

---

