---
name: nexus-mapper
description: Analyze a local repository and generate a persistent `.nexus-map/` knowledge base for future AI sessions. Use whenever the user asks to map a codebase, understand project architecture, build repo knowledge, create cold-start context, or assess change impact across an unfamiliar repository. Requires shell execution and local Python. Do not use for single-file questions, pure API environments, or tasks that only need a quick grep/read.
---

# nexus-mapper — AI 项目探测协议

> "你不是在写代码文档。你是在为下一个接手的 AI 建立思维基础。"

本 Skill 指导 AI Agent 使用 **PROBE 五阶段协议**，对任意本地 Git 仓库执行系统性探测，产出 `.nexus-map/` 分层知识库。

---

## ⚠️ CRITICAL — 五阶段不可跳过

> [!IMPORTANT]
> **在 PROFILE、REASON、OBJECT、BENCHMARK 完成前，不得产出最终 `.nexus-map/`。**
>
> 这不是为了形式完整，而是为了防止 AI 把第一眼假设直接写成结论。最终产物必须建立在脚本输出、仓库结构、反证挑战和回查验证之上。

❌ **禁止行为**：
- 跳过 OBJECT 直接写输出资产
- 在 BENCHMARK 完成前生成 `concept_model.json`
- PROFILE 阶段脚本失败后继续执行后续阶段

✅ **必须做到**：
- 每个阶段完成后显式确认「✅ 阶段名 完成」再进入下一阶段
- OBJECT 提出足以推翻当前假设的最少一组高价值质疑，通常为 1-3 条，绝不凑数
- `implemented` 节点的 `code_path` 必须在仓库中真实存在；`planned/inferred` 节点不得伪造 `code_path`（见守则2）

---

## 📌 何时调用 / 何时不调用

| 场景 | 调用 |
|------|:----:|
| 用户提供本地 repo 路径，希望 AI 理解其架构 | ✅ |
| 需要生成 `.nexus-map/INDEX.md` 供后续 AI 会话冷启动 | ✅ |
| 用户说「帮我分析项目」「建立项目知识库」「让 AI 了解这个仓库」 | ✅ |
| 运行环境无 shell 执行能力（纯 API 调用模式，无 `run_command` 工具） | ❌ |
| 宿主机无本地 Python 3.10+ | ❌ |
| 目标仓库无任何已知语言源文件（`.py/.ts/.java/.go/.rs/.cpp` 等均无） | ❌ |
| 用户只想查询某个特定文件/函数 → 直接用 `view_file` / `grep_search` | ❌ |

---

## ⚠️ 前提检查（缺失项要显式告知；可降级时优先降级而不是中止）

| 前提 | 检查方式 |
|------|---------|
| 目标路径存在 | `$repo_path` 可访问 |
| Python 3.10+ | `python --version` >= 3.10 |
| 脚本依赖已安装 | `python -c "import tree_sitter"` 无报错 |
| 有 shell 执行能力 | Agent 环境支持 `run_command` 工具调用 |

`git` 历史是加分项，不是硬阻塞项。没有 `.git` 或历史过少时，跳过热点分析，并在输出中明确记录这是一次降级探测。

---

## 📥 输入契约

```
repo_path: 目标仓库的本地绝对路径（必填）
```

**语言支持**：自动按文件扩展名 dispatch，优先用 bundled structural queries 提取模块/类/函数；若 grammar 可加载但当前没有结构 query，则至少保留 Module 级节点并在输出中标注 `module-only coverage`。当前已接入的常见语言包括 Python/JavaScript/TypeScript/TSX/Bash/Java/Go/Rust/C#/C/C++/Kotlin/Ruby/Swift/Scala/PHP/Lua/Elixir/GDScript/Dart/Haskell/Clojure/SQL/Proto/Solidity/Vue/Svelte/R/Perl。

**仓库本地语言扩展接口**：若目标仓库存在 `.nexus-mapper/language-overrides.json`，或显式通过 `--language-config` 提供 JSON 配置，则必须把它视为正式输入。后续 agent 可通过该文件：
- 新增/覆盖扩展名 → tree-sitter 语言名映射
- 为任意语言补充 `struct` / `imports` query
- 显式声明某些扩展名当前仍属于 `unsupported`

所有通过该接口新增的语言，必须继续遵守同一套覆盖诚实度规则：`structural` / `module-only` / `configured-but-unavailable` / `unsupported`，不得对自定义语言放宽标准。

---

## 📤 输出格式

执行完成后，目标仓库根目录下将产出：

```text
.nexus-map/
├── INDEX.md                    ← AI 冷启动主入口（< 2000 tokens）
├── arch/
│   ├── systems.md              ← 系统边界 + 代码位置
│   ├── dependencies.md         ← Mermaid 依赖图 + 时序图
│   └── test_coverage.md        ← 静态测试面：测试文件、覆盖到的核心模块、证据缺口
├── concepts/
│   ├── concept_model.json      ← Schema V1 机器可读图谱
│   └── domains.md              ← 核心领域概念说明
├── hotspots/
│   └── git_forensics.md        ← Git 热点 + 耦合对分析
└── raw/
    ├── ast_nodes.json          ← Tree-sitter 解析原始数据
    ├── git_stats.json          ← Git 热点与耦合数据
    └── file_tree.txt           ← 过滤后的文件树
```

所有生成的 Markdown 文件必须带一个简短头部，至少包含：
- `generated_by`
- `verified_at`
- `provenance`

如果 PROFILE 阶段发现已知但未支持的语言文件，`provenance` 必须明确写出哪些部分属于人工推断或降级分析。
如果 PROFILE 阶段发现 `module-only coverage`，也必须写清楚：这些语言已被计入 AST 文件覆盖，但没有类/函数级结构保证。
如果 PROFILE 阶段发现某个通过覆盖配置声明的语言仍然无法加载 parser，也必须写清楚：这是 `configured-but-unavailable`，不能伪装成已覆盖。

---

## 🔄 PROBE 五阶段协议

> [!IMPORTANT]
> **Reference 文件不是附录，而是阶段执行说明。** 进入对应阶段前先读对应 reference，
> 是为了减少漏判边界场景、误写 schema 和跳过自我校验的概率。

| 阶段 | 开始前必须读取（硬门控） | 核心动作 | 完成标志 |
|------|------------------------|---------|--------|
| **P**ROFILE | ⛔ `read_file references/01-probe-protocol.md` | 运行脚本，产出 `raw/` 核心输入 | `ast_nodes.json` 与 `file_tree.txt` 可用；git 数据按条件生成 |
| **R**EASON | ⛔ `read_file references/03-edge-cases.md`（检查是否触发边界场景） | 阅读 README/热点/文件树，识别主要系统 | 每个系统有职责描述 + 初步 `implementation_status` |
| **O**BJECT | ⛔ `read_file references/04-object-framework.md` | 按三维度提出最少一组高价值反驳点 | 每条质疑都有具体证据线索和验证计划 |
| **B**ENCHMARK | （无额外文件，使用已加载的协议） | 逐一验证异议，修正错误节点 | `implemented` 节点的 `code_path` 已验证；`planned/inferred` 节点的证据链完整 |
| **E**MIT | ⛔ `read_file references/02-output-schema.md`（校验 Schema 后才能写文件） | 原子写入全部 `.nexus-map/` 文件 | 全部文件通过 Schema 校验，并带验证日期与 provenance 头部 |

**强制阅读顺序总览**（按触发时间排列，不得颠倒或跳过）：

```
[Skill 激活时]     → 读  01-probe-protocol.md   （阶段步骤蓝图）
[REASON 前]        → 读  03-edge-cases.md        （确认是否命中边界场景）
[OBJECT 前]        → 读  04-object-framework.md  （三维度质疑模板）
[EMIT 前]          → 读  02-output-schema.md     （Schema 校验规范）
```

---

## 🛡️ 执行守则

### 守则1: OBJECT 拒绝形式主义

OBJECT 的存在意义是打破 REASON 的幸存者偏差。大量工程事实隐藏在目录命名和 git 热点背后，第一直觉几乎总是错的。

❌ **无效质疑（禁止提交）**：
```
Q1: 我对系统结构的把握还不够扎实
Q2: xxx 目录的职责暂时没有直接证据
```

▲ 问题不在于用了某几个词，而在于这类表述没有证据线索，也无法在 BENCHMARK 阶段验证。

✅ **有效质疑格式**：
```
Q1: git_stats 显示 tasks/analysis_tasks.py 变更 21 次（high risk），
    但 HYPOTHESIS 认为编排入口是 evolution/detective_loop.py。
    矛盾：若 detective_loop 是入口，为何 analysis_tasks 热度更高？
    证据线索: git_stats.json hotspots[0].path
    验证计划: view tasks/analysis_tasks.py 的 class 定义 + import 树
```

---

### 守则2: implemented 节点必须有真实 code_path

> [!IMPORTANT]
> 写入 `concept_model.json` 前，必须先区分节点是 `implemented`、`planned` 还是 `inferred`。
> 只有 `implemented` 节点允许写入 `code_path`，且必须亲手验证存在。

```bash
# BENCHMARK 阶段验证方式
ls $repo_path/src/nexus/application/weaving/   # ✅ 目录存在 → 节点有效
ls $repo_path/src/nexus/application/nonexist/  # ❌ [!ERROR] → 修正或删除此节点
```

对于 `planned` 或 `inferred` 节点，使用：

```json
{
  "implementation_status": "planned",
  "code_path": null,
  "evidence_path": "docs/architecture.md",
  "evidence_gap": "仓库中未发现 src/agents/monarch/，仅在设计文档中出现"
}
```

❌ 禁止：
- 用一个“勉强相关”的文件冒充 `code_path`
- `implementation_status` 为 `planned/inferred`，却写入伪精确目录
- `code_path: "PLANNED"` 这类把状态塞进路径字段的写法

---

### 守则3: EMIT 原子性

先全部写入 `.nexus-map/.tmp/`，全部成功后整体移动到正式目录，删除 `.tmp/`。

**目的**：中途失败不留半成品。下次执行检测到 `.tmp/` 存在 → 清理后重新生成。

✅ 幂等性规则：

| 状态 | 处理方式 |
|------|----------|
| `.nexus-map/` 不存在 | 直接继续 |
| `.nexus-map/` 存在且 `INDEX.md` 有效 | 询问用户：「是否覆盖？[y/n]」 |
| `.nexus-map/` 存在但文件不完整 | 「检测到未完成分析，将重新生成」，直接继续 |

---

### 守则4: INDEX.md 是唯一冷启动入口

`INDEX.md` 的读者是**从未见过这个仓库的 AI**。两个硬约束：
- **< 2000 tokens** — 超过就重写，不是截断
- **结论必须具体** — 不要用空泛的模糊词搪塞；证据不足时明确写出 `evidence gap` 或 `unknown`，并说明缺了什么证据

写完后执行 token 估算：行数 × 平均 30 tokens/行 = 粗估值。

---

## 🧭 不确定性表达规范

```
避免只写：待确认 · 可能是 · 疑似 · 也许 · 待定 · 暂不清楚 · 需要进一步 · 不确定
避免只写：pending · maybe · possibly · perhaps · TBD · to be confirmed
```

如果证据不足，可以这样写：
- `unknown: 未发现直接证据表明 api/ 是主入口，当前仅能确认 cli.py 被 README 引用`
- `evidence gap: 仓库没有 git 历史，因此 hotspots 部分跳过`

原则：允许诚实地写不确定，但必须解释不确定来自哪一条缺失证据，而不是把模糊词当结论。

---

### 守则5: 最小执行面与敏感信息保护

> [!IMPORTANT]
> 默认只运行本 Skill 自带脚本和必要的只读检查。不要因为“想更懂仓库”就执行目标仓库里的构建脚本、测试脚本或自定义命令。

- 默认允许：`extract_ast.py`、`git_detective.py`、目录遍历、文本搜索、只读文件查看
- 默认禁止：执行目标仓库的 `npm install`、`pnpm dev`、`python main.py`、`docker compose up` 等命令，除非用户明确要求
- 遇到 `.env`、密钥文件、凭据配置时：只记录其存在和用途，不抄出具体值

---

### 守则6: 降级与人工推断必须显式可见

> [!IMPORTANT]
> 如果 AST 覆盖不完整，或者某部分依赖图/系统边界来自人工阅读而非脚本产出，必须在最终文件中显式标注 provenance。

- `dependencies.md` 中凡是非 AST 直接支持的依赖关系，必须标注 `inferred from file tree/manual inspection`
- `domains.md`、`systems.md`、`INDEX.md` 如果涉及未支持语言区域，必须显式说明 `unsupported language downgrade`
- 若写入进度快照、Sprint 状态、路线图，必须附 `verified_at`，避免过期信息伪装成当前事实

---

## 🛠️ 脚本工具链

```bash
# 设置 SKILL_DIR（根据实际安装路径）
# 场景 A: 作为 .agent/skills 安装
SKILL_DIR=".agent/skills/nexus-mapper"
# 场景 B: 独立 repo（开发/调试时）
SKILL_DIR="/path/to/nexus-mapper"

# PROFILE 阶段调用
python $SKILL_DIR/scripts/extract_ast.py <repo_path> [--max-nodes 500] \
  > <repo_path>/.nexus-map/raw/ast_nodes.json

# 如仓库提供自定义语言扩展配置，可显式传入
python $SKILL_DIR/scripts/extract_ast.py <repo_path> [--max-nodes 500] \
  [--language-config <repo_path>/.nexus-mapper/language-overrides.json] \
  > <repo_path>/.nexus-map/raw/ast_nodes.json

python $SKILL_DIR/scripts/git_detective.py <repo_path> --days 90 \
  > <repo_path>/.nexus-map/raw/git_stats.json
```

**依赖安装（首次使用）**：
```bash
pip install -r $SKILL_DIR/scripts/requirements.txt
```

---

## ✅ 质量自检（EMIT 前必须全部通过）

- [ ] 五个阶段均已完成，每阶段有显式「✅ 完成」标记
- [ ] OBJECT 的质疑数量没有凑数；每条都带证据线索和可执行验证计划
- [ ] `implemented` 节点的 `code_path` 已亲手验证存在；`planned/inferred` 节点使用了 `implementation_status + evidence_path + evidence_gap`
- [ ] `responsibility` 字段：具体、可验证；证据不足时明确说明缺口
- [ ] `INDEX.md` 全文 < 2000 tokens，结论具体不过度装确定（守则4）
- [ ] 若发现未支持语言文件，最终 Markdown 头部和相关章节已显式标注降级与人工推断范围
- [ ] `arch/test_coverage.md` 已生成，并明确这是静态测试面而不是运行时覆盖率
