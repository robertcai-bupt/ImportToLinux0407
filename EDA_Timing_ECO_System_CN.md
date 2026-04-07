# AI辅助EDA时序Signoff修复系统

## 项目概述

基于 hipilot 多Agent平台，构建AI辅助的EDA时序ECO（工程变更）系统，解决signoff阶段修时序的三个核心痛点。

**技术栈**: Cadence (Innovus/PTEco/Xtop) + Synopsys (PrimeTime)
**数据格式**: PT/RC (Galois/ETM), SPEF, SDC
**目标场景**: Signoff / PTEco / Xtop / Route后的Eco

---

## 一、系统架构

### 1.1 Agent组成

| Agent                     | 名称          | 核心职责                |
| ------------------------- | ----------- | ------------------- |
| `timing-eco-orchestrator` | 时序ECO统筹     | 统筹协调整个时序ECO流程，质量门控  |
| `timing-data-parser`      | 时序数据解析      | 解析PT/RC、SPEF、SDC等格式 |
| `violation-analyzer`      | 违例分析        | 违例根因识别与分类           |
| `timing-comparator`       | 时序对比        | 版本/阶段对比分析           |
| `eco-strategy-generator`  | ECO策略生成     | 生成可执行ECO策略          |
| `cadence-tool-integrator` | Cadence工具接口 | Cadence工具链命令集成      |
| `signoff-quality-gate`    | Signoff质量门  | Signoff质量验证         |

### 1.2 数据流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    时序ECO统筹 (Orchestrator)                    │
└─────────────────────────────────────────────────────────────────┘
                                  │
         ┌────────────────────────┼────────────────────────┐
         ▼                        ▼                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  时序数据解析    │    │    违例分析      │    │   ECO策略生成    │
│   (Parser)      │───▶│   (Analyzer)     │───▶│  (Strategy)     │
│                 │    │                 │    │                 │
│ - PT/RC解析     │    │ - 违例分组       │    │ - 策略优先级     │
│ - 单位归一化(ps) │    │ - 根因识别       │    │ - 风险评估       │
│ - SPEF/SDC     │    │ - 模式识别       │    │ - Cadence命令   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                              │
         ┌────────────────────────────────────────────────────┘
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Cadence工具接口 (Integrator)                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ Innovus │  │ PrimeTime │  │  PTEco   │  │   Xtop   │       │
│  │ (C)     │  │   (S)    │  │   (C)    │  │   (C)    │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Signoff质量门 (Quality Gate)                  │
│   WNS≥0  │  TNS=0  │  Setup Margin>10ps  │  Hold Margin>5ps   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、三个核心痛点解决方案

### 痛点一：海量数据中识别违例根因

**问题**: 时序违例数据海量且多种类型（单位不同），难以快速识别根因并获得可执行迭代策略。

**解决流程**:

```
Step 1: 时序数据解析
   ├── 解析PT/RC (Galois/ETM) 格式
   ├── 解析SPEF寄生参数
   ├── 解析SDC约束
   └── 归一化单位到ps

Step 2: 违例分类分析
   ├── 按path_type分组 (setup/hold)
   ├── 按clock_domain分组
   ├── 按severity分组 (critical/major/minor)
   └── 模式识别 (逻辑深度/电容/时钟域)

Step 3: 根因归因
   ├── LOGIC_DEPTH: 逻辑深度问题 → 布尔优化
   ├── CELL_SIZING: Cell尺寸问题 → 放大/替换
   ├── NET_ROUTING: 布线问题 → 屏蔽/换层
   ├── CLOCK_TREE: 时钟树问题 → CTS平衡
   ├── CDC_ISSUE: 跨时钟域 → 同步器
   └── LIBRARY_ISSUE: 库模型问题 → 库更新

Step 4: ECO策略生成
   ├── 优先级: impact/effort矩阵
   ├── 风险评估: LOW/MEDIUM/HIGH
   └── Cadence命令: Innovus/PrimeTime具体命令
```

**Agent协作**:

- `Timing Data Parser` → `Violation Analyzer` → `ECO Strategy Generator` → `Cadence Tool Integrator` → `Signoff Quality Gate`

---

### 痛点二：版本/阶段对比分析

**问题**: 对比修复前后或不同阶段的设计+时序数据，需要识别趋势差异的根因。

**解决流程**:

```
Version A (基准)          Version B (对比)
     │                        │
     └────────┬───────────────┘
              ▼
┌───────────────────────────────────────┐
│         并行解析 (Parser x2)           │
└───────────────────────────────────────┘
              │
              ▼
┌───────────────────────────────────────┐
│           时序对比 (Comparator)         │
│                                       │
│  1. Metric Delta计算                   │
│     - violation_count_delta           │
│     - WNS_delta / TNS_delta          │
│     - 改善百分比                       │
│                                       │
│  2. 违例分类                          │
│     - FIXED: 修复的违例               │
│     - NEW: 新增违例                    │
│     - DEGRADED: 恶化的违例             │
│     - IMPROVED: 改善的违例             │
│                                       │
│  3. 关键路径迁移分析                   │
│     - Same path, different timing     │
│     - Path segment changed            │
│     - Completely new path             │
│                                       │
│  4. 根因归因                          │
│     - 与已知ECO行动的相关性            │
│     - 意外变化的调查                   │
└───────────────────────────────────────┘
              │
              ▼
┌───────────────────────────────────────┐
│           对比报告输出                  │
│                                       │
│  | Metric | A | B | Delta | % |      │
│  |--------|---|---|------|---|        │
│  | WNS(ps)|-150|-45| +105 |70%|      │
│  | TNS(ps)|-4500|-890|+3610|80%|     │
│  | Violations|127|45| -82 |65%|      │
│                                       │
│  Assessment: PASS/FAIL/CONDITIONAL   │
└───────────────────────────────────────┘
```

---

### 痛点三：扫尾阶段少量违例深度分析

**问题**: 多轮ECO后剩余少量违例（<50个），需要数据分析获得具体修复策略。

**解决流程**:

```
剩余违例 (<50个)
      │
      ▼
┌───────────────────────────────────────┐
│       深度分析 (Violation Analyzer)     │
│              DEEP DIVE MODE            │
│                                       │
│  1. 逐条路径提取                        │
│     - Cell-by-Cell delay分析           │
│     - Net-by-Net capacitance分析       │
│     - 时钟路径skew分析                  │
│                                       │
│  2. 多违例关联分析                      │
│     - 同一net影响多个endpoint?          │
│     - 同一时钟域影响?                   │
│     - 库corner问题?                    │
│     - 物理邻近性问题(拥塞)?             │
│                                       │
│  3. 库模型对比                          │
│     - 实际delay vs 库模型               │
│     - 定位边界工作情况                   │
│                                       │
│  4. 输出:                              │
│     - 具体根因 (不是"LOGIC_DEPTH")      │
│     - 精准修复策略                      │
│     - 置信度评分                        │
│     - 手动ECO标记                       │
└───────────────────────────────────────┘
      │
      ▼
┌───────────────────────────────────────┐
│    精准ECO策略生成 (ECO Strategy)        │
│         FINAL CLEANUP MODE             │
│                                       │
│  - 具体Cell修改: A→B                  │
│  - 具体Buffer插入: 位置+类型           │
│  - 具体布线修改: 层+屏蔽              │
│  - 批次优化: 一次修改解决多个违例      │
│  - 手动ECO标记: 高风险项               │
└───────────────────────────────────────┘
      │
      ▼
┌───────────────────────────────────────┐
│         Cadence命令执行                 │
│                                       │
│  ecoAddBuffer -cell BUFx4 -loc {X Y}  │
│  ecoChangeCell -cell XXX -to YYY      │
│  routeEco -net XXX -shielding VDD     │
└───────────────────────────────────────┘
      │
      ▼
┌───────────────────────────────────────┐
│         Signoff质量门验证               │
│                                       │
│  GO / NO-GO / CONDITIONAL            │
└───────────────────────────────────────┘
```

---

## 三、根因分类体系

### 3.1 按路径类型分类

| 类型           | 子类                 | 根因                          |
| ------------ | ------------------ | --------------------------- |
| Setup违例      | Data Path过长        | 逻辑深度过长、Net电容过大、Transition退化 |
|              | Clock Path Skew    | CTS不平衡、布线不对称                |
| Hold违例       | Data Path过短        | Min-delay不足、串扰              |
|              | Clock Path Advance | Hold预算被消耗                   |
| Transition违例 | -                  | Driver弱、负载电容过大              |
| Fanout违例     | -                  | 超过最大fanout                  |
| CDC违例        | -                  | 亚稳态风险、重汇聚问题                 |

### 3.2 按根因类别分类

| 类别            | 描述          | ECO策略                    |
| ------------- | ----------- | ------------------------ |
| BUFFER_SIZING | Buffer插入或放大 | ecoAddBuffer             |
| CELL_SIZING   | Cell尺寸调整    | ecoChangeCell            |
| NET_ROUTING   | 布线优化        | routeEco, shielding      |
| CLOCK_TREE    | 时钟树优化       | CTS balance, useful skew |
| LOGIC_DEPTH   | 逻辑优化        | Boolean优化、重定时            |
| CDC_ISSUE     | 跨时钟域        | 同步器插入                    |
| LIBRARY_ISSUE | 库问题         | 库更新、Cell替换               |
| PHYSICAL      | 物理问题        | placement优化、拥塞缓解         |

---

## 四、数据结构

### 4.1 违例记录 (Violation Record)

```markdown
## 标识
- violation_id: 唯一标识
- endpoint: 路径端点
- path_type: setup/hold/transition/fanout/cdc
- clock_domain: 时钟域名称

## 时序值
- slack: FLOAT (ps, 负值=违例)
- arrival_time: FLOAT (ps)
- required_time: FLOAT (ps)
- path_delay: FLOAT (ps)
- clock_skew: FLOAT (ps)

## 分类
- severity: critical/major/minor
- root_cause_category: 根因类别
- root_cause_confidence: 0-100%
- affected_nets: 受影响net列表
- affected_cells: 受影响cell列表

## 上下文
- violation_path: 完整时序路径
- path_depth: 逻辑级数
- suggested_strategies: 建议策略(优先级排序)
```

### 4.2 ECO策略记录 (ECO Strategy Record)

```markdown
## 标识
- strategy_id: 策略标识
- target_violation_ids: 目标违例列表
- priority: 优先级 (1=最高)
- strategy_type: buffer/sizing/routing/clock/logic/physical

## 分类
- impact_score: FLOAT (0-100, 预期改善)
- effort_score: INTEGER (1-5, 复杂度)
- risk_score: FLOAT (0-100, 回归风险)
- confidence: FLOAT (0-100, 置信度)

## Cadence命令
- Innovus: ecoAddBuffer, ecoChangeCell, routeEco
- PrimeTime (Synopsys): report_timing, timeDesign
- PTEco: eco_plan, timing_eco
```

---

## 五、Cadence工具链集成

### 5.1 工具对应表

| 工具      | 角色      | 关键命令                                        |
| ------- | ------- | ------------------------------------------- |
| Innovus | 布局布线    | `ecoAddBuffer`, `ecoChangeCell`, `routeEco` |
| PrimeTime | 静态时序分析 (Synopsys)  | `report_timing`, `timeDesign`               |
| PTEco   | 时序ECO规划 | `eco_plan`, `timing_eco`, `verify_timing`   |
| Xtop    | 高级ECO   | `xtop_eco`, `route_eco_opt`                 |

### 5.2 常用命令示例

```tcl
# Buffer插入
ecoAddBuffer -cell BUFx4 -loc {1250 3400} -net net_mem_data

# Cell替换
ecoChangeCell -cell DFF_SYS_0 -to DFFX4

# 布线屏蔽
routeEco -net net_critical -shielding VDD

# 时序报告
report_timing -path_type full -endpoint DFF_SYS_0/D

# ECO验证
verify_timing -eco
```

---

## 六、工作流文件

| 文件                                 | 针对痛点         |
| ---------------------------------- | ------------ |
| `workflow-timing-eco-signoff.md`   | 痛点1：海量数据根因识别 |
| `workflow-timing-comparison.md`    | 痛点2：版本/阶段对比  |
| `workflow-timing-final-cleanup.md` | 痛点3：扫尾阶段精准修复 |

---

## 七、质量门标准

| 指标           | Signoff阈值    | 内部目标         |
| ------------ | ------------ | ------------ |
| WNS (Setup)  | ≥ 0 ps       | > 10 ps      |
| WNS (Hold)   | ≥ 0 ps       | > 5 ps       |
| TNS (Setup)  | = 0 ps       | 0 ps         |
| TNS (Hold)   | = 0 ps       | 0 ps         |
| Setup Margin | > 10 ps      | > 20 ps      |
| Hold Margin  | > 5 ps       | > 10 ps      |
| Clock Skew   | < PVT允许值     | < 50% of PVT |
| DRC          | 0 violations | 0 violations |

---

## 八、调用示例

### 完整Pipeline调用

```
激活 TimingECO Orchestrator

项目: [芯片名称]
阶段: PostRoute Signoff
工具链: Cadence (Innovus/PTEco) + Synopsys (PrimeTime)
数据格式: PT/RC (Galois/ETM)

输入数据:
- 时序数据: /project/timing/pt_rc_output/
- SPEF文件: /project/timing/parasitics/
- SDC约束: /project/design/constraints/
- 前序ECO: /project/eco/v2_iteration/

解决痛点: [1_根因识别 | 2_版本对比 | 3_扫尾清理]

执行对应工作流，输出:
1. 违例分析与根因
2. 优先级排序的ECO策略
3. 实施方案
4. Signoff就绪评估
```
