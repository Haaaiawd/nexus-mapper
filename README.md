<p align="center">
  <img src="Icon.png" alt="nexus-mapper" width="96" height="96">
</p>

<h1 align="center">nexus-mapper</h1>

<p align="center">
  Build a persistent knowledge base for any codebase.<br>
  New AI session, zero re-exploration.
</p>

<p align="center">
  <a href="README.zh-CN.md">中文文档</a>
</p>

---

## What you get

Run nexus-mapper on a repository once. It writes a `.nexus-map/` directory with everything an AI needs to understand the codebase — architecture, boundaries, hot files, dependency graph. Next time you open a chat window, the AI reads one file and already knows where everything lives.

The mapper challenges its own first-pass hypothesis before it writes final assets. It prefers a small number of evidence-backed corrections over padded issue lists or fake certainty.

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
│                           These are where bugs hide and where changes break things.
└── raw/                  ← Source data: AST nodes, git statistics, filtered file tree.
```

`INDEX.md` is the entry point. It is intentionally small — an AI can load it in full without truncation, and then navigate to deeper files as needed.

Every generated Markdown file carries a small provenance header with `verified_at` and downgrade notes. If the repository contains known-but-unsupported languages, or languages that only have module-level AST coverage, nexus-mapper must say so explicitly instead of overstating parser confidence.

If a repository needs extra language support beyond the built-in language set, an agent should first extend the analysis through command-line inputs such as `--add-extension` and `--add-query`. If the setup becomes too large or the queries are too long for one command, the agent can then switch to `--language-config <JSON_FILE>` as an explicit input.

---

## Prerequisites

| Requirement | Check |
|-------------|-------|
| Python 3.10+ | `python --version` |
| Shell execution | Your AI client must support running terminal commands |

A git repository is recommended but not required. Without git history, the `hotspots/` analysis is skipped; everything else runs normally.

**Install script dependencies before first use:**

```bash
pip install -r skills/nexus-mapper/scripts/requirements.txt
```
---

## Install

```bash
npx skills add haaaiawd/nexus-mapper
```

Works with Claude Code, GitHub Copilot, Cursor, Cline, and any client that reads `SKILL.md`.

---

## Usage

Point your AI at a local repository path:

```
Analyze /Users/me/projects/my-app and generate a knowledge map
```

The AI runs the analysis, then writes `.nexus-map/` into the repository root. The next time you — or any other AI — needs to work on that codebase, start with:

```
Read .nexus-map/INDEX.md
```

That's the full context, compressed and ready.

For the best long-term behavior, add a short persistent instruction to your host tool's memory file such as `AGENTS.md`, `CLAUDE.md`, or an equivalent file:

```md
If .nexus-map/INDEX.md exists, read it before starting work to restore project context.

When a task changes the project's structural understanding, assess whether .nexus-map should be updated before delivery. Structural understanding includes system boundaries, entrypoints, dependencies, test surface, language support, roadmap, and stage/progress facts. Pure local implementation details do not require a map update by default.

Treat .nexus-map as part of the project's memory, not as static documentation.
```

---

## On-demand queries

`scripts/query_graph.py` reads the generated `ast_nodes.json` and answers structural questions without re-parsing.

```bash
# File structure and imports
python scripts/query_graph.py .nexus-map/raw/ast_nodes.json --file src/server/handler.py

# Who imports a module?
python scripts/query_graph.py .nexus-map/raw/ast_nodes.json --who-imports src.server.handler

# Impact radius (upstream + downstream)
python scripts/query_graph.py .nexus-map/raw/ast_nodes.json --impact src/server/handler.py

# Enrich with git risk and coupling data
python scripts/query_graph.py .nexus-map/raw/ast_nodes.json --impact src/server/handler.py \
  --git-stats .nexus-map/raw/git_stats.json

# Top fan-in / fan-out hubs
python scripts/query_graph.py .nexus-map/raw/ast_nodes.json --hub-analysis

# Per-directory summary
python scripts/query_graph.py .nexus-map/raw/ast_nodes.json --summary
```

Zero extra dependencies — pure Python stdlib. The PROBE protocol uses it during REASON, OBJECT, and EMIT stages; you can also call it ad-hoc during development.

---

## Language support

Parses 17+ languages automatically by file extension.

Python · JavaScript · JSX · TypeScript · TSX · Bash · Java · Go · Rust · C++ · C · C# · Kotlin · Ruby · Swift · Scala · PHP · Lua · Elixir · GDScript · Dart · Haskell · Clojure · SQL · Proto · Solidity · Vue · Svelte · R · Perl

Not every listed language has the same depth. Some are full structural parses, some are currently module-only, and some may be explicitly requested but still unavailable if no parser can be loaded. The output metadata tells you which is which.

Unknown extensions are skipped silently. Mixed-language repositories work without any configuration.

### Extending language support

If built-in coverage is not enough, first extend the run directly from the command line:

```bash
python skills/nexus-mapper/scripts/extract_ast.py <repo_path> \
  --add-extension .templ=templ \
  --add-query templ struct "(component_declaration name: (identifier) @class.name) @class.def"
```

If the configuration is too large for a single command, pass a JSON file explicitly:

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

This keeps every language on the same contract: structural coverage if a parser and query exist, module-only if only the parser is available, configured-but-unavailable if the agent explicitly asked for a language but the environment cannot load it, unsupported if it is explicitly marked as such.

---

## Repository structure

```
nexus-mapper/
├── README.md
├── README.zh-CN.md
├── Icon.png
├── evals/                        ← Evaluation assets and test plans for iterating on the skill
└── skills/
  └── nexus-mapper/
    ├── SKILL.md              ← Execution protocol and guardrails
    ├── scripts/
    │   ├── extract_ast.py    ← Multi-language AST extractor
    │   ├── query_graph.py    ← On-demand AST query tool (file, impact, hub-analysis…)
    │   ├── git_detective.py  ← Git hotspot and coupling analysis
    │   ├── languages.json    ← Shared language config (extensions + Tree-sitter queries)
    │   └── requirements.txt
    └── references/
      ├── 01-probe-protocol.md
      ├── 02-output-schema.md
      ├── 03-edge-cases.md
      ├── 04-object-framework.md
      └── 05-language-customization.md
```

If you are copying just the skill payload into another agent workspace, copy the `skills/nexus-mapper/` directory.

---

## License

MIT

