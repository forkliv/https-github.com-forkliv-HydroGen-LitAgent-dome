# HydroGen-LitAgent  GOAI世界人工智能开源大赛AI4S 复赛入围项目

**氢能储运装备材料文献调研与构效关系发现系统** — 多 Agent 智能体（8 角色 LangGraph/DAG 工作流）＋ 路线 A 搜索优化（GA/BO/MCTS + LLM 深度融合）

> 项目：HydroGen LitAgent + 路线A ｜ 完成状态：T1–T15 全部交付（15/15）
> 官方研究报告：`docs/research_report_final.md`（约 1.3 万字，7 章 + 3 附录）
> 复赛（R2/AI4R）当前进度：见 `docs/progress-2026-09-02.md`（15/17 AI 侧完成，2 项待线下数据）

---

## 一、项目简介

面向氢能储运装备（储氢容器、低温管道、高压气瓶等）材料文献调研与构效关系发现的端到端智能体系统，由 **8 个角色 Agent** 协作完成：任务规划、文献检索、文献筛选、PDF解析与知识抽取、跨文献知识融合、Research Gap 识别、证据核验、报告生成；路线 A 另含材料数据库对接、搜索/优化算法（GA/BO/MCTS）与 LLM 深度融合、构效关系挖掘与交叉验证。

**关键能力**

- 混合检索：Sciverse API 语义检索 + Sci-Base 本地关键词检索（16 个领域概念 token 同义词映射）+ 三阶段筛选（Embedding 粗筛 / Reranker 精排 / 规则过滤）
- 知识抽取：MinerU PDF 解析 + LLM Schema 抽取（无 key 自动降级规则抽取）+ D01–D20 gold 评估（字段级 F1 0.975）
- 跨文献融合：实体规范化、单位统一、知识图谱构建、矛盾/缺失检测
- Research Gap 识别（七要素判定）+ 证据核验（零虚假引用，15/15 PASS）
- 路线 A：遗传算法 / 贝叶斯优化 / MCTS 三种搜索算法与 LLM 四融合（种子生成 / 合理性评估 / 剪枝 / 轨迹解读）
- 端到端编排：12 节点 DAG（T3→T12），内置等价 DAG 引擎（不依赖 LangGraph 也可运行）

**实测指标（T14 正式系统评测）**

| 维度 | 指标 |
|---|---|
| 检索质量 | 全量召回 E1/E2/E3 均 1.000；P@10 0.320/0.330/0.330；R@10 0.925/0.975/0.975；nDCG@10 0.873/0.955/0.948 |
| 抽取质量 | 字段级 F1 0.975；数值匹配 1.000；隐含覆盖 1.000 |
| Gap 识别 | E4 vs E5：覆盖率 0.938 vs 0.500；证据完整率 1.000 vs 0.500 |
| 效率 | 检索评估实测 3.924s（10 查询×3 模式）；heuristic 后端零 API 成本 |

---

## 二、架构

```
                         ┌──────────────────────────────────────────┐
  用户需求 ──► 任务规划 ─►│             12 节点 DAG 编排（T13）       │
                         │  T3 检索筛选 ─► T4 知识抽取 ─► T5 跨文献融合│
                         │        ─► T6 Gap识别 ─► T7 证据核验       │
                         │        ─► T8 报告生成                     │
                         │  路线A：T9 数据库 ─► T10 搜索优化+LLM融合  │
                         │        ─► T11 构效挖掘 ─► T12 交叉验证    │
                         └──────────────────────────────────────────┘
                                │ 终态产物汇编（e2e_outputs/）
                                ▼
            knowledge_graph / gap_list / research_report /
            routeA_candidates / discoveries / evidence_chain
```

## 三、目录结构

```
HydroGen-LitAgent/
├── README.md                     # 本文件
├── env.template                  # 环境变量占位模板（复制为 .env 使用；严禁提交真实凭据）
├── requirements.txt              # 顶层依赖（含可选增强 extra）
├── modules/                      # 各任务模块代码
│   ├── retrieval/                # T3 检索与筛选（混合检索 + 三阶段筛选）
│   ├── extraction/               # T4 知识抽取（MinerU + LLM/Rule Schema 抽取）
│   ├── fusion/                   # T5 跨文献融合与矛盾检测
│   ├── gap/                      # T6 Research Gap 识别（七要素）
│   ├── verification/             # T7 证据核验（零虚假引用）
│   ├── report/                   # T8 报告生成
│   ├── routeA/                   # T10 搜索/优化算法 + LLM 融合（GA/BO/MCTS）
│   └── orchestrator/             # T13 12 节点 DAG 编排 + docker-compose
├── data/
│   ├── tasks/                    # 样例输入产物（T1–T12 轻量镜像，供 e2e reuse 演示）
│   └── e2e_outputs/              # 端到端终态产物样例（T13 实跑输出）
├── docs/
│   └── research_report_final.md  # 官方研究报告（T15）
└── scripts/
    └── make_pdf.sh               # 研究报告 PDF 转换脚本（见 §六）
```

## 四、快速开始

### 4.1 环境

- Python 3.10+（开发环境实测 3.11.15）
- 可选：LangGraph（`pip install langgraph`）——不安装时使用内置等价 DAG 引擎
- 可选：Docker + docker-compose（容器化部署，见 §五）

### 4.2 安装

```bash
# 基础依赖（标准库为主，增强依赖按需）
pip install -r requirements.txt
# 可选增强：Embedding/Reranker 模型与 LangGraph 引擎
pip install -r requirements.txt --extra-index-url ...   # 或按各模块 requirements 单独安装
```

### 4.3 环境变量

```bash
cp env.template .env     # 填写真实 API key（Sciverse/MinerU/LLM/MP 等）
# 不配置任何 key 时全部自动降级（mock/规则/启发式），系统仍可端到端运行
```

### 4.4 端到端运行（T13，默认 reuse 模式，零网络零 key）

```bash
cd modules/orchestrator
./run_e2e.sh                 # 或 python run_e2e.py
```

默认 **reuse 模式**：校验并复用 `data/tasks/` 样例产物，组装终态产物到 `outputs/`。
分步/断点/重跑模式：

```bash
python run_e2e.py --steps T5,T6
python run_e2e.py --from T9 --to T12
python run_e2e.py --rerun
python run_e2e.py --rerun-t10 --fe-base-min 0.5
```

### 4.5 各模块自测（不依赖外部 API）

```bash
cd modules/retrieval && python run_selftest.py    # T3：检索质量 Q1–Q10
cd modules/extraction && python run_selftest.py   # T4：解析/单位/规则抽取/评估
cd modules/fusion && python run_selftest.py       # T5：融合/矛盾检测
cd modules/routeA && python self_test.py          # T10：GA/BO/MCTS + LLM 融合（13 项断言）
```

## 五、Docker 部署（PostgreSQL / Milvus / Neo4j）

```bash
cd modules/orchestrator
cp ../../env.template .env        # 填写环境变量
docker compose up -d litagent-e2e
```

> 注：`docker-compose.yml` 面向编排器运行（挂载输入产物、输出终态产物）。
> PostgreSQL / Milvus / Neo4j 的完整服务编排见项目 T9/T13 交付说明（数据仓库依赖为可选）。

## 六、研究报告 PDF 生成

环境不支持 pandoc/LaTeX 时使用自带转换脚本：

```bash
bash scripts/make_pdf.sh
```

脚本逻辑（任一可用即成功）：
1. pandoc + xelatex（含 CJK 支持）：`pandoc docs/research_report_final.md -o research_report_final.pdf --pdf-engine=xelatex -V CJKmainfont="Noto Sans CJK SC"`
2. Python weasyprint：`python -m weasyprint docs/research_report_final.md research_report_final.pdf`
3. 若无转换工具：脚本输出失败提示与安装指引（`apt-get install pandoc texlive-xetex fonts-noto-cjk`）

## 七、零虚假引用与安全约定

- **零凭据入库**：所有 API key 只通过 `.env` 注入，仓库仅含 `env.template` 占位；禁止向聊天/仓库提交真实凭据
- **零虚假引用**：所有文献引用、指标、Gap 证据均要求可回溯（docId/sourceSpan/证据链）；`data/e2e_outputs/manifest.json` 提供全链路溯源
- **黑箱拒绝**：搜索/抽取/核验均要求可解释（理由字段、轨迹记录、降级链路显式标注 is_mock）

## 八、许可证与说明

本仓库为竞赛/科研项目交付物，代码按原样提供。各模块保留原任务编号（T3–T13）与负责人信息，详见各模块 README_T*.md。

---

*HydroGen LitAgent + 路线A · T1–T15 全链路交付 · 15/15 完成*
forkliv@163.com
