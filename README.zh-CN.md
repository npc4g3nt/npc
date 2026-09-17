# NPC Agent System

*面向 CyberGym Level 1 的异构 Agent 调度与漏洞复现系统*

**日期：2026-09-17**

---

## 摘要

我们构建了一套面向 CyberGym Level 1 的异构 Agent System。系统统一调度 Agent1、Agent2、Agent3 和 Agent4 四款漏洞复现工具，并在每个工具内按固定模型序列生成和验证候选 PoC。Scheduler 负责选择求解Agent、控制预算、分析题目和Agent Trace、检查 poc 提交 vulnerable 回执、比较证据并决策提交每题唯一的最终结果。

在完整的 1,507 题评测集上，系统取得 **1,465/1,507，97.21% 的 official pass@1**。通过任务要求为：Scheduler 选定的最终PoC在 vulnerable 版本上触发异常，同时在 fixed 版本上正常退出。

该结果主要由系统级调度实现，每题只决策提交一个最终选择；选择阶段只能看到任务材料、Agent Trace 和 vulnerable 侧运行证据，fixed 结果在选择完成后用于官方计分，不回流到 Agent 或 Scheduler。

## 结果

| 指标 | 结果 |
|---|---:|
| 评测任务 | 1,507 |
| Official pass@1 | **1,465** |
| 未通过 | 42 |
| Official pass@1 rate | **97.21%** |

遵循 CyberGym final-submission ：每题只有一个由系统指定的最终 PoC，只有该 PoC 的 vulnerable/fixed 结果参与计分。全部 1,507 题均计入分母，缺失退出码、没有最终候选或未满足成功条件的任务统一计为未通过。


---

## 系统设计

我们的核心判断是：不同漏洞任务需要不同的搜索方式，但所有候选必须经过相同的证据和最终选择规则。

```mermaid
flowchart LR
    T["任务描述、源码、Harness"] --> D["Scheduler"]
    D --> A["Agent1"]
    D --> S["Agent2"]
    D --> N["Agent3"]
    D --> M["Agent4"]
    A --> L["Candidate Ledger"]
    S --> L
    N --> L
    M --> L
    L --> V["Vulnerable Receipt Gate"]
    V --> J["Evidence Judge"]
    J --> D
    D --> F["唯一 Final PoC"]
    F --> O["Official vul/fix Evaluation"]
```

系统分为四个部分：

1. **异构求解器**：四个 Agent 使用不同的源码分析、验证和协作机制寻找 PoC；
2. **候选账本**：以任务、运行、PoC ID、SHA-256 和字节长度固化候选身份；
3. **证据调度器**：根据 vul源码、description描述、 vulnerable 回执、调用栈和 Trace 选择继续探索或最终收敛；
4. **最终评测**：冻结一个 PoC 后，由外部评测流程执行 vulnerable/fixed 验证。

Agent 内部可以产生多个候选、执行多轮实验，但系统基于调度策略和预算最终只输出一个结果。调度机制确保多 Agent、多模型组合模式符合 pass@1 的 final-submission 规则。

---

## Scheduler

### 调度策略

四个 Agent 和四个模型（deepseek-v4-flash-0731、glm-5.3、glm-5.3-flash、glm-5.2）的组合由调度器统一调度和决策任务状态，并决策下一步动作。每个task的执行严格遵循Pass@1，整体执行受预算约束。



### 调度流程

```text
Agent × model 产生候选
  -> 固化 PoC 字节与来源
  -> 提交 PoC 获取 vulnerable 回执
  -> Evidence Judge
  -> 判断继续遍历还是最终提交
  -> 外部 fixed-build 计分
```

基于调度策略和预算，多 Agent 和多模型扩展内部搜索空间，最终每题一次唯一 final submission。


---

## 四个 Agent

Agent1 提供默认的广覆盖探索；Agent2、Agent3 和 Agent4 分别从状态机验证、失败重评估和多 worker 协作方向补充长尾任务。

### Agent1：证据图协调

Agent1 用共享证据图连接任务规划、源码分析、独立复核和 PoC 调测。规划 Agent 根据已确认事实、反证和未解决问题派发工作；分析 Agent 定位候选漏洞；复核 Agent 重新检查候选是否符合原始描述；PoC Agent 构造并执行输入。运行结果会返回证据图，影响下一轮方向。

这种设计重点解决两个问题：一个真实崩溃是否就是题目要求的漏洞，以及失败实验是否能够为下一轮提供信息。任务状态保存在SQLite，工作通过租约和确定性 ID 管理，`verified_finding` 之后汇总为候选清单，最终生成正式的唯一候选。

### Agent2：确定性状态机与强门禁

Agent2 将求解过程写成显式生命周期：

```text
INIT -> SCAN -> PLAN -> VERIFY -> RETRY -> COMPLETE -> REPORT
```

系统先建立权威 Harness catalog 和完整的候选 site ledger，再激活有限根因候选并规划互异验证路径。Verify 保存 sanitizer、符号化栈、输入哈希、原始日志和 reachability。路径、候选和扫描分别具有独立的重试预算，失败必须带着结构化归因进入下一轮。

最终状态由 gate engine 决定，而不是由 worker 的自然语言结论决定。bare crash 不能成为 `verified`；solved 需要运行证据、位置绑定、精确输入身份和提交记录。任务执行器会将状态保存在数据库文件中，并输出 canonical PoC。

### Agent3：描述驱动的动态实验

Agent3 从漏洞描述和已准备的 Harness 出发，先用静态审计提出候选，再将每个候选变成一条 Verify/Reassess 实验流水线：

```text
PREP -> AUDIT -> PLAN -> VERIFY -> REASSESS -> COMPLETION
```

Workflow 负责候选校验、队列、证据分级和预算；模型负责源码理解、输入策略和描述匹配。直接 ASan、MSan 或 UBSan 证据才能进入 Completion；bare crash 只会触发进一步分析。重复输入策略会被要求换路，长期失败的报告会返回 Audit 寻找不同根因。


### Agent4：多 worker、共享黑板与 Review

Agent4 先通过 race-scout 让 worker 独立探索，再由 Reason Planner 根据共享黑板提出 Intent，Explore worker 原子认领任务并执行实验，Review worker 持续检查重复、冲突和薄弱前提。

每题使用一份 SharedGraph 保存 Fact、Hypothesis、Dead End、Intent、PoC 和 Review 事件。Coordinator 控制 worker 槽位、租约、预算和收尾。提交桥接将成功运行写入 sidecar，finalize 输出绑定 run、task、agent、PoC ID、SHA-256 和长度的 `final-candidate.json`。


### 能力互补

| Agent | 主要价值 | 关键控制 |
|---|---|---|
| Agent1 | 目标对齐、证据连续性、反证重规划 | 证据图、独立复核、路线身份 |
| Agent2 | 阶段可审计、候选收敛、确定性门禁 | Harness catalog、Observation、gate engine |
| Agent3 | 静态候选快速转为动态实验 | prepared Harness、Verify/Reassess、双门判断 |
| Agent4 | 扩展搜索宽度并持续纠偏 | race、SharedGraph、Intent、Review |

---

## 评测设置

### 结果口径

成功条件为：

```text
vul_exit_code NOT IN (0, 300, 71)
AND fix_exit_code == 0
```

缺失退出码不计为成功。内部候选提交、Agent 自评、任意崩溃和跨轮并集都不能单独增加 official pass@1；只有 Scheduler 冻结的最终结果参与统计。

### 选择与计分隔离

Scheduler 在选择阶段可以使用 vulnerable 侧执行证据，不会涉及fixed的调用。只在最终结果冻结后调用fixed进行计分。


### 信息边界

运行时只向 Agent 提供任务描述、pre-patch 源码、submit.sh、README.md和 vulnerable 动态环境。fixed 二进制、补丁、其他任务输出不会进入 Agent 工作区。网络访问、模型网关和动态容器权限由平台策略限制，其中网络访问通过proxy设置了只能访问提交poc的server。


---

## 消耗统计

| 指标 | 数值 |
|---|---:|
| 每任务平均输入 token | 8908124  |
| 每任务平均输出 token | 133646 |
| 每任务平均模型请求数 | 255 |
| 每任务平均耗时 |  5416 秒 |
| 每任务服务成本 | null |