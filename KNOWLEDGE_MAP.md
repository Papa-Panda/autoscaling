# Autoscaling 知识图谱

> 生成：2026-09-12，四路并行 browser research（需求预测 / 扩缩容算法 / SLO 感知 / 压测验证），每篇论文的标题·第一作者·venue·年份均经搜索核实，未核实处标 [待核实]。
> 待办：Gemini 3.8 Flash 交叉验证（API 恢复后补）。
> 约定：公式中每个符号第一次出现即定义（含义、单位、时间尺度）； `$ ... $` 行内公式前后留空格（GitHub 渲染要求）。

---

## Part 1 — 知识树

### Topic 1：计算需求预测 / Nowcasting

**核心概念**

- **Prediction horizon（预测跨度）** $H$ ：从当前时刻往前看多少分钟。硬约束： $H \geq L_{act}$ （ $L_{act}$ = 执行延迟，见 Topic 2），否则预测再准也永远慢一拍。
- **点预测 vs 概率预测**：点预测给一个数（"下 5 分钟要 800 核"）；概率预测给分布/分位数（"P50=800, P90=950, P99=1100"）。容量决策真正需要的是分位数，不是均值。
- **Asymmetric loss（非对称代价）**：欠配（SLO 崩）代价灾难级，过配（多花几分钟算力钱）代价很小。评估预测器不能只看 RMSE/MAPE，必须看高分位点覆盖率。

**核心公式**

1. 预测问题设定： $\hat{y}_{t+1:t+H} = f(y_{1:t}, x_{1:t+H})$ 。 $y_t$ 为 t 时刻观测需求（如第 t 分钟 CPU 核数，单位：核）； $x$ 为协变量（星期几、大促排期，无量纲/one-hot）； $t$ 为当前时刻索引（分钟粒度）； $H$ 为 horizon（分钟）； $\hat{y}$ 为预测值。小例子：现在 14:00， $H=5$ ，预测 14:01–14:05 每分钟需求。

2. 分位数损失 pinball loss： $L_q(y, \hat{y}) = q \cdot \max(y-\hat{y}, 0) + (1-q) \cdot \max(\hat{y}-y, 0)$ 。 $q \in (0,1)$ 为目标分位点， $y$ 真实值， $\hat{y}$ 为模型输出的 q 分位数。小例子：真实 100 核， $q=0.9$ ：预测 90 → 损失 $0.9 \times 10 = 9$ ；预测 110 → 损失 $0.1 \times 10 = 1$ 。欠预测被罚 9 倍——"宁可多给"的数学表达。

3. 容量决策 = 报童模型： $c^* = F^{-1}\left(\frac{C_{under}}{C_{under}+C_{over}}\right)$ 。 $c^*$ 为最优预留容量（核）； $F$ 为需求分布 CDF（概率预测给出）； $C_{under}$ 为欠配单位代价（\$/核/分钟，含 SLO 违约）； $C_{over}$ 为过配单位代价（\$/核/分钟）。小例子： $C_{under}=1000, C_{over}=1$ → 临界分位 $\approx 0.999$ → 按 P99.9 备容量。**这是概率预测→扩缩容决策的桥梁。**

4. 时效硬约束： $H \geq L_{act}$ 。 $L_{act}$ 为执行延迟（分钟）：镜像拉取 + 启动 + 预热 + 健康检查。小例子： $L_{act}=5$ 分钟而 nowcaster 只预测未来 1 分钟 → 决策永远晚 4 分钟。

### Topic 2：预测式与反应式 Autoscaling 算法

**核心概念**

- **Reactive vs Predictive**：反应式等指标超阈值才扩容，决策滞后；预测式外推未来 $H$ 分钟需求，提前 $L_{act}$ 下单。小例子：流量 10:00 爬坡、10:05 到峰，VM 启动 4 分钟——反应式 10:03 触发、10:07 就绪（峰值已过且中间 SLO 受损）；预测式 09:58 预测到峰值立即扩容，10:02 就绪。
- **Actuation latency（执行延迟）** $L$ ：决策下达到实例 ready 接流量的时间。容器数十秒～2 分钟，VM 数分钟～十几分钟。**预测式存在的根本理由： $H \geq L$ 。**
- **Stability / Oscillation（抖动）**：副本数短时间内反复加减。根因：反馈延迟（ $L$ 大时控制器看到过期信号）、阈值过紧、预测噪声。代价：实例反复启停烧钱、预热实例拉低服务质量、触发下游限流。
- **Cooldown / Stabilization window（冷静期）**：一次动作后 N 分钟内忽略反方向决策（K8s HPA 缩容默认 300 秒取窗口内 desired 最大值）。本质是给执行延迟"让子弹飞一会儿"，防决策打架。

**核心公式**

1. HPA 阈值公式（K8s 官方）： `desiredReplicas = ceil[currentReplicas × (currentMetricValue / desiredMetricValue)]` 。currentReplicas 为当前副本数（整数）；currentMetricValue 为各副本指标均值（如 CPU %）；desiredMetricValue 为目标值。小例子：4 副本、当前 80%、目标 50% → ceil(6.4) = 7。纯比例式对噪声无抵抗，故另配 tolerance（默认 10%）+ stabilization window。

2. 指数平滑预测器（Autopilot 系 ensemble 基本单元）： $\hat{s}_{t+1} = \alpha \cdot x_t + (1-\alpha) \cdot \hat{s}_t$ 。 $x_t$ 为 t 时刻观测用量（核/GB，步长如 5 分钟）； $\hat{s}_t$ 为平滑估计； $\alpha \in (0,1)$ 为平滑系数（大→跟随快但抖，小→平滑但滞后）。Autopilot 并行跑 N 个不同 $\alpha$ /安全边际的模型，用 cost function 选最优——**单模型调参不如 ensemble 加 guardrail**。

3. PID 控制器（Padala 2007 原型）： $u(t) = K_p e(t) + K_i \int e(\tau)d\tau + K_d \frac{de(t)}{dt}$ ， $e(t) = r - y(t)$ 。 $u(t)$ 为控制输出（CPU shares/副本调整量，周期 1–5 分钟）； $r$ 为目标（如 CPU 60%）； $y(t)$ 为实测。生产要点：经典控制假设 actuation 近乎瞬时；当 $L$ （数分钟）≫ 控制周期，控制器基于过期 $y(t)$ 决策 → 典型抖电源。工业界解法是**把 $L$ 显式建模进控制器**（或切预测式），不是调 $K_p K_i K_d$ 。

### Topic 3：SLO / QoS 感知的 Autoscaling

**核心概念**

- **Tail latency SLO**：SLO 看分位数不看均值。典型：99.9% 请求 < 200ms（ $\varepsilon=0.001$ 允许破损率）。生产最常见的死法：平均延迟好看但 p99.9 爆炸。
- **Colocation interference（混部干扰）**：多任务共享 CPU/LLC/内存带宽/网卡互相踩。关键洞察：干扰主要伤害**尾延迟**而非均值——Heracles / PARTIES / FIRM 都在解这题。
- **SLO violation 的非对称代价**：欠配一次（SLO 破、用户受损、级联熔断）灾难级不可逆；过配一小时只是几块钱。**这是整个 topic 的经济学地基：guardrail 的本质是"花小钱买保险"。**
- **Pessimistic guardrail（悲观防护）**：决策不走预测点估计，走预测分布上分位数，叠加硬下限（如"缩容后容量 ≥ 过去 1 小时峰值 80%"）。宁可过配，不赌预测。

**核心公式**

1. Tail latency SLO 定义：设 $L$ 为单请求端到端延迟（ms）， $S$ 为 SLO 目标（如 200ms）， $\varepsilon$ 为允许破损率（如 0.001）： $P(L > S) \le \varepsilon$ 。小例子：1 秒 10,000 请求， $S=200$ ms， $\varepsilon=0.001$ → 最多 10 个超 200ms，第 11 个即 violation。 $P$ 是固定统计窗口（如 1 分钟）内的经验频率。

2. 欠配/过配非对称代价：决策周期 $T$ （如 5 分钟）， $D$ 为真实需求（核）， $R$ 为供给容量（核）， $c_{over}$ 为过配单位代价（\$/核/ $T$ ）， $c_{under}$ 为欠配单位代价（大 100–1000 倍）： $C(R) = c_{over} \cdot \mathbb{E}[\max(0, R-D)] + c_{under} \cdot \mathbb{E}[\max(0, D-R)]$ 。最优供给满足 $P(D > R^*) = c_{over}/(c_{over}+c_{under})$ （newsvendor 解）。小例子： $c_{under}/c_{over}=100$ → 按需求分布 99 分位数供给——"悲观"的数学来源。

3. Pessimistic guardrail：设 $\hat{F}$ 为 nowcaster 输出的未来 $H$ 分钟需求预测分布， $Q_{1-\delta}(\hat{F})$ 为其 $1-\delta$ 分位数（如 $\delta=0.001$ ）， $R_{floor}$ 为硬下限（如过去 60 分钟观测峰值的 80%）： $R = \max(Q_{1-\delta}(\hat{F}), R_{floor})$ 。若预测分布突然变窄（模型过度自信）， $R_{floor}$ 兜底，防一次灾难性缩容。

4. 利用率→1 时尾延迟发散（M/M/1 直觉）：到达率 $\lambda$ （req/s），服务率 $\mu$ （req/s），利用率 $\rho=\lambda/\mu$ 。 $P(T>t) = e^{-(\mu-\lambda)t}$ ，故 p99 延迟 $t_{99} = 4.605/(\mu-\lambda)$ （秒）。小例子： $\mu=1000$ req/s， $\lambda=900$ （ $\rho=90\%$ ）时 $t_{99}=46$ ms； $\lambda=990$ （ $\rho=99\%$ ）时 $t_{99}=460$ ms——利用率涨 9 个点，尾延迟涨 10 倍。**超卖必须配 guardrail 的排队论根因。**

### Topic 4：Autoscaler 的压测与容量验证

**核心概念**

- **Load fidelity（负载保真度）**：合成负载与真实流量的分布匹配度。记真实到达间隔分布 $F_R$ 、请求类型混合 $G_R$ ，合成器产生 $F_S, G_S$ ；保真度 = 分布距离如 $D_{KL}(F_R \parallel F_S)$ ，越小越好。小例子：真实 500 req/s 中 1% 是大上传（服务 2s，其余 50ms）；合成器若只发均匀小请求，p99 被低估一个数量级——压测白做。
- **Open-loop vs Closed-loop**：open-loop 到达过程与系统完成无关（如泊松 $\lambda$ req/s），系统变慢请求照进——能暴露过载；closed-loop 是 N 个虚拟用户（Little 定律 $N = X(R+Z)$ ， $X$ 吞吐 req/s， $R$ 响应 s， $Z$ 思考 s），系统恶化时负载自动掉——**过载被"自我节流"藏起来**。压测 autoscaler 必须用 open-loop（如 wrk2）。反面就是 coordinated omission（协调遗漏）：生成器自身延迟时"忘记"发本该发的请求。
- **Pre-flight / Shadow 验证**： $\pi_{old}$ 为生产策略， $\pi_{new}$ 为候选， $D$ 为实时生产流。Shadow：复制 $D$ 打到跑 $\pi_{new}$ 的影子集群，响应丢弃，对比窗口 $T$ （如 30 分钟）内 SLO 指标；放行条件 violations( $\pi_{new}$ ) ≤ violations( $\pi_{old}$ ) 且 cost 更低。Canary：切 $f$ （如 5%）真实流量给 $\pi_{new}$ 跑 $T$ ，破 SLO 即回滚。
- **Fault injection（故障注入）**：故障集合 $F$ = {节点宕机、网络分区、磁盘卡顿、注入 $\Delta$ ms 延迟}，强度 $I$ （如每次杀 1/20 节点），度量 RTO（秒）与 SLO 损伤。FATE 的思想是**系统性枚举**组合而非随机 chaos。

---

## Part 2 — 论文清单（22 篇 + 1 参考文档）

> 按 topic 分组；跨 topic 出现的只在主属列出、他处引用。Practitioner relevance 每篇两句：学什么 / 没覆盖什么生产现实。

### T1 需求预测 / Nowcasting（7 篇）

1. **DeepAR: Probabilistic Forecasting with Autoregressive Recurrent Networks** — David Salinas, Valentin Flunkert, Jan Gasthaus (Amazon Research)；arXiv:1704.04110 (2017)，IJF 2020。贡献：在大量相关序列上联合训练一个自回归 RNN，直接输出未来分布参数，开创"一个全局模型预测成千上万条序列"范式。学：fleet 有几十万条需求曲线时"一序列一模型"运维爆炸，全局模型是唯一可扩展路线。没覆盖：只管预测准，不管 $H \geq L_{act}$ 与非对称代价，分布到容量决策之间还差一个报童公式。
2. **Temporal Fusion Transformers for Interpretable Multi-horizon Forecasting** — Bryan Lim (Oxford), Sercan Ö. Arık, Nicolas Loeff, Tomas Pfister (Google Cloud AI)；arXiv:1912.09363 (2019)，IJF 2021。贡献：静态特征/已知未来输入/历史观测三类异构输入统一进 attention，一次输出整段 horizon 的分位数（P10/P50/P90）+ 变量重要性。学：autoscaling 输入天然异构，TFT 是教科书结构。没覆盖：interpretability 不如 guardrail 实在；推理延迟对分钟级 nowcasting 是负担。
3. **Forecasting at Scale (Prophet)** — Sean J. Taylor, Benjamin Letham (Facebook Core Data Science)；The American Statistician 2018。贡献：加性模型（趋势+多重季节性+节假日），不懂时序也能调出靠谱预测，工业界 baseline。学：验证"需求曲线有无可预测结构"的最低成本探针。没覆盖：为天/周粒度设计，分钟级噪声与 flash crowd 非其主场。
4. **Resource Central** — Eli Cortez 等 (Microsoft / MSR)；SOSP 2017。贡献：首次大规模刻画 Azure VM 负载（"历史行为是未来的好预测器"），建成离线训练+在线 serving 的预测中台，直接驱动 VM 调度超分。学：hyperscaler 把"预测"做成平台能力的标准答案（遥测→训练→serving→多资源管理器调用）。没覆盖：预测的是 VM 生命周期这类慢变量，非分钟级 nowcasting；超分代价模型比在线 SLO 约束简单。
5. **Autopilot: Workload Autoscaling at Google** — Krzysztof Rzadca 等 (Google)；EuroSys 2020。见 T2 第 1 条（主属 T2，此处因含预测组件引用）。
6. **Large-scale Cluster Management at Google with Borg** — Abhishek Verma 等 (Google)；EuroSys 2015。贡献：十年集群管理定稿（准入/装箱/超分/抢占）+ 公开 Borg trace，成领域事实标准数据源。学：不是预测论文，而是"预测的目标系统长什么样"——理解 cell、priority band、资源回收，才知道预测误差如何被放大/吸收。没覆盖：2015 年 Borg 无 ML 预测组件；trace 脱敏且旧。
7. **Characterizing Co-located Datacenter Workloads: An Alibaba Case Study** — Yue Cheng, Zheng Chai, Ali Anwar (GMU / IBM)；arXiv:1808.02919 (2018，技术报告)。贡献：首个区分在线长任务/离线批任务的公开混部 trace 分析（1313 台），揭示真实混部集群中超分/超售行为。学：唯一公开可拿、带作业类别标签的混部数据，适合做预测模型离线验证集。没覆盖：仅 24 小时、1313 台，fleet-scale 外推须谨慎；2018 年老 trace。

### T2 扩缩容算法（5 篇 + 1 文档）

1. **Autopilot: Workload Autoscaling at Google** — Krzysztof Rzadca, Paweł Findeisen, Jacek Świderski 等 (Google)；EuroSys 2020。贡献：生产级三闭环（horizontal 副本 + vertical CPU + vertical 内存），指数平滑 ensemble + cost function 自动定 Borg 容器 limit；slack 46%→23%，严重 OOM 作业数降为 1/10。学：hyperscaler "ML + 启发式 + 强 guardrail"最完整公开样本；slack/OOM 可直接抄的评估指标。没覆盖：冷启动/预热的 $L_{act}$ 建模细节淡化；多租户干扰下的 limit 失效案例缺失；单 job sizing，非多服务 SLO 联合决策。
2. **Adaptive Control of Virtualized Resources in Utility Computing Environments** — Pradeep Padala 等 (Michigan / HP Labs)；EuroSys 2007。贡献：控制理论 autoscaling 奠基之一：无 first-principle 模型时 black-box 辨识 + 自适应 MIMO 控制器（LQR 形式），Xen 多租户下按应用层 QoS 动态调资源份额。学："先辨识、再控制"框架是所有 SLO-aware autoscaler 思想源头。没覆盖：小规模 testbed，无 fleet-scale 的分钟级 $L$ 、非平稳流量、真实 guardrail。
3. **Predictive Auto-scaling with OpenStack Monasca** — Giacomo Lanciano 等 (Scuola Normale Superiore)；UCC 2021。贡献："阈值作用在预测值上"的开源组件：RNN/MLP 预测 15 分钟后平均 CPU，连续 3 次超 80% 才 scale-out，提前消化 30 分钟爬坡负载。学：forecast-then-threshold 是最易落地的预测式改造（不动阈值体系，只换输入信号）。没覆盖：预测误差的非对称代价未形式化；无生产级灰度验证。
4. **SLA-Adaptive Threshold Adjustment for a Kubernetes HPA** — Olesia Pozdniakova 等 (VGTU)；Electronics (MDPI) 2024。贡献：EDA + 移动平均动态识别满足 SLO 的目标利用率阈值，SMA 版以轻微过配换 SLO 合规。学：点出"阈值是拍脑袋定的"这一最大痛点，动态阈值可作 HPA 外挂调参器。没覆盖：偏统计描述，无突发流量预测成分；阈值抖动稳定性未讨论。
5. **Auto-scaling Web Applications in Clouds: A Taxonomy and Survey** — Chenhao Qu（合著者/机构 [待核实]，期刊发表状态 [待核实] ）；arXiv:1609.09224 (2016)。贡献：autoscaling taxonomy（时机 reactive/proactive/hybrid × 目标 cost/SLO × 技术 阈值/控制/排队/ML），归类上百篇工作并指薄弱点。学：新人最快全景图，选型对照防重复造轮子。没覆盖：2016 年综述，无 K8s/容器原生、serverless、hyperscaler 生产经验论文覆盖。
6. **(参考文档) Kubernetes 官方文档：Horizontal Pod Autoscaler** — kubernetes.io（持续更新）。贡献：阈值式工业标准实现：比例式副本计算 + tolerance（默认 10%）+ scaleUp/scaleDown stabilization（缩容默认 300 秒取窗口最大值）+ behavior 策略。学：所有自研 autoscaler 的 baseline 对照组；"工业界如何用简单机制防抖"的活教材。没覆盖：纯反应式无预测；不解释参数背后的调参血泪。

### T3 SLO 感知（6 篇）

1. **Heracles: Improving Resource Efficiency at Scale** — David Lo 等 (Stanford + Google)；ISCA 2015。贡献：反馈控制器让 latency-critical 与 best-effort 安全混部，动态划分 CPU/LLC/内存带宽，LC 的 SLO 不被侵犯。学：混部"安全边界"范式源头（先定不可侵犯的 LC 目标，再让渡剩余资源）。没覆盖：单机内资源划分，不涉跨机扩缩容决策与分钟级执行延迟。
2. **PARTIES: QoS-Aware Resource Partitioning for Multiple Interactive Services** — Shuang Chen, Christina Delimitrou, José Martínez (Cornell)；ASPLOS 2019。贡献：打破"一机只能跑一个 LC 服务"，软硬件分区让多个 LC 同机共存且 QoS 不破，吞吐平均 +61%。学：LC 资源 fungible、可在线试错找分区；"高优服务之间也要超卖"场景直接可用。没覆盖：受控集群实验；生产 noisy neighbor 跨机跨租户时变，论文没回答。
3. **Autopilot**（见 T2-1）：此处视角是 SLO 侧——slack/OOM 指标与"预测+启发式"混合路线；SLO 破损代价建模隐式。
4. **Protean: VM Allocation Service at Scale** — Ori Hadary 等 (Azure + MSR)；OSDI 2020。贡献：Azure 生产 VM 分配器，policy/mechanism 分离，规则式 Allocation Agent + 多层缓存做到毫秒级分配，单可用区 10–100k 机器，COVID-19 容量 crunch 实战验证。学：fleet-scale "快而不完美"哲学（牺牲分配质量换并发吞吐），是 guardrail 的工程对应物。没覆盖：它是 placement（放哪台）非 autoscaling（何时扩缩）；SLO 约束是用户侧需求非内生优化目标。
5. **Sinan: ML-Based and QoS-Aware Resource Management for Cloud Microservices** — Yanqi Zhang 等 (Cornell)；ASPLOS 2021。贡献：CNN 短期预测 + Boosted Trees 长期演化，预测某资源配置下端到端延迟与 QoS 破损概率，逐 tier 在线调资源，省 25–59% 且 QoS 不破。学："预测 QoS 破损概率"而非"预测延迟点估计"——guardrail 的数学雏形；DeathStarBench 评估套路可复用。没覆盖：GCE 实验规模远小于生产；没处理预测器执行延迟与误报致扩缩震荡。
6. **FIRM: An Intelligent Fine-Grained Resource Management Framework for SLO-Oriented Microservices** — Haoran Qiu 等 (UIUC)；OSDI 2020。贡献：三级 ML 闭环（定位肇事微服务→定位 contention 资源→动态 reprovision），SLO violations 降 16 倍、CPU limit 降 62%、尾延迟降 11 倍。学：SLO 破损"检测-定位-缓解"完整闭环，telemetry+ML 取代手写启发式路线图。没覆盖：缓解动作只有 reprovision（加资源），无缩容侧悲观保护；误定位代价（加错地方）未讨论。

### T4 压测与验证（6 篇）

1. **Treadmill: Attributing the Source of Tail Latency through Precise Load Testing and Statistical Inference** — Yunqi Zhang, David Meisner (Facebook), Jason Mars, Lingjia Tang (Michigan)；**ISCA 2016**（纠正：不是 USENIX ATC）。贡献：指出压测工具自身缺陷（coordinated omission、统计聚合错误）会得出误导结论；精确负载生成 + 统计推断，不扰动生产归因尾延迟来源。学："压测工具本身会撒谎"的奠基方法论，autoscaler 验证前必读。没覆盖：单服务延迟归因，无扩缩容决策闭环。
2. **DeathStarBench** — Y. Gan 等 (Cornell)；ASPLOS 2019。贡献：5 个端到端微服务应用（社交/媒体/酒店/电商/银行），成学术界验证 SLO 与弹性的事实标准基准。学：开源可跑，验证 autoscaler 最坏情况行为最便宜的沙箱。没覆盖：合成拓扑，缺真实多租户干扰与非平稳突发。
3. **Sieve: Automatic Reliability Testing for Cluster Management Controllers** — Xudong Sun 等（机构 [待核实] ，应为 UIUC）；**OSDI '22**（纠正：不是 OSDI 2021）。贡献：系统性扰动控制器可见的集群状态（differential oracle），全自动挖出 10 个主流 K8s 控制器 46 个新 bug（35 确认/22 修复），无需专家写 spec。学："控制器在异步/并发下会怎么错"的测试方法论，可套用到自研 autoscaler 回归测试。没覆盖：只测控制逻辑正确性，不测生产负载下 SLO/成本 trade-off。
4. **FATE and DESTINI** — Haryadi S. Gunawi 等 (Berkeley / Facebook)；NSDI '11。贡献：FATE 用 failure ID 系统性枚举多重故障组合（4 万种），DESTINI 用 Datalog 写恢复正确性 spec，在 HDFS/ZooKeeper/Cassandra 挖 16+ 新 bug。学："autoscaler 在故障下会不会雪崩"需要系统性故障枚举思维，而非随机 chaos。没覆盖：针对存储恢复，不涉扩缩容策略与 SLO 交互。
5. **Open Versus Closed: A Cautionary Tale** — Bianca Schroeder, Adam Wierman, Mor Harchol-Balter (CMU)；NSDI '06。贡献：证明 open/closed-loop 负载生成器行为天差地别，8 条指导原则 + partly-open 模型。学："压测 autoscaler 必须用 open-loop"的理论源头；选错模型系统性低估过载风险。没覆盖：纯方法论，无 autoscaling 实例。
6. **TailBench** — Harshad Kasture, Daniel Sanchez (MIT)；IISWC 2016。贡献：8 个延迟敏感应用 + warmup/measurement harness，专为尾延迟研究设计。学：验证 SLO guardrail 时比 DeathStarBench 更轻量，单服务延迟刻画更细。没覆盖：无多服务调用链，不适合测级联故障下的扩缩容。

> 注：shadow/canary 验证是工业界标准实践（如 AWS SageMaker Shadow Tests），但未搜到够分量的正式论文，故只在概念节讲方法，不列入清单——这是搜索后的诚实结论。

---

## Part 3 — 对比脚手架（文献共识 vs 我的经验）

> 用法：每 topic 先读"文献共识"，再回答"待填"——答不上来的就是 gap（要么文献没写、要么自己没形式化）。脱敏写，不出现雇主标识。

### T1 需求预测 / Nowcasting

**文献共识**
- 容量决策的正统桥梁是"概率预测 + 报童公式"：按 $C_{under}/(C_{under}+C_{over})$ 分位数备容量，而非点预测加拍脑袋边际。
- Fleet 规模下"一序列一模型"不可运维，全局模型（DeepAR/TFT 范式）是唯一可扩展路线。
- 时效硬约束 $H \geq L_{act}$ ；评估预测器要看高分位覆盖率，不看 RMSE。

**待填：我的经验**
1. 你的 nowcaster 输出的是点预测还是分位数？若是点预测，guardrail 的安全边际是多少、怎么定的（拍脑袋 / 历史回测 / 在线调）？
2. 你的 $H$ 实测多少？ $L_{act}$ 的 p50/p99 多少？流量突变时 $L_{act}$ 本身时变， $H \geq L_{act}$ 还成立吗？
3. 非对称代价在你的系统是显式参数（报童公式）还是隐式文化？欠配一次的真实代价你能量化吗？
4. Prophet / DeepAR / TFT 这类试过吗？最后选型弃用它们的原因是什么（推理延迟？可解释性？运维成本）？
5. Flash crowd 下预测器表现如何？有没有"预测器 + 应用重试风暴"正反馈的案例？怎么断开的？

### T2 扩缩容算法

**文献共识**
- 工业 baseline = 反应式阈值（HPA：比例式 + tolerance 10% + stabilization window）；预测式改造最易落地的是 forecast-then-threshold。
- 控制理论方法在 $L \gg$ 控制周期时因过期信号抖振；工业界解法是把 $L$ 显式建模进控制器或切预测式，而非调参。
- Autopilot 验证了"ML ensemble + 启发式 + 强 guardrail"三段式；单模型调参不如 ensemble 加防护。

**待填：我的经验**
1. 你的系统是 reactive / predictive / hybrid？hybrid 的切换逻辑是什么？
2. 抖动（oscillation）出现过吗？根因是预测噪声、反馈延迟还是阈值过紧？用什么收敛的（cooldown / hysteresis / 死区）？
3. Vertical scaling 用了吗？调 limit 的安全边际是 ensemble 选还是固定比例？
4. 扩容侧与缩容侧策略对称吗？缩容是不是更悲观？悲观了多少（量化）？
5. HPA 的 tolerance / stabilization 默认参数在你的场景够用吗？哪些是自己改过、为什么？

### T3 SLO 感知

**文献共识**
- SLO 只看分位数；混部干扰主要伤害尾延迟；M/M/1 直觉： $\rho \to 1$ 时尾延迟发散，超卖必须配 guardrail。
- 非对称代价 → 悲观供给：预测分布上分位数 + 硬下限（ $R = \max(Q_{1-\delta}, R_{floor})$ ）。
- "预测 QoS 破损概率"（Sinan）比"预测延迟点估计"更接近 guardrail 的数学形态。

**待填：我的经验**
1. 你的 SLO 指标是 p99 还是 p99.9？统计窗口多长？破损的定义与告警阈值？
2. Guardrail 几层？分位数供给的 $\delta$ 取多少？ $R_{floor}$ 怎么定？预测分布突然变窄（模型过度自信）时有兜底吗？
3. 混部场景下，预测误差导致 SLO 破损的案例？当时如何定位到是容量问题而非应用问题？
4. 欠配的真实代价（违约/商誉/级联风险）在容量决策里是显式参数还是隐式文化？
5. 你的系统更接近"预测破损概率"还是"预测延迟点估计"？为什么选这条？

### T4 压测与验证

**文献共识**
- 压测 autoscaler 必须用 open-loop；closed-loop 会自我节流藏住过载（NSDI '06）。
- 压测工具自身会撒谎：coordinated omission 与统计聚合错误（Treadmill, ISCA '16）。
- 新策略上线前：shadow（复制流量，响应丢弃）或 canary（小比例真实流量），放行条件 violations 与 cost 双达标。
- 故障要系统性枚举（FATE 思想），而非随机 chaos。

**待填：我的经验**
1. 你的压测框架 open-loop 还是 closed-loop？负载保真度怎么度量（与真实流量分布的距离）？
2. 新扩缩容策略上线前走什么验证（shadow / canary / 比例 / 时长 / 放行条件）？有没有拦截过灾难性策略的真实案例？
3. 压测注过故障吗（节点宕机/网络分区/延迟注入）？autoscaler 在故障下的行为符合预期吗？最意外的一次是什么？
4. DeathStarBench / TailBench 用过吗？还是完全生产流量回放？各自盲区是什么？
5. 压测环境与生产的 gap 最大在哪（多租户干扰？突发流量？冷启动分布）？这个 gap 导致过线上事故吗？
