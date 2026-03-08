# query_graph.py — 五个查询模式详解

> **什么时候读这个文件**：当你需要用 `query_graph.py` 做精准局部查询时——
> 无论是在 PROBE 的 REASON/OBJECT/EMIT 阶段，还是在日常开发中辅助决策。

`query_graph.py` 的定位是"放大镜"——`.nexus-map/` 是地图，它是在地图上做精准查询的工具。
**零额外依赖**，纯标准库，输入 `ast_nodes.json` 即可运行。

---

## --file \<路径\> — 文件骨架解剖

```bash
python query_graph.py ast_nodes.json --file <path>
python query_graph.py ast_nodes.json --file <path> --git-stats git_stats.json
```

**输出**：指定文件的完整骨架——类、方法、行号范围、所有 import（区分内部/外部，内部 import 解析成真实路径）。加 `--git-stats` 追加变更热度和耦合文件。

**深层价值**：AI 不读源码也能掌握文件结构，精确到行号；针对大型遗留代码库（"屎山"）尤其有效——一个 3000 行的 legacy 模块可能有数十个类，逐行阅读效率极低，`--file` 秒出骨架，再按行号精准跳转。

**适用场景**：
- 接手超大遗留模块，建立结构地图再动手，避免盲目深入
- 连续重构任务中，确认当前目标文件的完整方法签名，避免改了一处遗漏其他重载
- Bug 调查：快速确认候选类/函数的行号范围，缩小 `view_file` 的读取区间
- Code Review 辅助：确认一个 PR 涉及的方法是否在预期边界内
- EMIT 阶段：为 `dependencies.md` 生成精确的模块结构数据

---

## --who-imports \<路径或模块名\> — 反向依赖追踪

```bash
python query_graph.py ast_nodes.json --who-imports <module_or_path>
```

**输出**：反向查询——所有引用了该模块的文件（区分源码文件和测试文件）。

**深层价值**：改接口之前唯一必须跑的命令。任何修改公共函数签名、删除方法、重命名类的操作如果跳过这步，都是在赌不会炸。

**适用场景**：
- 删函数/改签名/迁移模块前，确认「炸弹清单」——列出所有需要同步修改的调用方
- 连续开发任务中，修改了 step 1 的接口后，确认后续步骤哪些受影响，规划工作顺序
- 决定测试策略：知道谁依赖了目标模块，才能做精准回归测试而不是全量跑
- 评估「是否值得重构这个老接口」：0 个依赖方 = 可以随意重构，50 个依赖方 = 需要完整迁移计划
- OBJECT 阶段：验证「这个模块是否真的是边界清晰的孤立组件」

---

## --impact \<路径\> [--git-stats] — 影响半径量化

```bash
python query_graph.py ast_nodes.json --impact <path>
python query_graph.py ast_nodes.json --impact <path> --git-stats git_stats.json
```

**输出**：同时看两个方向——这个文件 import 了谁（上游依赖），谁 import 了这个文件（下游被依赖），给出「X upstream, Y downstream」影响半径数字。加 `--git-stats` 追加 git 风险等级和耦合对。

**深层价值**：最有实战价值的单一命令。`0 upstream, 24 downstream` 一眼告诉你这是基础层，改动影响最广。与 git-stats 叠加后：**high-risk + high downstream = 当前最危险的改动点**。

**适用场景**：
- 衡量一个功能修改的风险与实际工作量，在 Sprint 估时/技术债偿还决策时有直接参考价值
- 评估「当前改动是局部手术还是全局手术」——downstream 数字决定了测试范围
- 架构评审时，对候选重构模块做影响量化：改哪个代价最小？
- 遗留系统拆分规划：downstream 很高的模块先不动，从边缘模块开始清理
- OBJECT 阶段：验证系统边界假设，`0 upstream` 证实基础层，`24 downstream` 证实高影响

---

## --hub-analysis [--top N] — 架构核心节点识别

```bash
python query_graph.py ast_nodes.json --hub-analysis
python query_graph.py ast_nodes.json --hub-analysis --top 10
```

**输出**：扫描整个项目，按扇入（被多少模块引用）和扇出（引用了多少模块）排序，找出真正的核心节点。

**深层价值**：目录名 ≠ 重要性。命名为 `core/` 的不一定是实际核心；真正的高耦合节点往往藏在不起眼的工具类、数据模型或配置模块里。扇入高 + git-stats risk high = 最值得重构的目标。

**适用场景**：
- REASON 阶段：用数据验证「哪个模块才是真正的核心」而不是凭目录名猜测
- 架构评审：找出「明星模块」——被所有人依赖却从不在路线图里出现的那个
- 技术债优先级排序：扇入高 + change frequency 高 = 最值得治理的目标
- 遗留代码清理入口：扇出极高但扇入很低的模块，是可以安全替换或拆分的「孤立节点」
- 新人接手项目：先看扇入 Top 5，这几个模块搞懂了，项目的一半架构就清楚了

---

## --summary — 全局目录聚合

```bash
python query_graph.py ast_nodes.json --summary
```

**输出**：按顶层目录分区统计模块数/类数/函数数/行数，附带各区域的 import 方向关系。

**深层价值**：用一条命令建立系统分层意识。哪个目录是「业务逻辑层」、哪个是「基础设施层」、哪个是「测试层」，从 import 关系里直接读出来比读任何文档都客观。

**适用场景**：
- EMIT 阶段：为 `systems.md` / `dependencies.md` 提供客观数据支撑
- 项目初次接触（5 秒建立全局认知）：比读 README 更客观，比手动 ls 更结构化
- 识别循环依赖风险区域：两个顶层目录互相 import 是架构坏味道，`--summary` 立刻暴露
- 评估测试覆盖均衡性：tests 目录的函数数 vs src 目录的函数数，比例是否合理

---

## 使用时机速查

| 你此刻的问题 | PROBE 阶段 | 开发中 |
|-------------|-----------|--------|
| 这个文件有哪些类/方法，各在哪几行 | EMIT | ✅ 改动前摸底 |
| 改这个接口/删函数，哪些文件跟着改 | — | ✅ 必检，否则炸 |
| 这个改动最终影响多少模块 | OBJECT | ✅ 估工作量 |
| 这个改动 + git 热度 = 风险有多高 | OBJECT | ✅ Sprint 决策 |
| 项目中谁是真正的核心依赖节点 | REASON | ✅ 架构评审 |
| 整个项目的模块分布和层级 | EMIT | ✅ 项目交接 |
| 连续重构，改完一处查影响链 | — | ✅ 顺序：`--who-imports` → `--impact` |

---

## 前提说明

`ast_nodes.json` 来自 `extract_ast.py`，通常位于 `.nexus-map/raw/ast_nodes.json`。
`git_stats.json`（可选）来自 `git_detective.py`，通常位于 `.nexus-map/raw/git_stats.json`。

可以用绝对路径或相对路径直接指定：

```bash
python $SKILL_DIR/scripts/query_graph.py /tmp/ast_nodes.json --file src/core/vision.py
```
