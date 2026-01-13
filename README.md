# Causal_Learner

本仓库是一个多模块工作区，主要包含两部分：

- `ECCV/`：面向长视频/多模态监督的数据生成与三阶段流水线（包含脚本、规范文档与校验工具）。
- `Qwen-PC/`：Qwen3‑VL 相关的 demo、微调框架、以及点云方向的 `PointLLM` 子项目。

更细的开发/贡献约定见根目录 `AGENTS.md`，以及子目录 `ECCV/AGENTS.md`、`Qwen-PC/AGENTS.md`。

## 目录结构

- `ECCV/three_stage/`：三阶段流水线实现与文档（`THREE_STAGE_PIPELINE.md` 等）。
- `ECCV/causal_spafa_plan_dataset*/`、`ECCV/generated_plans_output_*`：生成物/数据产出（如需发布数据集再纳入版本管理）。
- `Qwen-PC/qwen-vl-finetune/`：训练框架与脚本（`qwenvl/train/`、`scripts/`、`demo/`）。
- `Qwen-PC/PointLLM/`：PointLLMv2官方代码。

## 快速开始（常用命令）

### ECCV：三阶段流水线

```bash
python ECCV/three_stage/pipeline.py --input-video /abs/path/video.mp4
```
