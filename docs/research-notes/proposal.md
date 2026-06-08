# CCMB：面向文明城市创建的多模态 Benchmark

## 1. 研究动机

多模态大模型（MLLM）正越来越多地部署于真实城市治理场景中，然而现有 benchmark 主要聚焦于通用视觉理解（VQA、图像描述）或狭窄的学术任务。我们识别出一个关键空白：**尚无 benchmark 评估 MLLM 在城市治理巡查的端到端工作流中的能力**——从视觉感知，到基于标准的违规判定与归因推理，再到可操作的治理建议生成和智能体工具编排。

我们提出 **CCMB（Civilized City Multimodal Benchmark，文明城市多模态评测基准）**，基于全国文明城市创建过程中的真实巡查数据构建。该 benchmark 包含 5,000 条精选样本，覆盖 50 种点位类型和 8 类检查标准，通过四个互补的任务维度系统评估模型作为"AI 辅助城市治理巡查员"的综合能力。

## 2. 数据集概览

### 2.1 原始数据

原始数据集包含 44,367 条实地巡查记录，每条记录包含：

| 字段 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `image` | JPEG/PNG | 现场照片 | 楼道地面烟头照片 |
| `prob_desc` | 中文文本（中位数 10 字） | 简短问题描述 | "南门处地面存在垃圾" |
| `pt_type` | 分类标签（50 类） | 点位类别 | 居民小区、农贸(集贸) 市场、主次干道/商业大街 |
| `insp_std` | 分类标签（8 类） | 检查标准类别 | 公共环境、公共秩序、基础设施、公益宣传 |

经规则化清洗（剔除必填字段缺失和多图记录），保留 **39,039 条有效候选记录**。从中按 `pt_type` 分层抽样 **5,000 条**作为核心 benchmark 集，按 80/10/10 划分训练/验证/测试集，确保每种 `pt_type` 在测试集中至少出现 5 次。

### 2.2 核心集统计

| 指标 | 数值 |
|------|------|
| 总样本数 | 5,000 |
| 点位类型数 | 50 |
| 检查标准类别数 | 8 |
| 问题描述平均长度 | 10.4 字符 |
| 图片可用率 | 100% |
| 数据划分 | 训练 4,000 / 验证 500 / 测试 500 |

## 3. 任务设计

Benchmark 定义**四个互补的任务维度**：

### 任务一：多模态理解（C + D）

**子任务 1a — 视觉识别：** 给定一张图片，判断 `prob_desc` 描述的问题是否存在，输出二分类判定及定位依据。

**子任务 1b — 上下文分类：** 给定图片及其 `pt_type`，预测违反的 `insp_std` 类别。要求模型结合点位类型上下文来解释视觉证据。

**评估指标：** F1-macro（处理类别不平衡），生成描述的 BLEU/ROUGE-L。

### 任务二：思维链推理（A + C）

模型需输出**四步结构化推理链**：

| 步骤 | 名称 | 输入 | 输出 |
|------|------|------|------|
| 1 | 视觉感知 | 图像 | 观察到的事实列表（物体、行为、异常点） |
| 2 | 标准对照 | 事实列表 + `pt_type` + 领域知识 | 违规判定、匹配的标准条款及推理依据 |
| 3 | 归因分析 | 违规事实 + `pt_type` 上下文 | 结构化原因分类、置信度、因果链条 |
| 4 | 治理建议 | 原因 + 违规事实 + `pt_type` | 分级行动方案（即时/短期/长期）及责任主体 |

**评估指标：** 逐步骤准确率。步骤一用事实召回率（Fact Recall），步骤二至四用结构化字段 F1 值。四步分数等权平均。

### 任务三：治理建议生成

给定图片、`pt_type` 和违规判定结果，生成结构化治理响应：

- **根因分类**（设施缺失 / 保洁不足 / 居民行为 / 管理缺位）
- **紧急程度**（即时 / 短期 / 长期）
- **责任主体**（物业 / 社区 / 城管 / 商户）
- **行动方案**（各时间维度的具体措施）
- **预估成本与难度**

**评估指标：** 双轨评估——（1）结构化字段精确匹配 F1（权重 70%），（2）LLM-as-Judge 可行性打分 1-5 分（权重 30%）。

### 任务四：智能体工具使用与自我纠错

**子任务 4a — 工具编排：** 模型扮演巡查智能体，可调用 5 个工具：

| 工具 | 功能 | 说明 |
|------|------|------|
| `observe(image)` | 视觉感知 | 检测图像中的物体、行为和异常 |
| `search_standard(query, pt_type)` | 知识检索 | 从领域知识库检索匹配的检查标准条款 |
| `match_violation(observations, standards)` | 标准匹配 | 将观察事实与标准对照，判定违规 |
| `infer_cause(violations, context)` | 归因分析 | 从违规事实和点位上下文推断根因 |
| `generate_notice(violations, cause, pt_type)` | 通知生成 | 生成结构化整改通知书 |

**子任务 4b — 自我纠错：** 模型给出初步判定后，接收巡查监督员反馈（如"烟头问题漏判了"或"原因分析过于泛化"），需对输出进行修正。

**评估指标：** 最终输出质量（60%），工具选择准确率（25%），自我纠错改善率（15%）。

### 综合排名

采用**场景加权聚合 + 雷达图可视化**。定义三个典型使用场景，各任务权重不同：

| 场景 | 任务一 | 任务二 | 任务三 | 任务四 |
|------|--------|--------|--------|--------|
| 自动化巡查 | 40% | 35% | 15% | 10% |
| 辅助决策 | 15% | 30% | 35% | 20% |
| 全功能 AI 巡查员 | 15% | 25% | 25% | 35% |

各子任务分数归一化至 0-1 区间。雷达图支持逐维度横向对比，避免强制产出单一"第一名"。

## 4. 领域知识库

基于《全国文明城市测评体系》及相关城市管理条例等公开政府文件，构建结构化领域知识库。

### 结构

```
domain_knowledge/
├── standards.json              # 检查标准 → 条款明细、检查要点、扣分规则
├── point_type_matrix.json      # pt_type × insp_std 适用范围矩阵
├── rectification_library.json  # 问题类型 → 分级整改措施
└── regulations/                # 原始法规文件（引用溯源）
```

#### `standards.json` 示例

```json
{
  "公共环境": {
    "source": "《全国文明城市测评体系》相关条款",
    "description": "保持公共区域环境卫生整洁",
    "check_points": [
      "地面无垃圾、杂物、烟头",
      "墙面无乱涂乱画、无污渍",
      "绿化带无隐藏垃圾",
      "垃圾桶外观整洁、无满溢"
    ],
    "deduction_rules": "每发现一处扣0.5分",
    "applicable_point_types": ["居民小区", "主次干道/商业大街", "背街小巷", "农贸(集贸) 市场"],
    "examples": [
      {"description": "楼道地面有烟头", "severity": "minor"},
      {"description": "小区广场散落大量生活垃圾", "severity": "severe"}
    ]
  }
}
```

#### `point_type_matrix.json` 示例

```json
{
  "居民小区": {
    "applicable_standards": ["公共环境", "公益宣传", "基础设施", "文明行为"],
    "key_concern_areas": ["楼道", "电梯", "垃圾投放点", "小区出入口"],
    "typical_problems": ["楼道堆物", "垃圾未分类", "公益广告破损"],
    "responsible_parties": ["物业公司", "业委会", "社区居委会"]
  }
}
```

#### `rectification_library.json` 示例

```json
{
  "公共环境_烟头": {
    "immediate": ["安排保洁人员现场清理"],
    "short_term": ["增设吸烟点和烟蒂收集器", "在重点区域张贴禁烟提示"],
    "long_term": ["纳入社区文明公约", "建立保洁巡查考核制度"],
    "responsible": ["物业公司", "社区居委会"],
    "estimated_cost": "low",
    "typical_difficulty": "easy"
  }
}
```

### 构建方式

1. **LLM 生成骨架：** GPT-4o、Claude、Qwen-Max 三个模型各自独立生成 8 条标准 × 50 种点位的结构化知识内容。
2. **一致性校验：** 交叉比对三份输出，标记低一致性条目。
3. **终端验证：** 对 200 条样本进行端到端 GT 正确性人工检验，作为独立质量校验集。
4. **诚实声明：** 明确声明知识库为 LLM 衍生的"银标准"，非权威法律参考。版本化管理，支持社区贡献和勘误。

## 5. Ground Truth 构建

### 5.1 多模型投票流水线

四个任务的 GT 均通过三模型投票流水线生成：

| 角色 | 模型 | 职责 |
|------|------|------|
| 标注者 1 | Qwen2-VL-7B-Instruct | 独立标注 |
| 标注者 2 | Qwen2-VL-72B-Instruct | 独立标注 |
| 标注者 3 | GPT-4o | 独立标注 |
| 仲裁者 | GPT-4o（独立调用） | 裁决开放文本字段冲突 |

- **结构化字段**（`is_violation`、`primary_cause`、`severity` 等）：取多数投票；连续值取中位数。
- **开放文本字段**（`rationale`、`causal_chain`、行动描述等）：仲裁模型从三份候选中择最优或融合生成。
- **低置信度条目**（无多数或仲裁置信度 < 0.7）：标记待人工复核。

### 5.2 MLLM 富集 JSON Schema

MLLM 富集阶段使用统一的 JSON Schema prompt，指导模型输出结构化标注：

```json
{
  "visual_perception": {
    "scene_description": "图像整体场景描述（1-2句）",
    "observations": [
      {
        "id": "obs_1",
        "object": "烟头",
        "location": "3栋2单元一楼地面",
        "condition": "散落",
        "severity": "minor"
      }
    ],
    "uncertainties": ["光线较暗，墙角杂物难以判断类型"]
  },
  "violation_judgment": {
    "is_violation": true,
    "matched_standards": [
      {
        "standard_name": "公共环境",
        "clause": "保持公共区域清洁，无垃圾杂物",
        "confidence": 0.95,
        "rationale": "地面存在烟头属于垃圾杂物，违反清洁要求"
      }
    ],
    "violation_severity": "moderate",
    "missed_in_original": []
  },
  "root_cause_analysis": {
    "primary_cause": "居民卫生习惯不足",
    "primary_confidence": 0.8,
    "alternative_causes": [
      {"cause": "保洁频次不足", "confidence": 0.3}
    ],
    "contributing_factors": [
      "楼道未设置垃圾桶",
      "缺乏禁烟标识"
    ],
    "causal_chain": "居民习惯性随手丢弃 → 保洁未能及时发现清理 → 形成持续性卫生问题"
  },
  "governance_recommendations": {
    "immediate_actions": [
      {"action": "安排保洁清理楼道烟头", "responsible": "物业公司", "deadline": "24小时内"}
    ],
    "short_term_actions": [
      {"action": "增设吸烟点标识和垃圾桶", "responsible": "物业公司", "deadline": "1周内"},
      {"action": "开展入户文明宣传", "responsible": "社区居委会", "deadline": "2周内"}
    ],
    "long_term_actions": [
      {"action": "将禁烟纳入小区文明公约", "responsible": "业委会", "deadline": "1个月内"}
    ],
    "expected_difficulty": "easy",
    "estimated_cost": "low"
  },
  "agent_trajectory": {
    "suggested_tool_sequence": [
      "observe",
      "search_standard",
      "match_violation",
      "infer_cause",
      "generate_notice"
    ],
    "key_decision_points": [
      {"step": "match_violation", "decision": "烟头属于'公共环境'而非'文明行为'"},
      {"step": "infer_cause", "decision": "排除设施缺失，判定为行为习惯问题"}
    ]
  }
}
```

### 5.3 质量保障

- **必检字段：** `violation_judgment.is_violation` 和 `root_cause_analysis.primary_cause` 为核心决策点，必须人工抽检。
- **银标准定位：** 论文中明确 GT 为"银标准"——多模型共识自动生成，非人工标注。200 条验证集提供经验质量估计，与 benchmark 结果一并报告。
- **透明报告：** 公开标注者间一致率、仲裁比例、人工验证通过率。

## 6. 智能体环境

### 6.1 工具实现

五个工具均有实际 API 后端支撑：
- `observe`：封装 MLLM 视觉感知步骤
- `search_standard`：基于嵌入向量在 `standards.json` 上做语义检索
- `match_violation`：规则匹配 + 嵌入相似度双重判定
- `infer_cause`：LLM 结构化输出，以 `rectification_library.json` 为锚定
- `generate_notice`：模板 + LLM 填充

### 6.2 自我纠错场景构建

错误轨迹通过以下方式生成：
1. **弱模型生成：** Qwen2-VL-2B 生成初始（易出错的）输出，其天然存在漏判和误分类。
2. **规则注入：** 注入 1-2 个典型错误（如删除某个违规项、替换原因类别），确保每个场景至少有一处明确的纠正机会。
3. **反馈模板：** 监督员反馈模板针对特定错误类型设计（"漏检"、"标准误判"、"原因过于泛化"）。

## 7. data-juicer 集成

Benchmark 基于 [data-juicer](https://github.com/modelscope/data-juicer) 框架构建和交付：

| 角色 | 说明 |
|------|------|
| **预处理引擎** | 通过 data-juicer OPs 进行规则过滤（文本长度、语言识别、图片尺寸/宽高比） |
| **MLLM 富集引擎** | 通过 `mllm_mapper` OP 执行 JSON Schema 富集，支持 Qwen2-VL 及其他模型 |
| **Baseline Runner** | 标准化推理流水线，支持 GPU 调度和批处理 |
| **数据集导出** | 提供 HuggingFace `datasets` 格式，支持脱离 data-juicer 独立使用 |

## 8. 评估协议

### 8.1 各任务评估指标

| 任务 | 主要指标 | 辅助指标 |
|------|---------|---------|
| 多模态理解 | F1-macro（分类），BLEU/ROUGE-L（生成） | — |
| 思维链推理 | 逐步骤结构化字段 F1 | 事实召回率（步骤一），推理质量评分（仲裁模型打分） |
| 治理建议生成 | 结构化字段精确匹配 F1（70%）+ LLM-Judge 可行性评分（30%） | — |
| 智能体工具使用 | 最终输出正确性（60%）+ 工具选择 F1（25%）+ 纠错改善率（15%） | 任务完成效率（辅助报告，不计入排名） |

### 8.2 Baseline 模型

| 类别 | 模型 |
|------|------|
| 开源 MLLM | Qwen2-VL (2B/7B/72B)、InternVL2、LLaVA-NeXT、DeepSeek-VL2 |
| 闭源 MLLM | GPT-4o、Claude 3.5 Sonnet、Gemini 2.0 Flash |
| 智能体框架 | ReAct、Reflexion（作为任务四的 baseline） |

## 9. 伦理与局限性

- **隐私保护：** 原始数据为公共空间拍摄的巡查照片，按原始巡查数据集原样使用。样本级元数据（ID、坐标等）已做匿名化处理。
- **银标准声明：** Ground Truth 由 LLM 自动生成，承载标注模型的固有偏差。我们将透明报告标注者间一致率和人工验证率。
- **领域特异性：** Benchmark 面向中国城市治理场景，不声称对其他语言或法规体系的泛化能力。
- **知识库局限性：** 领域知识库由 LLM 基于公开政府文件综合生成，非法律专家撰写。知识库版本化管理，接受社区修正。
- **数据偏差：** 原始数据存在天然类别不平衡（`pt_type` 和 `insp_std` 分布不均），评估采用加权指标予以缓解。

## 10. 执行计划

| 阶段 | 周期 | 交付物 |
|------|------|--------|
| **Phase 1: 基础设施** | 1 周 | 5,000 条分层抽样核心集；JSON enrichment prompt 调优验证；领域知识库 v0.1 |
| **Phase 2: GT 生成** | 2 周 | 三模型投票流水线输出；仲裁融合标注；200 条人工验证集 |
| **Phase 3: 智能体环境** | 1 周 | 5 工具 API 实现；错误轨迹生成；自我纠错场景组装 |
| **Phase 4: Benchmark 组装** | 1 周 | HuggingFace datasets 打包；data-juicer pipeline 配置；评估脚本；使用文档 |
| **Phase 5: Baseline 与论文** | 2 周 | 5-8 个 MLLM baseline 结果；分析与消融实验；论文初稿 |
| **合计** | **约 7 周** | Benchmark v1.0 + 论文初稿 |

## 11. 贡献

1. **CCMB Benchmark：** 首个面向文明城市创建治理的多模态 benchmark，覆盖视觉感知、思维链推理、治理建议生成和智能体工具使用四个维度。
2. **领域知识库：** 基于公开政府文件的 LLM 合成结构化知识库，透明声明来源与局限性，支持版本化演进。
3. **多模型投票 GT 流水线：** 可复现的银标准标注管线，通过多模型共识与人工验证提供可量化的质量指标。
4. **data-juicer 集成：** 展示 data-juicer 作为领域化多模态 benchmark 构建端到端框架的能力，覆盖预处理、富集到 baseline 评估全流程。

---

*由 [Claude Code](https://claude.com/claude-code) 辅助撰写*
