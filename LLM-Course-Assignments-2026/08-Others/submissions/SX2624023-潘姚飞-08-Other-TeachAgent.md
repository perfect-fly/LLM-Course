# 选题方案（定稿）：TeachAgent —— 面向高中数学的多角色教学智能体

> 课程：《大模型原理与技术》（2026 秋）
> 方向：**08-Other（自由选题 · AI+教育）**
> 学号 / 姓名：SX2624023 / 潘姚飞
> 定位：做成一个**完整可用的 Agent 系统**，并具备可发论文的创新点；与导师的 paper 内容解耦，成果可后期互补复用。
> 算力条件：云 GPU（AutoDL / Colab / Kaggle）

---

## 一、题目

**中文**：TeachAgent：基于 Skill 编排与认知诊断的高中数学教学智能体
**英文**：TeachAgent: A Multi-Agent Math Tutoring System with Skill Orchestration and Cognitive Diagnosis

## 二、选题一句话

面向高中数学（人教 A 版范围），构建一个会**先判断教学意图、再用苏格拉底式追问引导、能用 sympy 校验解答、能记录学情并自适应出题**的多角色教学智能体；学生端可直接提问/上传题目，教师端可看学情看板。

## 三、为什么选这个题（08-Other 硬性要求：选题理由）

1. **真实痛点**：高中数学答疑高度依赖教师一对一；通用大模型倾向"直接甩答案"，学生抄完不会，且数学推理易出错。
2. **与课程方向一致**：2026 培养方案明确把 **AI+教育** 列为 08-Other 鼓励方向；教育智能体同期的 08 提交中无人做。
3. **与导师 paper 解耦**：本作业独立成系统，不依赖导师具体研究内容；做好后其评测协议、数据、Agent 框架可作为 paper 的补充实验或工程基础。
4. **可评测**：教学能力有 MathTutorBench 公开基准与排行榜可比；数学正确性有 CMATH/Math23K 可测。
5. **技术覆盖全**：Agent（多角色/工具/Skill）+ RAG（教材教辅）+ 微调（LoRA/DPO）+ 评估 + 部署，满足 08-Other "大模型核心技术全流程"要求。

## 四、相关工作与基准（08-Other 硬性要求）

| 工作 / 基准 | 关键点 | 与本方案的关系 |
|---|---|---|
| [MathTutorBench](https://github.com/eth-lre/mathtutorbench)（[EMNLP 2025](https://aclanthology.org/2025.emnlp-main.11/)） | 把教学能力拆成 Math Expertise / Student Understanding / Pedagogy 三类 9 任务，自动评测 + 排行榜 | **主评测基准**；本方案做公平对比 |
| [SocraticLM](https://arxiv.org/abs/2410.03017)（NeurIPS 2024） | 苏格拉底式数学教学，引导而非给答案 | 教学策略基线；本方案把它 Skill 化并可插拔 |
| [LLM Agents for Education（EMNLP 2025 Findings）](https://aclanthology.org/2025.findings-emnlp.743/) | 教育 Agent 的分类与能力综述 | 定位创新点 |
| [A Comprehensive Survey on LLM-Based Agents for Education（IEEE 2025）](https://xplorestaging.ieee.org/document/11192789) | 教育 Agent 架构综述 | 架构参考 |
| [Unifying AI Tutor Evaluation（NAACL 2025）](https://aclanthology.org/2025.naacl-long.57/) | AI 导师教学能力评测分类 | 自建中文 rubric 的依据 |
| [Agent-orchestrated RAG for Personalized Learning（IEEE 2025）](https://ieeexplore.ieee.org/document/11566356) | Agent 编排 RAG 做个性化学习 | 检索层参考 |
| [Multi-Agent Collaborative Framework for Personalized Teaching（ACM 2025）](https://dl.acm.org/doi/full/10.1145/3797552.3797712) | 多智能体个性化教学 | 用 Skill 编排替代硬编码角色 prompt |

## 五、系统设计

```
学生输入（文字题 / 图片题 / 代码式步骤）
        │
        ▼
┌──────────────────────────────┐
│ 教学规划 Agent (Router/Planner) │  意图：概念讲解 / 解题引导 / 错题诊断 / 出题 / 复习
└──────────────┬───────────────┘
               │
   ┌───────────┼────────────┬────────────┐
   ▼           ▼            ▼            ▼
[讲解 Agent] [苏格拉底 Tutor] [评测 Agent] [出题 Agent]
  RAG教材     提示阶梯        sympy校验     知识点/难度
  引用溯源    不直接给答案    步骤检查       自适应
   └───────────┴────────────┴────────────┘
               │
               ▼
      [学情 Profiler]  知识图谱 + 掌握度(BKT) + 错题本
               │
               ▼
   Skill 层：讲解 / 追问 / 提示 / 举例 / 验证 / 鼓励 / 出题
               │
               ▼
   记忆：对话摘要 + 长期学生画像（SQLite）
               │
               ▼
   Gradio：学生端（对话/上传题目/步骤引导/错题本）
           教师端（学情看板/薄弱点分布）
```

**核心模块**
1. **教学规划 Agent**：分类意图，决定"直接讲解 / 引导 / 出题 / 复习"，避免所有问题都走同一条链。
2. **苏格拉底 Tutor Agent**：实现**提示阶梯（Hint Ladder）**——启发式提问 → 关键提示 → 部分步骤 → 完整解答，由学情决定从哪一级开始。
3. **解题验证 Agent**：用 `sympy` 做符号计算、方程求解、化简验证；对分步解答做"每步是否等价"检查，专门治数学幻觉。
4. **RAG 讲解 Agent**：教材/教辅/例题分块入库（BGE-M3 + Chroma），回答带知识点与教材出处；证据不足则拒答。
5. **学情 Profiler**：知识点图谱（函数/三角/数列/立体几何/解析几何/概率统计/导数）+ 掌握度估计 + 错题本。
6. **Skill 层**：每个教学能力封装为 `SKILL.md`（name/description/正文），按需加载、可版本管理、可消融。

## 六、数据方案（中文高中数学）

| 用途 | 数据 | 规模 | 说明 |
|---|---|---|---|
| 知识库 | 人教 A 版教材/讲义、高考真题解析 | 千级文档 | 需自行整理/爬取，注意版权仅本地研究用 |
| 解题训练/评测 | [Math23K](https://arxiv.org/html/2009.11506v1) | 2.3 万题 | 中文数学应用题，带解 |
| 解题训练/评测 | Ape210K | 21 万题 | 中文数学应用题，CC BY 4.0 |
| 中文评测 | CMATH / MathBench | 千级 | 覆盖小初高，做中文泛化测试 |
| 教学对话 | LLM 合成 + 人工抽检 | ≥5k 轮 | 以教材例题为锚点合成苏格拉底对话 |
| 教学能力评测 | [MathTutorBench](https://github.com/eth-lre/mathtutorbench) | 公开 | 9 个任务，可直接跑 |
| 教学偏好数据 | [pedagogical-rewardmodel-data](https://huggingface.co/datasets/dmacjam/pedagogical-rewardmodel-data) | 公开 | 用于 DPO/奖励模型 |

## 七、技术路线

1. **RAG 层**：教材/例题 → 语义分块（保留知识点标签）→ BGE-M3 向量化 → Chroma；检索 top-k + rerank；回答附知识点与原文出处。
2. **Agent 编排**：LangGraph（或自研状态机）实现"规划→路由→执行→反思"；每个角色可单独开关，便于消融。
3. **Skill 机制**：把讲解/追问/提示/出题等写成 `SKILL.md`，渐进式披露；与 lab1 的 Skill 内容直接打通。
4. **数学工具**：`sympy` 求解与等价校验；可选接入公式 OCR（拍照搜题）。
5. **微调**：
   - SFT：在合成的苏格拉底教学对话上做 LoRA（Qwen2.5-7B-Instruct，4bit）。
   - 进阶：用 MathTutorBench 的 pedagogical reward model 做 DPO，直接优化"教学性"。
6. **记忆**：对话摘要 + 学生画像（知识点掌握度、错题、偏好）。
7. **部署**：Gradio（流式、引用侧栏、代码/公式渲染），学生端 + 教师端双视图。

## 八、评估方案

**① 教学能力（主）**：MathTutorBench 9 任务
- Math Expertise：Problem Solving、Socratic Questioning
- Student Understanding：Solution Correctness、Mistake Location、Mistake Correction
- Pedagogy：Scaffolding Win Rate、Pedagogy IF Win Rate（含 hard 版）

**② 数学正确性**：CMATH / Math23K 测试集准确率，对比原生模型。

**③ 自建中文教学评测**：200 条人工标注（引导性、正确性、不泄题、鼓励性、知识点贴合），双人标注 + Kappa。

**④ 学习增益（加分）**：同学志愿者小规模前后测，实验组 TeachAgent vs 对照组直接看答案。

**⑤ 消融实验**：Base → +RAG → +多角色/Skill → +SFT → +DPO。

**⑥ 效率**：单轮时延、token 消耗、显存占用。

## 九、创新点（论文向）

1. **Skill 化教学策略编排**：把教学策略从硬编码 prompt 升级为可插拔、可消融的 Skill 单元，并支持渐进式披露。
2. **认知诊断驱动的自适应提示阶梯**：根据学生掌握度动态决定"从哪个提示级别介入"，而非固定策略。
3. **双通道可信数学推理**：LLM 负责表达与教学，sympy 负责形式化校验，降低数学幻觉并给出可验证步骤。
4. **中文高中数学教学评测协议**：对齐 MathTutorBench 的能力维度，补充中文高中场景的自建 rubric 与数据。
5. **可复现的完整 Agent 系统**：开源数据、代码、评测脚本与 Demo，便于同行复现。

## 十、里程碑（12 周）

| 周次 | 目标 | 产出 |
|---|---|---|
| W1–2 | 报名、仓库搭建、数据源跑通 | Issue 报名 + 独立仓库 + 数据脚本 |
| W3–4 | 教材/题库入库；单 Agent RAG 答疑 + sympy 校验 + 基线评测 | 基线 Demo + MathTutorBench 首轮结果 |
| W5–6 | 苏格拉底 Skill + 提示阶梯 + 引用/拒答 | v1 系统 + 指标对比 |
| W7–8 | 多角色编排 + 学情画像 + 合成教学数据 + LoRA | v2 系统 + 微调模型 |
| W9–10 | Gradio 双端部署、中文教学评测、学习增益小实验 | Demo 视频 + 评测报告 |
| W11–12 | DPO/消融、报告撰写、最终 PR | 最终报告 + 仓库链接 |

## 十一、交付物与 08-Other 规范

- 独立公开 GitHub 仓库：`data/`（脚本，不放受限语料）、`src/`（agents/rag/tools/memory）、`eval/`、`app/`、`README.md`、`requirements.txt`。
- 报告：`SX2624023-潘姚飞-08-Other-TeachAgent.md`（或 `.pdf`），必含选题理由、相关工作对比、创新点、基准比较。
- PR 标题：`[作业08-Other] SX2624023 潘姚飞 - TeachAgent 高中数学教学智能体`。

## 十二、风险与对策

| 风险 | 对策 |
|---|---|
| 多角色系统过重 | 先单 Tutor Agent MVP，再逐步拆角色；每步都可评测 |
| 教材/真题版权 | 仓库不放原文语料，只放处理脚本与公开数据集；本地研究使用 |
| 数学幻觉 | sympy 强校验 + 分步等价检查 + 拒答 |
| MathTutorBench 为英文 | 主评测用原版（保证可比），另做中文自建评测补充 |
| 云 GPU 成本 | 7B + 4bit LoRA；评测小样本先行，全量放夜间任务 |
| 与 paper 冲突 | 本系统独立成仓库；后期只复用评测协议/框架，不绑定 paper 内容 |
