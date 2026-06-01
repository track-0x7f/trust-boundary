# Case Study: curl 安全披露信任链测绘

**状态**: 第一版完成
**日期**: 2026-06-02
**观测者**: Track-0x7F

## 1. 为什么是 curl

curl 拥有目前开源生态中最透明的安全披露流程之一。Daniel Stenberg 是项目维护者、CVE 分配者（curl 是 CNA）、安全公告发布者、安全团队入选决定者——事实上是信任链上的几乎每一个关键节点。

这种"单一决策者"模式不是技术上最优的，但它是 curl 项目清醒选择的设计。它的信任架构值得测绘不是因为它是坏的——是因为它的边界条件已经被真实事件测试过多次，有足够的数据点供分析。

## 2. curl 安全披露流程图

```
发现者 → [HackerOne] → 安全团队(私密列表) → Daniel(最终决策)
                                 ↕
                            distros@openwall
                                 ↕
                            CVE分配 → 修复 → 公告
```

### 各节点分析

### 2.1 入口：HackerOne 独占

> "The curl project cannot handle vulnerability reports sent to us over email. We lose track of the reports."

**信任关系**: 发现者信任 HackerOne 平台会安全地将报告传递给 curl 安全团队。同时信任 HackerOne 不会泄露报告内容。

**结构性脆弱点**:
- HackerOne 是唯一的报告入口。如果 HackerOne 变更条款、下线、或一个发现者无法/不愿使用 HackerOne（地缘政治封锁、账号限制、隐私担忧），他们的报告不会被接收。
- 这创造了**双重门禁**: 发现者不仅要通过技术评估，还要先通过 HackerOne 的账号验证。
- 政策明确拒绝对邮件报告进行处理——这不是技术限制，是设计选择。结果是部分报告从源头被放弃。

### 2.2 安全团队：私密列表

> "We do not make the list of participants public mostly because it tends to vary somewhat over time."

**信任关系**: 报告者信任一个未知成员组成的团队会公正评估其报告。发现者不参与安全团队的组成决策。

**结构性脆弱点**:
- 团队组成不透明。发现者无法知道谁在评估其报告。
- 入选标准是主观的："long-term presence" 和 "shown an understanding"——都是隐式信任判断，无客观标准。
- 没有公开的投诉或上诉路径——如果发现者认为评估不公。

### 2.3 严重性评级：无 CVSS，自定义 4 级

> "We do not support CVSS as a method to grade security vulnerabilities. We believe CVSS is a broken system."

**信任关系**: 社区信任 curl 安全团队基于"所有因素"（attack vector, complexity, privileges, config, protocols, platform, impact）自定严重性。

**结构性脆弱点**:
- 拒绝 CVSS 的同时没有提供等价的标准化替代方案。下游（发行版、企业安全团队）需要自行判断严重性。
- 严重性决定基于"我们觉得"——在边界情况下，发现者和团队可能严重分歧，但无正式仲裁机制。
- **关键数据点**: curl 历史上只标记过 **1 次 Critical**（CVE-2000-0973，25 年前）。即使 CVE-2023-38545（SOCKS5 堆溢出，被外部称为"probably the worst curl security flaw in a long time"）也只评为 **High** 而非 Critical。这暗示严重性标尺压缩了——Critical 实际上不可达。
- 政策文档承认边界案例存在："There can be a discussion about what the documentation actually means, which might end up with us still agreeing that it is a security problem."——这是对边界模糊性的诚实承认，但也意味着发现者在边缘案例中处于弱势。

### 2.4 经济激励："No Bug Bounty" vs 实际支付

**这是最重要的信任边界发现**。

> 政策文档: "There is no bug bounty and the curl project never offers rewards for reported vulnerabilities."

> 实际数据（vuln.csv）: 发现者持续收到 $480–$4,660 的赏金。

| CVE | 赏金 | 说明 |
|-----|------|------|
| CVE-2023-38545 | $4,660 | SOCKS5 堆溢出（High） |
| CVE-2025-14017 | $2,540 | 多线程 LDAPS TLS 选项 |
| CVE-2024-6197 | $2,540 | ASN.1 栈缓冲区释放 |
| CVE-2023-38546 | $540 | cookie 注入（Low） |
| CVE-2025-9086 | $505 | cookie 路径越界读（Low） |

**信任边界分析**:
- 技术上 curl 项目确实不支付赏金——赏金来自 HackerOne 或第三方赞助商。
- 但**实际体验**是：发现者提交报告，得到确认修复，拿到赏金。这个体验与"no bug bounty"的政策声明不一致。
- **影响**: 新发现者阅读政策后可能认为"没有赏金那算了"，从而不提交。实际上提交后可能有赏金，但发现者在决策时刻信息不对称。
- 这是**善意设计缺口**的一个典型案例: 政策声明是为了诚实（"我们不付钱"），但实际运营中另一个实体付了钱，导致诚实声明变成了误导。

### 2.5 distros@openwall 协调

> "No more than seven days before release, inform distros@openwall to prepare them."

**信任关系**: curl 信任 distros 列表的成员（各发行版安全团队）会负责任地处理预披露信息。

**结构性脆弱点**:
- 7 天窗口期是 curl 设定的——但某些复杂修复可能需要更长时间，而与 distros 的协调时间被压缩。
- distros 列表严格限制"仅 Linux 发行版"——Windows 和 macOS 的发行渠道被排除在外。

### 2.6 信用归属

> "make sure to credit all contributors properly"

**信任关系**: 发现者信任 curl 团队会公开、正确地给予信用。

**分析**: 这是系统中最可靠的信任关系之一。curl 的安全公告始终包含发现者的名字或化名，并且 HackerOne 报告 URL 公开可查。但这条信任关系依赖于 Daniel 的个人诚信记录，而不是系统的可验证性。

## 3. 信任链汇总

| # | 信任关系 | 担保形式 | 可验证性 | 边界条件 |
|---|---------|---------|---------|---------|
| T1 | 发现者→HackerOne | 平台协议 | 强（可迁移） | HackerOne 变更/下线 |
| T2 | 发现者→安全团队 | 声誉 | 弱 | 团队组成不透明 |
| T3 | 发现者→Daniel (严重性) | 历史记录 | 中（可对比历史案例） | 边界案例无仲裁 |
| T4 | 下游→curl (严重性) | 公告 | 弱（无标准化评分） | 偏离 CVSS 无替代标准 |
| T5 | 发现者→系统 (赏金) | 政策声明 | 弱（政策与实际不一致） | "No bounty" 声明误导 |
| T6 | curl→distros (预披露) | 信任协议 | 中（window 固定 7 天） | 仅 Linux，排除其他平台 |
| T7 | 发现者→系统 (信用) | 历史记录 | 强（公告可验证） | 依赖个人诚信 |

## 4. 结构性脆弱点总结

### P1: "No Bug Bounty" 善意声明缺口
- **善意假设**: 政策声明"无赏金"是诚实的，发现者会理解这是指项目本身不支付，而非无经济激励。
- **意外场景**: 新发现者读到政策后放弃提交；实际提交者通过 HackerOne 获得 $480–$4,660。
- **影响**: 信息不对称——有经验的发现者知道实际有赏金，新发现者被政策声明劝退。

### P2: 单点决策者的信任透支
- **善意假设**: Daniel Stenberg 会一直健康、积极、公正地运作安全流程。
- **意外场景**: Daniel 不可用时（健康、休假、burnout），安全流程无文档化的继任计划。
- **影响**: 信任链在单点断裂时无备用路径。与人人信上马克被透支的模式一致——单点担保者友好时系统运转良好，单点担保者出问题时无缓冲。

### P3: 入口独占 + 拒绝备选通道
- **善意假设**: 所有发现者都可以且愿意使用 HackerOne。
- **意外场景**: 地缘政治封锁、隐私顾虑、账号限制。
- **影响**: 报告从未到达安全团队。这是一个比"报告被拒绝"更糟糕的场景——报告者被沉默地拒绝在门外。

### P4: 严重性标尺压缩
- **善意假设**: 自定义 4 级系统能准确传达风险。
- **意外场景**: 外部对严重性的预期（基于 CVSS）与 curl 内部评估严重不一致。
- **影响**: 发行版和企业自行重新评分，curl 的评级失去参考价值；发现者对"Low"评级不满时无上诉渠道。

## 5. 修复提案（草案）

### FP1: 澄清赏金政策
- 在政策页面增加一句话: "curl 项目本身不支付赏金，但通过 HackerOne 提交的报告可能有资格获得由 HackerOne 或第三方赞助商提供的赏金（参见历史数据）"。
- 这不是改变政策——是消除善意声明缺口。

### FP2: 安全团队透明化
- 公开安全团队成员名单（或至少数量 + 角色）。
- 建立报告者争议时的升级路径——不是非公开的"讨论"，是文档化的仲裁流程。

### FP3: 冗余入口
- 增加一个非 HackerOne 的报告方法（加密邮件 + GPG 密钥），即使政策偏好 HackerOne。

## 6. 测绘方法说明

本测绘仅使用公开数据：
- curl 官网披露政策文档
- curl 漏洞 CSV 数据（vuln.csv）
- curl 安全公告页面
- 所有分析均基于 curl 项目自我发布的信息，未涉及非公开交流或内部流程。

---

*这是 TrustBoundary 项目的第一份完整案例报告。测绘模板详见 `/template/trust-chain-map.md`。*
