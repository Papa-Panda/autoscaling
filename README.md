# Autoscaling — 知识图谱

用论文搭 autoscaling 的知识图谱，对标 6 年 hyperscaler 生产经验（分钟级 nowcasting 预测式扩缩容、SLO guardrail、压测验证），双向找 gap：
文献缺了什么（我知道的），我缺了什么（文献知道的）。

> 状态：2026-09-12 初始化。知识树 + 论文清单正在由 Gemini 3.8 Flash 生成（API 波动重试中），到位后落到 `KNOWLEDGE_MAP.md`。
> 范围：先做 autoscaling（需求预测 → 扩缩容算法 → SLO 感知控制 → 压测验证）。机房热工/PUE、通用调度等以后再扩展。

## 目录结构

- `KNOWLEDGE_MAP.md` — 知识树（topic → subtopic → 核心概念/数学，符号全部定义）+ 对比脚手架（文献共识 vs 我的经验待填）
- `reading-log.csv` — 论文阅读流水，列：`date,title,org,tags,one_line_takeaway,folder,link`
- `PAPER_TEMPLATE.md` — 单篇论文 NOTES 模板
- `{slug}/NOTES.md` — 单篇论文笔记（平铺，如 `day-01-2020-autopilot/`）

## 约定

- 与主 repo（post-training）一致：**不出现雇主标识**，经验部分脱敏写
- 数学 notation-first：每个符号第一次出现就定义（含义、单位、时间尺度）
- 论文信息未核实的标 `[待核实]`，不编造标题/venue/year
- 比较视角固定四个：actuation latency、非对称代价（欠配 vs 过配）、非平稳/突发流量、灰度预检

## 路线

1. 知识树 + 20–25 篇 autoscaling 论文清单（Gemini 生成，人工核验）
2. 按 topic 逐篇读，做 NOTES，填对比脚手架
3. 收敛：哪些经验值得写成 experience paper（USENIX ATC / EuroSys / SoCC 方向）
