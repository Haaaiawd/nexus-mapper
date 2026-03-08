<p align="center">
  <img src="Icon.png" alt="nexus-skills" width="96" height="96">
</p>

<h1 align="center">nexus-skills</h1>

<p align="center">
  为 AI Agent 生成可持续复用的代码仓库知识库，并提供精准的代码结构查询能力。<br>
  Build a persistent knowledge map of any codebase. Query its structure with precision. Every AI session starts smarter.
</p>

<p align="center">
  <a href="README.zh-CN.md">中文文档</a>
</p>

---

## Two Skills, One Philosophy

| Skill | What It Does | When to Use |
|-------|-------------|-------------|
| **nexus-mapper** | Analyzes the full repository and writes a persistent `.nexus-map/` knowledge base for future AI sessions | When starting on an unfamiliar repo, onboarding a team, or preparing for major architectural work |
| **nexus-query** | Queries file structure, reverse dependencies, change impact radius, and coupling hotspots directly from AST data | During active development — before interface changes, refactoring sprints, or legacy codebase navigation |

Both skills use the same underlying scripts. `nexus-query` can reuse a `.nexus-map/` produced by `nexus-mapper`, or generate its own `ast_nodes.json` on demand.

---

## nexus-mapper

nexus-mapper is a repository-mapping skill for AI agents. It analyzes a local codebase, writes a persistent `.nexus-map/` knowledge base, and gives the next session a concrete place to start instead of forcing it to rediscover architecture from scratch.

This is not a generic "summarize the repo" prompt. The skill runs a gated PROBE workflow, challenges its own first-pass assumptions, and only then writes final assets. That design matters: it reduces the usual AI failure mode of turning first impressions into fake certainty.

```
.nexus-map/
├── INDEX.md              ← Load this first. Full architectural context, under 2000 tokens.
├── arch/
│   ├── systems.md        ← Every subsystem: what it owns, exactly where it sits in the repo.
│   ├── dependencies.md   ← How components connect. Rendered as a Mermaid dependency graph.
│   └── test_coverage.md  ← Static test surface: what is tested, what is not, and where evidence is thin.
├── concepts/
│   ├── concept_model.json ← Machine-readable knowledge graph. Structured for programmatic use.
│   └── domains.md        ← The domain language this codebase speaks, in plain terms.
├── hotspots/             ← Present when git metadata is available.
│   └── git_forensics.md  ← Files that change constantly, and pairs that always change together.
└── raw/                  ← Source data: AST nodes, git statistics, filtered file tree.
```

`INDEX.md` is the entry point. It stays small on purpose so an AI can load it in full, recover global context quickly, and then drill into deeper files only when needed.

---

## nexus-query

nexus-query gives precise, instant answers to structural questions during active development — without reading the entire codebase.

```bash
# File skeleton: classes, methods, line numbers, imports
python skills/nexus-query/scripts/query_graph.py ast_nodes.json --file src/core/vision.py

# Reverse dependency: who imports this module?
python skills/nexus-query/scripts/query_graph.py ast_nodes.json --who-imports src.core.vision

# Impact radius: upstream dependencies + downstream dependents
python skills/nexus-query/scripts/query_graph.py ast_nodes.json --impact src/core/vision.py \
  --git-stats git_stats.json

# Architectural hub analysis: highest fan-in / fan-out modules
python skills/nexus-query/scripts/query_graph.py ast_nodes.json --hub-analysis

# Per-directory structural summary
python skills/nexus-query/scripts/query_graph.py ast_nodes.json --summary
```

Zero extra dependencies. Pure Python stdlib. The `ast_nodes.json` can come from an existing `.nexus-map/raw/` or from a fresh `extract_ast.py` run.

**When it matters most:**
- Before any interface change: `--who-imports` tells you exactly what breaks
- Before a sprint: `--impact --git-stats` quantifies risk and work scope
- In a legacy codebase: `--file` gives you a skeleton map without reading 3000 lines
- At architecture review: `--hub-analysis` finds the real coupling hotspots, not just the ones named `core/`

---

## Why They Are Different

- **Phase-gated**: nexus-mapper's PROFILE, REASON, OBJECT, BENCHMARK, and EMIT are not optional.
- **Honest provenance**: Both skills distinguish `implemented`, `planned`, and `inferred`. If parsing is partial, it says so.
- **Composable**: nexus-query works standalone or on top of a nexus-mapper map.
- **Optimized for future sessions**: every artifact is designed to be loaded next time, not just this time.

---

## Prerequisites

| Requirement | Check |
|-------------|-------|
| Python 3.10+ | `python --version` |
| Shell execution | Your AI client must support running terminal commands |

A git repository is recommended but not required. Without git history, hotspot analysis is skipped and the rest still runs.

**Install script dependencies:**

```bash
# nexus-mapper
pip install -r skills/nexus-mapper/scripts/requirements.txt

# nexus-query (same dependencies)
pip install -r skills/nexus-query/scripts/requirements.txt
```

---

## Install

```bash
# Full suite
npx skills add haaaiawd/nexus-skills

# Or just the skill you need
npx skills add haaaiawd/nexus-skills/skills/nexus-mapper
npx skills add haaaiawd/nexus-skills/skills/nexus-query
```

Works with Claude Code, GitHub Copilot, Cursor, Cline, and any client that reads `SKILL.md`.

---

## How To Use nexus-mapper

Point your AI at a local repository path:

```
Analyze /Users/me/projects/my-app and generate a knowledge map
```

The AI runs the protocol and writes `.nexus-map/` into the repository root. The next time you work on that codebase, start with:

```
Read .nexus-map/INDEX.md
```

For the best long-term behavior, add a short persistent instruction to your host tool's memory file such as `AGENTS.md` or `CLAUDE.md`:

```md
If .nexus-map/INDEX.md exists, read it before starting work to restore global project context.

If a task requires local structure, dependency, impact radius, or boundary validation, use query_graph.py against .nexus-map/raw/ast_nodes.json instead of guessing.

When a task changes the project's structural understanding, assess whether .nexus-map should be updated before delivery.
```

---

## Language Support

Parses 30+ languages automatically by file extension.

Python · JavaScript · JSX · TypeScript · TSX · Bash · Java · Go · Rust · C++ · C · C# · Kotlin · Ruby · Swift · Scala · PHP · Lua · Elixir · GDScript · Dart · Haskell · Clojure · SQL · Proto · Solidity · Vue · Svelte · R · Perl

Not every listed language has the same depth. Some are full structural parses, some are module-only. The output metadata tells you which is which.

### Extending language support

```bash
python skills/nexus-mapper/scripts/extract_ast.py <repo_path> \
  --add-extension .templ=templ \
  --add-query templ struct "(component_declaration name: (identifier) @class.name) @class.def"
```

---

## Repository Structure

```
nexus-skills/
├── README.md
├── README.zh-CN.md
├── Icon.png
├── evals/
└── skills/
    ├── nexus-mapper/
    │   ├── SKILL.md              ← Protocol, guardrails, output schema
    │   ├── scripts/
    │   │   ├── extract_ast.py    ← Multi-language AST extractor
    │   │   ├── query_graph.py    ← On-demand AST query tool
    │   │   ├── git_detective.py  ← Git hotspot and coupling analysis
    │   │   ├── languages.json    ← Language config
    │   │   └── requirements.txt
    │   └── references/
    │       ├── 01-probe-protocol.md
    │       ├── 02-output-schema.md
    │       ├── 03-edge-cases.md
    │       ├── 04-object-framework.md
    │       ├── 05-language-customization.md
    │       └── 06-query-guide.md
    └── nexus-query/
        ├── SKILL.md              ← Query modes, guardrails, use cases
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
