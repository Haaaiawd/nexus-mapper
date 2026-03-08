<p align="center">
  <img src="Icon.png" alt="nexus-skills" width="96" height="96">
</p>

<h1 align="center">nexus-skills</h1>

<p align="center">
  为 AI Agent 生成可持续复用的代码仓库知识库，并提供精准的代码结构查询能力。<br>
  Build a persistent knowledge map of any codebase. Query its structure with precision. Every AI session starts smarter.
</p>

<p align="center">
  <a href="README.md">English</a>
</p>

---

## 两个技能，一种设计哲学

| 技能 | 干什么 | 什么时候用 |
|------|--------|-----------|
| **nexus-mapper** | 全量分析代码仓库，写出持久化的 `.nexus-map/` 知识库供后续 AI 会话使用 | 接手陌生仓库、团队协作交接、大型重构开始前 |
| **nexus-query** | 直接从 AST 数据查询文件结构、反向依赖、改动影响半径和耦合热点 | 开发过程中——修改接口前、重构 Sprint 期间、导航遗留代码库时 |

两个技能共用同一套底层脚本。`nexus-query` 可以直接复用 `nexus-mapper` 产出的 `.nexus-map/`，也可以按需独立生成 `ast_nodes.json`。

---

## nexus-mapper

nexus-mapper 是一个给 AI Agent 用的仓库建图 skill。它分析本地代码库，写出持久化的 `.nexus-map/` 知识库，让后续会话先恢复全局上下文，再进入具体任务，而不是每次都从零摸索。

它不是一个泛泛的"总结仓库"提示词。这个 skill 会按 PROBE 协议分阶段执行，先产出证据，再挑战初始判断，最后才写正式资产。它解决的是 AI 最常见的一个问题：把第一眼印象误写成结论。

```
.nexus-map/
├── INDEX.md              ← 冷启动入口。完整架构上下文，控制在 2000 tokens 以内。
├── arch/
│   ├── systems.md        ← 每个子系统的职责和代码位置。
│   ├── dependencies.md   ← 组件间的调用关系，Mermaid 依赖图。
│   └── test_coverage.md  ← 静态测试面：哪些核心模块有测试、哪些没有、哪里证据不足。
├── concepts/
│   ├── concept_model.json ← 机器可读的知识图谱，供程序化使用。
│   └── domains.md        ← 这个代码库使用的领域语言，人能读懂的版本。
├── hotspots/             ← 仅在存在 git 元数据时生成。
│   └── git_forensics.md  ← 变更最频繁的文件，以及总是同时变更的文件对。
└── raw/                  ← 原始数据：AST 节点、git 统计、过滤后的文件树。
```

`INDEX.md` 是唯一的冷启动入口，刻意保持很小。AI 可以一次性完整读入，先恢复全局，再按需下钻。

---

## nexus-query

nexus-query 在开发过程中回答精准的结构问题——不用读完整个仓库，也不用重跑整个 PROBE 流程。

```bash
# 文件骨架：类、方法、行号、import 清单
python skills/nexus-query/scripts/query_graph.py ast_nodes.json --file src/core/vision.py

# 反向依赖：谁在引用这个模块？
python skills/nexus-query/scripts/query_graph.py ast_nodes.json --who-imports src.core.vision

# 影响半径：上游依赖 + 下游被依赖者
python skills/nexus-query/scripts/query_graph.py ast_nodes.json --impact src/core/vision.py \
  --git-stats git_stats.json

# 架构核心节点：最高扇入/扇出的模块
python skills/nexus-query/scripts/query_graph.py ast_nodes.json --hub-analysis

# 按顶层目录聚合的结构摘要
python skills/nexus-query/scripts/query_graph.py ast_nodes.json --summary
```

零额外依赖。纯 Python 标准库。`ast_nodes.json` 可以来自已有的 `.nexus-map/raw/`，也可以按需现场生成。

**最有价值的场景**：
- 改接口前：`--who-imports` 告诉你哪些地方会炸
- Sprint 估时前：`--impact --git-stats` 量化风险和工作量
- 遗留代码库里：`--file` 秒出骨架，不用读 3000 行
- 架构评审时：`--hub-analysis` 找出真正的耦合核心，而不是名叫 `core/` 的那个

---

## 它们为什么不一样

- **阶段门控**：nexus-mapper 的 PROFILE、REASON、OBJECT、BENCHMARK、EMIT 都不能跳。
- **诚实的 provenance**：两个技能都强制区分 `implemented`、`planned`、`inferred`。解析不完整时也必须说明。
- **可组合**：nexus-query 可以独立使用，也可以叠加在 nexus-mapper 的地图上。
- **面向未来的会话**：每个产物都是为了下次能更快进入状态而设计的。

---

## 前提条件

| 要求 | 说明 |
|------|------|
| Python 3.10+ | `python --version` |
| Shell 执行能力 | AI 客户端需支持运行终端命令 |

有 git 历史会更完整，但不是必须的。没有 git 历史时，`hotspots/` 分析会跳过，其余照常运行。

**首次使用前安装脚本依赖：**

```bash
# nexus-mapper
pip install -r skills/nexus-mapper/scripts/requirements.txt

# nexus-query（同一套依赖）
pip install -r skills/nexus-query/scripts/requirements.txt
```

---

## 安装

```bash
# 完整套件
npx skills add haaaiawd/nexus-skills

# 或只安装你需要的技能
npx skills add haaaiawd/nexus-skills/skills/nexus-mapper
npx skills add haaaiawd/nexus-skills/skills/nexus-query
```

适配 Claude Code、GitHub Copilot、Cursor、Cline，以及所有支持 `SKILL.md` 协议的 AI 客户端。

---

## 怎么使用 nexus-mapper

把本地仓库路径告诉你的 AI：

```
帮我分析 /Users/me/projects/my-app 并生成知识库
```

AI 跑完整个协议后，会在仓库根目录写入 `.nexus-map/`。下次打开这个项目时，直接说：

```
读取 .nexus-map/INDEX.md
```

为了让这种行为在长期更稳定，建议把一小段持久规则写进 `AGENTS.md`、`CLAUDE.md` 或类似文件：

```md
如果仓库中存在 .nexus-map/INDEX.md，开始任务前必须先阅读它恢复全局上下文。

如果任务需要判断局部结构、依赖关系、影响半径或边界归属，使用 query_graph.py 基于 .nexus-map/raw/ast_nodes.json 做验证；不要重新猜结构。

当一次任务改变了项目的结构认知时，应在交付前评估是否同步更新 .nexus-map。
```

---

## 语言支持

按文件扩展名自动 dispatch，支持 30+ 语言：

Python · JavaScript · JSX · TypeScript · TSX · Bash · Java · Go · Rust · C++ · C · C# · Kotlin · Ruby · Swift · Scala · PHP · Lua · Elixir · GDScript · Dart · Haskell · Clojure · SQL · Proto · Solidity · Vue · Svelte · R · Perl

这些语言的覆盖深度并不完全相同：有些是完整结构提取，有些只有 Module 级别。最终输出里的 metadata 会诚实标出这一点。

### 扩展语言支持

```bash
python skills/nexus-mapper/scripts/extract_ast.py <repo_path> \
  --add-extension .templ=templ \
  --add-query templ struct "(component_declaration name: (identifier) @class.name) @class.def"
```

---

## 仓库结构

```
nexus-skills/
├── README.md
├── README.zh-CN.md
├── Icon.png
├── evals/
└── skills/
    ├── nexus-mapper/
    │   ├── SKILL.md              ← 执行协议与守则
    │   ├── scripts/
    │   │   ├── extract_ast.py    ← 多语言 AST 提取器
    │   │   ├── query_graph.py    ← 按需 AST 查询工具
    │   │   ├── git_detective.py  ← Git 热点与耦合分析
    │   │   ├── languages.json    ← 语言配置
    │   │   └── requirements.txt
    │   └── references/
    │       ├── 01-probe-protocol.md
    │       ├── 02-output-schema.md
    │       ├── 03-edge-cases.md
    │       ├── 04-object-framework.md
    │       ├── 05-language-customization.md
    │       └── 06-query-guide.md
    └── nexus-query/
        ├── SKILL.md              ← 查询模式与使用场景
        └── scripts/
            ├── extract_ast.py
            ├── query_graph.py
            ├── git_detective.py
            ├── languages.json
            └── requirements.txt
```

---

## License

MIT

