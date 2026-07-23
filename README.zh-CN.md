# Negative-First Mesh Computing

[한국어](README.ko.md) | [English](README.en.md) | [日本語](README.ja.md) | **简体中文**

## “否定优先网状推理”概念概述与系统设计

`Negative-First Mesh Computing` 是一种计算与推理架构。它在接收到问题后，不立即生成最可能的答案，而是将可能的候选项以网状结构同时激活，并通过**优先关闭明显不符合条件的候选项**来得出答案。

其核心原则如下：

> 不逐个候选项寻找正确答案。  
> 先同时激活全部候选项，优先关闭错误项，只使用剩余项作答。

传统搜索系统和生成式 AI 通常会顺序探索候选项，或者持续生成当前上下文中概率最高的输出。Negative-First Mesh 则把问题拆解为多个条件信号，将这些信号传播到整个候选网中，然后优先排除不匹配、相互矛盾、超出范围或缺乏证据的候选项。

最终回答只能使用存活的候选项。如果没有任何候选项存活，系统不得编造内容，而应将结果归类为以下状态之一：

- 错误
- 在指定范围内未找到
- 因证据不足而无法判断
- 问题自身的条件相互矛盾
- 需要继续探索下一个候选组

---

# 1. 问题定义

当前 AI 与传统计算机在大规模数据搜索和推理任务中存在多种限制。

## 1.1 顺序探索成本

随着数据量增加，系统通常需要逐个比较候选项，或反复执行检索、评分、排序和再次验证。

```text
检查候选1 → 排除
检查候选2 → 排除
检查候选3 → 保留
...
检查候选N
```

GPU 和向量搜索能够并行化大量比较，但从逻辑流程上看，系统仍需要加载候选项、计算分数、排序并重新评估结果。

## 1.2 生成优先 AI 的幻觉

即使缺乏充分的直接证据，生成式 AI 仍可能生成语言上自然的后续内容。

```text
问题
  ↓
生成高概率语句
  ↓
以概率方式填补证据空白
  ↓
生成看似合理但错误的答案
```

## 1.3 混淆“未找到”与“不知道”

以下三种表述并不等价：

- 实际上不存在。
- 在指定数据范围内未找到。
- 根据当前可用资料无法判断。

许多系统不能严格保持这种区分，并错误地把一次搜索失败扩展为对现实世界的不存在断言。

## 1.4 数据与计算分离

大多数计算机需要先把数据从内存移动到处理单元，再进行比较或推理。随着 AI 系统规模增长，内存搬运和数据探索会与计算量一起成为瓶颈。

Negative-First Mesh 的长期目标是建立一种结构，使**条件比较与候选抑制能够在数据存储位置或其附近发生**。

---

# 2. 核心概念

## 2.1 候选网

信息不只以文件或地址的形式保存，而是表示为具有多重语义和关系的节点与连接。

例如，`苹果` 可以具有以下关系：

```text
苹果
├─ 分类: 水果
├─ 颜色: 红色、绿色
├─ 形状: 圆形
├─ 功能: 可食用
├─ 关系: 生长在树上
└─ 属性: 甜、酸
```

当问题进入系统时，系统不是只搜索完整句子，而是把构成问题的各项条件传播到对应的网中。

```text
问题: “可食用的红色水果”

可食用 ─┐
红色 ───┼─ 苹果节点被激活
水果 ───┘
```

## 2.2 否定优先抑制

每个候选项最初都处于暂时激活状态。一旦候选项明确违反必需条件，就立即将其关闭。

```text
候选A: 不是水果       → OFF
候选B: 不可食用       → OFF
候选C: 不是红色       → OFF
候选D: 满足全部条件   → ON
```

只要一个必需条件失败，系统就不再把该候选项作为有效答案继续计算。

## 2.3 零检测

没有任何活跃候选项的状态被视为一种独立的计算结果。

```text
活跃候选数 > 0
→ 验证存活候选项

活跃候选数 = 0
→ 禁止生成答案
→ 分类为不存在、错误、不确定或搜索范围不足
```

在该架构中，`0` 不只是失败。它是用于切换到下一搜索组或返回准确不确定性状态的控制信号。

## 2.4 候选组迁移

如果精确表达组返回零个候选项，系统按照受控顺序扩展搜索范围。

```text
精确表达
  ↓ 0
拼写与记法变体
  ↓ 0
同义词与缩写
  ↓ 0
上位概念与下位概念
  ↓ 0
关系、原因与结果
  ↓ 0
无法判断或在范围内未找到
```

随着搜索扩展，候选项与原问题之间的语义距离增大，因此每个候选项都需要增加距离惩罚。

## 2.5 证据约束回答

最终回答只能由具有足够证据的候选项组成。

```text
ACTIVE_SUPPORTED
→ 可直接用于回答

ACTIVE_TENTATIVE
→ 只能作为可能性或推测表达

OFF_UNSUPPORTED
→ 从回答中排除

UNKNOWN_UNRESOLVED
→ 表示为无法判断
```

---

# 3. 与现有技术的关系

Negative-First Mesh 不是单一孤立技术，而是将多个现有概念整合成统一架构。

## 3.1 内容寻址存储器

传统内存通过指定地址读取数据，内容寻址存储器则把搜索内容与大量存储项同时比较。

Negative-First Mesh 将这种比较从精确键扩展到语义、关系、条件、时间和来源可信度。

## 3.2 联想记忆

系统可以根据部分线索恢复完整模式。

```text
部分问题
→ 相关节点同时激活
→ 候选项竞争并相互抑制
→ 收敛到稳定模式
```

## 3.3 神经形态计算

受神经元和突触启发的分布式处理单元响应事件，只激活必要区域。

## 3.4 存内计算

在内存内部或附近执行比较与累加，以减少数据搬运成本。

## 3.5 波与干涉计算

使用电信号、光信号、相位、频率或到达时间差，对多个候选状态进行并行增强或抵消。

## 3.6 图推理

将实体、属性、原因、结果、来源和时间关系表示为节点与边。

Negative-First Mesh 的差异在于，它按照以下流程整合这些技术：

```text
激活整个候选域
→ 优先传播否定条件
→ 抑制无效候选项
→ 检测零状态
→ 迁移到下一候选组
→ 仅把有证据的存活候选项交给生成式AI
```

---

# 4. 目标

## 4.1 第一阶段目标

在软件中实现以下能力：

- 自动拆解问题条件
- 生成候选组
- 同时搜索多个数据源
- 优先排除不匹配候选项
- 区分不存在、错误与无法判断
- 基于证据生成答案
- 抑制幻觉
- 追踪证据与候选排除日志

## 4.2 第二阶段目标

结合向量数据库、图数据库、搜索引擎和规则引擎，实现大规模并行联想搜索。

## 4.3 长期目标

将候选激活与抑制转移到忆阻器、FPGA、光学电路、神经形态芯片或多层波网络等物理计算介质中。

---

# 5. 总体架构

```text
┌──────────────────────────────┐
│            用户问题           │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 1. Question Normalizer       │
│ - 拆解目标、主张与条件         │
│ - 判定范围、时间、风险与输出型  │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 2. Mesh Planner              │
│ - 生成候选组                  │
│ - 生成搜索轴与关系轴           │
│ - 制定并行探索计划             │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 3. Candidate Activator       │
│ - 激活精确候选项              │
│ - 激活相似候选项              │
│ - 激活关系候选项              │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 4. Negative Gate Engine      │
│ - 定义不匹配                  │
│ - 必需条件失败                │
│ - 时间不匹配                  │
│ - 矛盾                       │
│ - 证据不足                   │
│ - 未达到安全阈值              │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 5. Zero Detector             │
│ - 统计活跃候选数              │
│ - 分类零状态                  │
│ - 决定下一候选组              │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 6. Evidence Resolver         │
│ - 来源可信度                  │
│ - 时效性                     │
│ - 直接性                     │
│ - 候选项之间的矛盾            │
│ - 语义距离                   │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 7. Answer Composer           │
│ - 只使用存活候选项            │
│ - 区分事实、推测与未知         │
│ - 显示范围与限制              │
└──────────────────────────────┘
```

---

# 6. 主要组件

## 6.1 Question Normalizer

Question Normalizer 将原始问题转换为结构化框架。

```json
{
  "target": "要搜索或判定的对象",
  "claim": "要验证的主张",
  "required_conditions": [],
  "excluded_conditions": [],
  "scope": {
    "sources": [],
    "time_range": null,
    "region": null,
    "closed_world": false
  },
  "freshness_required": false,
  "risk_level": "LOW | MEDIUM | HIGH",
  "answer_type": "FACT | SEARCH | DIAGNOSIS | DESIGN | RECOMMENDATION"
}
```

### 职责

- 分离含糊的对象
- 提取否定条件与必需条件
- 判断搜索范围是否为闭合范围
- 判断是否需要最新验证
- 设置错误回答的风险等级

---

## 6.2 Mesh Planner

Mesh Planner 定义候选项的探索轴与顺序。

### 默认候选组

1. 精确匹配
2. 拼写或记法变体
3. 同义词或缩写
4. 语义相似
5. 上位概念
6. 下位概念
7. 邻近时间范围
8. 邻近地区
9. 原因与结果
10. 相反概念
11. 跨来源交叉验证
12. 对用户前提的替代解释

### 候选组定义示例

```json
{
  "group_id": "G03",
  "group_type": "SYNONYM",
  "distance": 0.15,
  "activation_policy": "ON_PREVIOUS_ZERO",
  "queries": [
    "离线使用",
    "无需互联网连接即可运行",
    "首次激活后本地使用"
  ]
}
```

---

## 6.3 Candidate Activator

搜索结果、规则结果、图路径和模型建议会被转换为候选节点。

```json
{
  "candidate_id": "C1024",
  "label": "候选项说明",
  "group_id": "G03",
  "state": "ACTIVE_TENTATIVE",
  "matched_conditions": [],
  "failed_conditions": [],
  "evidence_ids": [],
  "distance": 0.15
}
```

即使在无法物理上同时处理所有候选项的环境中，同一候选组也应按照相同的门控顺序批量处理。

---

## 6.4 Negative Gate Engine

Negative Gate Engine 是该架构的核心组件。

### 定义门

检查候选项与问题目标的定义是否一致。

```text
问题: 无定期订阅的一次性购买应用
候选项: 需要每月付费的订阅服务

定义不匹配 → OFF_FALSE
```

### 条件门

只要违反一个必需条件，就关闭该候选项。

```text
必需条件:
- 价格低于1,000美元
- 至少16GB内存
- 无定期订阅

候选项仅支持月度订阅 → OFF_FALSE
```

### 时间门

检查证据时间是否与问题要求的时间一致。

```text
2024年的发布价格
问题询问当前售价
→ OFF_OUT_OF_SCOPE
```

### 存在门

在验证范围为闭合范围时，判断对象是否存在。

```text
完整搜索上传的3份说明书
未发现离线使用说明
→ NOT_FOUND_IN_SCOPE
```

在开放世界中，搜索失败不能视为不存在的证明。

```text
网页搜索未发现
→ UNKNOWN_UNRESOLVED
```

### 矛盾门

当存在更强证据或内部矛盾时，排除候选项。

### 证据门

如果缺乏足以写入回答的直接证据，则将候选项设为 `OFF_UNSUPPORTED`。

### 因果门

防止把相关性直接断言为已经验证的因果关系。

### 安全门

在医疗、法律、金融、安全等高风险领域，提高证据阈值。

---

# 7. 候选状态模型

| 状态 | 说明 |
|---|---|
| `ACTIVE_SUPPORTED` | 有强证据支持，保留为回答候选项 |
| `ACTIVE_TENTATIVE` | 有一定可能性，但仍需进一步验证 |
| `OFF_FALSE` | 明确不符合问题条件 |
| `OFF_CONTRADICTED` | 被更强证据或矛盾排除 |
| `OFF_OUT_OF_SCOPE` | 超出当前时间、地区或数据范围 |
| `OFF_UNSUPPORTED` | 缺乏足够证据支撑主张 |
| `UNKNOWN_UNRESOLVED` | 现有资料不足以判断真伪 |
| `NOT_FOUND_IN_SCOPE` | 在定义明确的闭合范围内未找到 |

关键规则：

```text
没有证据 ≠ 错误
未找到 ≠ 不存在
可能性低 ≠ 不可能
```

---

# 8. 零检测与候选组迁移

## 8.1 零检测条件

```text
ACTIVE_SUPPORTED 数量 = 0
AND
ACTIVE_TENTATIVE 数量 = 0
```

## 8.2 零状态分类

```text
问题条件彼此矛盾
→ CONTRADICTED_QUERY

完整检查了闭合数据集
→ NOT_FOUND_IN_SCOPE

强反证推翻主张
→ FALSE

现有来源不足
→ UNKNOWN_UNRESOLVED

仍存在下一个相关候选组
→ EXPAND_NEXT_GROUP
```

## 8.3 候选组迁移策略

```text
G1 精确匹配
  ↓ 0
G2 记法变体
  ↓ 0
G3 同义词
  ↓ 0
G4 语义相似
  ↓ 0
G5 上位与下位概念
  ↓ 0
G6 关系、原因与结果
  ↓ 0
最终零状态判定
```

每个候选组都具有与原问题之间的语义距离。

```text
distance = 0.00  精确匹配
distance = 0.10  记法变体
distance = 0.20  同义词
distance = 0.35  语义相似
distance = 0.50  上位或下位概念
distance = 0.70  邻接关系
```

距离越大，最终回答中的确定性表述就应越弱。

---

# 9. 可信度计算

候选项可信度可根据以下要素计算：

- `D`: 直接证据强度
- `S`: 来源可信度
- `C`: 条件满足度
- `R`: 时效性
- `X`: 矛盾惩罚
- `G`: 语义距离惩罚

```text
confidence =
D × S × C × R × (1 - X) × (1 - G)
```

建议解释：

| 分数 | 判定 |
|---:|---|
| 0.85以上 | 已确认 |
| 0.65～0.84 | 很可能 |
| 0.40～0.64 | 不确定 |
| 低于0.40 | 从回答候选项中排除，或仅标为推测 |

该分数不是数学上保证的概率，而是用于内部比较和优先级排序的指标。

---

# 10. 回答生成策略

## 10.1 默认回答结构

```text
[直接回答]
证据最强的结论

[判定]
已确认 / 很可能 / 不确定 / 范围内未找到 / 无法判断 / 错误

[验证范围]
检查了哪些来源、日期和地区

[优先排除的主要候选项]
被排除的重要候选项及原因

[剩余不确定性]
仍未得到确认的部分
```

面向普通用户时，应把内部状态代码转换为自然语言，而不是直接暴露。

## 10.2 禁止确定回答的条件

在以下情况下不得生成确定答案：

- 所有候选项均为 `OFF_UNSUPPORTED`
- 存活候选项之间存在强烈矛盾
- 需要最新验证但当前无法取得资料
- 无法判断搜索范围是闭合还是开放
- 核心条件存在歧义且会显著改变结果
- 高风险问题未达到证据阈值

---

# 11. 软件实现设计

## 11.1 建议组件

```text
Frontend
- 问题输入
- 验证范围选择
- 候选网可视化
- 排除原因显示
- 最终回答与证据显示

API
- 问题规范化
- 候选组生成
- 搜索与工具编排
- Negative Gate执行
- 零检测
- 回答生成

Storage
- PostgreSQL: 问题、候选项、状态、证据、执行日志
- Redis: 活跃候选项、候选组队列、临时分数
- Vector DB: 语义相似搜索
- Graph DB或PostgreSQL图模型: 关系遍历
- Object Storage: 原始文件、附件、证据快照

Workers
- exact-search-worker
- semantic-search-worker
- graph-search-worker
- contradiction-worker
- evidence-verification-worker
- freshness-check-worker
```

---

# 12. 数据模型

## 12.1 questions

```sql
CREATE TABLE questions (
    id BIGSERIAL PRIMARY KEY,
    raw_question TEXT NOT NULL,
    normalized_target TEXT,
    normalized_claim TEXT,
    answer_type VARCHAR(32),
    risk_level VARCHAR(16),
    closed_world BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## 12.2 candidate_groups

```sql
CREATE TABLE candidate_groups (
    id BIGSERIAL PRIMARY KEY,
    question_id BIGINT NOT NULL REFERENCES questions(id),
    group_type VARCHAR(32) NOT NULL,
    semantic_distance NUMERIC(6,5) NOT NULL DEFAULT 0,
    sequence_no INTEGER NOT NULL,
    activated_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ
);
```

## 12.3 candidates

```sql
CREATE TABLE candidates (
    id BIGSERIAL PRIMARY KEY,
    group_id BIGINT NOT NULL REFERENCES candidate_groups(id),
    label TEXT NOT NULL,
    state VARCHAR(32) NOT NULL,
    confidence NUMERIC(6,5),
    matched_conditions JSONB NOT NULL DEFAULT '[]',
    failed_conditions JSONB NOT NULL DEFAULT '[]',
    disabled_reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## 12.4 evidence

```sql
CREATE TABLE evidence (
    id BIGSERIAL PRIMARY KEY,
    candidate_id BIGINT NOT NULL REFERENCES candidates(id),
    source_type VARCHAR(32) NOT NULL,
    source_uri TEXT,
    source_title TEXT,
    published_at TIMESTAMPTZ,
    retrieved_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    directness NUMERIC(6,5),
    reliability NUMERIC(6,5),
    freshness NUMERIC(6,5),
    excerpt TEXT
);
```

## 12.5 gate_results

```sql
CREATE TABLE gate_results (
    id BIGSERIAL PRIMARY KEY,
    candidate_id BIGINT NOT NULL REFERENCES candidates(id),
    gate_type VARCHAR(32) NOT NULL,
    passed BOOLEAN NOT NULL,
    resulting_state VARCHAR(32),
    reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

# 13. 核心 API

## 执行问题

```http
POST /api/mesh/questions
Content-Type: application/json
```

```json
{
  "question": "上传的说明书是否明确说明可以离线使用？",
  "scope": {
    "document_ids": [101, 102, 103],
    "closed_world": true
  },
  "mode": "negative_first"
}
```

## 执行结果

```json
{
  "status": "CONFIRMED",
  "answer": "已确认存在首次激活后可离线运行的说明。",
  "scope": {
    "document_ids": [101, 102, 103],
    "closed_world": true
  },
  "active_candidates": [
    {
      "candidate": "首次激活后，无需互联网连接即可运行。",
      "confidence": 0.93
    }
  ],
  "disabled_candidates": [
    {
      "candidate": "云同步功能",
      "state": "OFF_FALSE",
      "reason": "该功能不能证明应用可以离线运行。"
    }
  ],
  "uncertainties": []
}
```

---

# 14. 核心算法

```text
function answer_with_negative_first_mesh(question, sources):

    frame = normalize_question(question)
    groups = build_candidate_groups(frame)

    for group in groups:

        candidates = activate_candidates(group, sources)

        parallel_for candidate in candidates:

            if not definition_matches(candidate, frame):
                disable(candidate, OFF_FALSE)
                continue

            if violates_required_condition(candidate, frame):
                disable(candidate, OFF_FALSE)
                continue

            if outside_time_or_scope(candidate, frame):
                disable(candidate, OFF_OUT_OF_SCOPE)
                continue

            if contradicted(candidate):
                disable(candidate, OFF_CONTRADICTED)
                continue

            if lacks_required_evidence(candidate):
                disable(candidate, OFF_UNSUPPORTED)
                continue

            candidate.state = score_candidate(candidate)

        survivors = get_survivors(candidates)

        if survivors is not empty:
            verified = cross_check(survivors)

            if verified is not empty:
                return compose_answer_from_verified_only(verified)

        zero_state = classify_zero_state(candidates, frame)

        if zero_state == EXPAND_NEXT_GROUP:
            continue

        return report_zero_state(zero_state)

    return {
        "status": "UNKNOWN",
        "answer": "根据当前可用证据，尚无法作出判断。"
    }
```

---

# 15. 与 AI 模型集成

Negative-First Mesh 并不是为了取代生成式 AI。其目标是缩小生成模型的职责，使其主要负责解释已经有证据支持的结果。

## 传统结构

```text
问题
→ 大语言模型
→ 检索或概率推理
→ 生成答案
```

## 建议结构

```text
问题
→ 条件拆解
→ 激活整个候选网
→ 优先排除无效候选项
→ 验证证据
→ 只把存活候选项交给小型或大型语言模型
→ 生成说明
```

### 预期效果

- 减少幻觉
- 减少不必要的 Token 使用
- 缩小检索范围
- 可追踪回答证据
- 提高“未知”判定质量
- 在企业闭合数据中准确判断未找到
- 支持多个 AI 模型之间的交叉验证

---

# 16. 物理网状扩展

软件实现稳定后，可把部分运算转移到物理网状计算设备。

## 16.1 可选介质

- 忆阻器交叉阵列
- FPGA 并行比较网络
- 光学干涉器
- 超导电路
- 自旋波器件
- 神经形态芯片
- 多层 PCB 传输网络
- 频率与相位复用电路

## 16.2 物理节点的作用

```text
输入信号
→ 沿多条路径同时传播
→ 抵消违反条件的路径
→ 增强满足条件的路径
→ 仅保留超过阈值的节点
→ 在输出端测量激活模式
```

## 16.3 虚拟多层化

与其无限堆叠物理电路板，不如在一条物理路径中叠加多个逻辑层：

- 不同频率
- 不同相位
- 不同时间槽
- 不同光波长
- 不同偏振
- 不同编码模式
- 不同方向

这样可以用有限的物理资源构建更大的逻辑状态空间。

---

# 17. 性能目标

## 软件第一阶段

- 对100万候选项进行精确条件的并行排除
- 按候选组执行批量搜索
- 在1秒内完成首次零状态判定
- 证据可追踪率100%
- 通过不存在与未知分类测试

## 软件第二阶段

- 联合处理向量、图与规则
- 同时搜索多个数据源
- 分布式处理数千万候选项
- 实时可视化活跃候选项
- 在调用 LLM 前排除至少90%的候选项

## 硬件实验阶段

- 8×8 或16×16网格
- 验证同相增强
- 验证反相抵消
- 验证多输入阈值判定
- 以电气方式检测零候选状态
- 将 FPGA 控制与候选组迁移逻辑连接

---

# 18. 测试标准

## 事实性

- 是否避免把缺乏证据判定为错误？
- 是否避免把过时信息当作当前事实？
- 是否避免把相关性断言为因果关系？

## 范围

- 是否只在闭合范围内确认不存在？
- 是否说明哪些外部来源没有被检查？
- 是否防止语义距离过大的候选项冒充精确答案？

## 零处理

- 当没有候选项存活时，是否停止生成内容？
- 是否合理扩展到下一候选组？
- 是否区分 `错误`、`范围内未找到` 和 `无法判断`？

## 回答质量

- 是否先给出结论？
- 是否简要说明主要排除项？
- 是否避免暴露不必要的内部推理？
- 是否同时显示证据与不确定性？

---

# 19. 限制

## 19.1 真正的无限状态不可能实现

有限设备无法存储无限多个独立状态。但可以通过频率、相位、时间、空间和连接组合构建极大的状态空间。

## 19.2 物理资源成本

完全并行处理以更多内存、连接、电力和芯片面积换取更短的计算时间。

## 19.3 全连接图的扩展问题

随着节点数量增加，连接数会快速增长。实际系统需要采用分层结构：

- 局部网
- 领域专用网
- 上层摘要网
- 稀疏连接
- 仅在需要时激活的路径
- 通过波长或频率复用形成虚拟连接

## 19.4 语义比较的不确定性

与精确键匹配不同，语义比较具有概率性。因此，语义相似结果在验证前必须保持为暂定候选项。

## 19.5 证明不存在的困难

在开放世界中，未找到并不能证明不存在。Negative-First Mesh 必须把这种限制明确显示为结果状态。

---

# 20. 开发阶段

## Phase 1. 技能与规则引擎

- 问题规范化
- 候选状态模型
- Negative Gate
- 零检测
- 候选组迁移
- 回答模板
- 测试场景

## Phase 2. 检索编排器

- 精确搜索
- 语义搜索
- 图搜索
- 文件搜索
- 网页搜索
- 数据库搜索
- 交叉验证

## Phase 3. 运营平台

- 问题执行历史
- 候选网可视化
- 门控通过与排除原因
- 证据管理
- 管理员可配置阈值
- 领域专用策略配置
- 审计日志

## Phase 4. AI 集成

- 多 LLM 候选生成
- 候选项之间的矛盾检查
- 自动关闭无证据候选项
- 只把存活上下文交给 LLM
- 回答可信度与不存在判定

## Phase 5. 硬件实验

- 基于 FPGA 的候选项并行比较
- 模拟信号增强与抵消
- 忆阻器或光学网实验
- 软件 Mesh Engine 与硬件加速器集成

---

# 21. 项目结构

```text
negative-first-mesh-skill/
├── README.md
├── README.ko.md
├── README.en.md
├── README.ja.md
├── README.zh-CN.md
├── SKILL.md
├── SKILL.ko.md
├── SKILL.en.md
├── SKILL.ja.md
├── SKILL.zh-CN.md
├── INSTALL.md
├── INSTALL.ko.md
├── INSTALL.en.md
├── INSTALL.ja.md
├── INSTALL.zh-CN.md
├── EXAMPLES.md
├── EXAMPLES.ko.md
├── EXAMPLES.en.md
├── EXAMPLES.ja.md
├── EXAMPLES.zh-CN.md
├── TESTS.md
├── TESTS.ko.md
├── TESTS.en.md
├── TESTS.ja.md
├── TESTS.zh-CN.md
└── packages/
    ├── negative-first-mesh-ko.zip
    ├── negative-first-mesh-en.zip
    ├── negative-first-mesh-ja.zip
    └── negative-first-mesh-zh-CN.zip
```

---

# 22. 一句话定义

> Negative-First Mesh Computing 是一种计算与推理架构。它将问题传播到整个候选网，优先抑制不匹配、相互矛盾、超出范围和缺乏证据的候选项，然后只基于存活证据作答，或者准确分类活跃候选数归零的原因。

---

# 23. 核心原则

```text
不要先生成答案。
先展开候选域。
优先关闭无效项。
不要把零视为失败。
达到零时，转到下一候选组。
合理候选组全部归零后，不得编造。
只陈述存活证据能够支持的内容。
区分错误、不存在与未知。
```
---

# 24. 应用于现有LLM

本技能可以原生安装到Codex、Claude Code和Gemini CLI。

| 目标 | 项目安装位置 | 用户级安装位置 |
|---|---|---|
| Codex | `.agents/skills/negative-first-mesh/SKILL.md` | `~/.agents/skills/negative-first-mesh/SKILL.md` |
| Claude Code | `.claude/skills/negative-first-mesh/SKILL.md` | `~/.claude/skills/negative-first-mesh/SKILL.md` |
| Gemini CLI | `.gemini/skills/negative-first-mesh/SKILL.md` 或 `.agents/skills/...` | `~/.gemini/skills/negative-first-mesh/SKILL.md` 或 `~/.agents/skills/...` |

选择一个语言版技能文件，并在安装位置准确命名为 `SKILL.md`。不支持原生技能的LLM可通过项目指令或系统/开发者提示词应用。

详细方法见 [INSTALL.zh-CN.md](INSTALL.zh-CN.md)。
