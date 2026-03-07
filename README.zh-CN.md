<p align="center">
  <img src="Icon.png" alt="nexus-mapper" width="96" height="96">
</p>

<h1 align="center">nexus-mapper</h1>

<p align="center">
  为 AI Agent 生成可持续复用的代码仓库知识库。<br>
  新开对话窗口，直接上下文接续，无需重新探索。
</p>

<p align="center">
  <a href="README.md">English</a>
</p>

---

## 你能得到什么

对一个仓库运行一次 nexus-mapper，它会在项目根目录写入 `.nexus-map/` 知识库，记录架构边界、系统分工、高频变更文件和依赖关系图。之后每次打开新对话，AI 读一个文件就知道整个项目的全貌。

它不会把第一眼猜测直接写成结论。正式产出前，协议会先用代码证据和 git 线索挑战初始判断，宁可少报，也不为了“看起来全面”而凑问题或装确定。

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
│                           改动这些地方最容易出问题。
└── raw/                  ← 原始数据：AST 节点、git 统计、过滤后的文件树。
```

`INDEX.md` 是唯一的入口，刻意做得很小——AI 可以完整加载它，不会截断，需要深入时再按需读取其他文件。

所有生成的 Markdown 文件都应带一个简短 provenance 头部，至少写明 `verified_at` 和降级说明。若仓库包含当前未支持的语言，或某些语言只有 Module 级 AST 覆盖，nexus-mapper 都必须显式说明，不能夸大解析可信度。

如果仓库需要补充超出内建范围的语言支持，agent 应先通过命令行参数扩展这次分析，例如 `--add-extension` 和 `--add-query`。只有当配置较长、query 过于复杂时，再改用 `--language-config <JSON_FILE>` 作为显式输入。

---

## 前提条件

| 要求 | 说明 |
|------|------|
| Python 3.10+ | `python --version` |
| Shell 执行能力 | AI 客户端需支持运行终端命令 |

有 git 历史会更完整，但不是必须的。没有 git 历史时，`hotspots/` 的分析会被跳过，其余正常运行。

**首次使用前安装脚本依赖：**

```bash
pip install -r skills/nexus-mapper/scripts/requirements.txt
```

---

## 安装

```bash
npx skills add haaaiawd/nexus-mapper
```

适配 Claude Code、GitHub Copilot、Cursor、Cline，以及所有支持 `SKILL.md` 协议的 AI 客户端。

---

## 使用方式

把本地仓库路径告诉你的 AI：

```
帮我分析 /Users/me/projects/my-app 并生成知识库
```

AI 完成分析后会在仓库根目录写入 `.nexus-map/`。下次打开这个项目时，直接说：

```
读取 .nexus-map/INDEX.md
```

完整的上下文就这么接续了。

为了让这种行为在长期更稳定，建议把一小段持久规则写进宿主工具的记忆文件，例如 `AGENTS.md`、`CLAUDE.md` 或类似文件：

```md
如果仓库中存在 .nexus-map/INDEX.md，开始任务前优先阅读它恢复项目上下文。

当一次任务改变了项目的结构认知时，应在交付前评估是否同步更新 .nexus-map。结构认知包括：系统边界、入口、依赖关系、测试面、语言支持、路线图或阶段性进度事实。纯局部实现细节默认不更新。

不要把 .nexus-map 视为静态文档；它是项目记忆的一部分。新对话优先读取，重要变更后按需同步。
```

---

## 语言支持

按文件扩展名自动 dispatch，支持 17+ 语言：

Python · JavaScript · JSX · TypeScript · TSX · Bash · Java · Go · Rust · C++ · C · C# · Kotlin · Ruby · Swift · Scala · PHP · Lua · Elixir · GDScript · Dart · Haskell · Clojure · SQL · Proto · Solidity · Vue · Svelte · R · Perl

这些语言的覆盖深度并不完全相同：有些是完整结构提取，有些当前只有 Module 级别，还有些虽然被显式要求支持了，但当前环境无法加载 parser。最终输出里的 metadata 会诚实标出这一点。

未知扩展名静默跳过。多语言混合仓库无需任何配置。

### 扩展语言支持

如果内建覆盖还不够，优先直接在命令行扩展本次运行：

```bash
python skills/nexus-mapper/scripts/extract_ast.py <repo_path> \
  --add-extension .templ=templ \
  --add-query templ struct "(component_declaration name: (identifier) @class.name) @class.def"
```

如果配置较多，不适合直接写在一条命令中，再显式传入 JSON 文件：

```json
{
  "extensions": {
    ".templ": "templ",
    ".gd": "gdscript"
  },
  "queries": {
    "templ": {
      "struct": "(component_declaration name: (identifier) @class.name) @class.def",
      "imports": ""
    }
  },
  "unsupported_extensions": {
    ".legacydsl": "legacydsl"
  }
}
```

```bash
python skills/nexus-mapper/scripts/extract_ast.py <repo_path> \
  --language-config /custom/path/to/language-config.json
```

这样所有语言都走同一套契约：有 parser 且有 query 就是 `structural coverage`，只有 parser 没有 query 就是 `module-only`，agent 明确要求支持但当前环境加载不到 parser 就是 `configured-but-unavailable`，明确标记不支持的仍然归为 `unsupported`。

---

## 仓库结构

```
nexus-mapper/
├── README.md
├── README.zh-CN.md
├── Icon.png
├── evals/                        ← 评测集与测试计划，用于持续迭代 skill
└── skills/
  └── nexus-mapper/
    ├── SKILL.md              ← 执行协议与守则
    ├── scripts/
    │   ├── extract_ast.py    ← 多语言 AST 提取器
    │   ├── git_detective.py  ← Git 热点与耦合分析
    │   └── requirements.txt
    └── references/
      ├── 01-probe-protocol.md
      ├── 02-output-schema.md
      ├── 03-edge-cases.md
      ├── 04-object-framework.md
      └── 05-language-customization.md
```

如果只是把 skill 本体复制到其他 agent 工作区，复制 `skills/nexus-mapper/` 这一层即可。

---

## License

MIT
