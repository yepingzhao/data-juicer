# CCMB — Domain Glossary

> Civilized City Multimodal Benchmark（文明城市多模态评测基准）

## Core Entities

- **巡查记录 (Inspection Record)**: 一条原始数据，包含现场照片、问题描述、点位类型、检查标准。共 44,367 条原始记录，经规则清洗后 39,039 条候选。
- **点位类型 (Point Type / `pt_type`)**: 巡查地点类别，共 50 类（如"居民小区"、"农贸市场"、"主次干道/商业大街"）。
- **检查标准 (Inspection Standard / `insp_std`)**: 判定违规的依据类别，共 8 类（如"公共环境"、"公共秩序"、"基础设施"、"公益宣传"）。
- **问题描述 (Problem Description / `prob_desc`)**: 巡查员对现场问题的简短中文描述，中位数约 10 字。

## Design Decisions (in progress)

### Strategy
- **Local-first, cloud-later**: 初期全部在 2× RTX 4090 24GB 本地环境运行；后期引入中国云端大模型 API 扩展能力边界。
- **All 4 tasks retained**: 任务一至三初期本地评估，任务四（智能体）仅在后期云端 API 阶段评估。

### GT Pipeline
- **Generator**: Qwen3-VL (8B class) — 主标注者，生成完整结构化标注
- **Verifier**: InternVL3.5 (8B class) — 校验者，审核纠错
- **Arbiter**: Human — 双模型冲突时人工介入裁决

### Architecture
- **data-juicer**: Phase A 预处理+抽样 + Phase C 导出
- **DataFlow**: Phase B GT 生成+校验 (利用 vLLM serving 批处理和多模型级联)

### Model Constraints (2× 4090 24GB)
- Max practical model size: ~8B parameters (~18-20GB VRAM)
- 72B+ models unavailable locally

## Project Status

- Phase 1 in progress: preprocessing pipeline built, 39,039 candidates, 5-sample tiny test set
- Benchmark definition under redesign (2026-06-08)

## Open Terms

> Terms to be resolved during grilling sessions.
