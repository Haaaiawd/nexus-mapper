# 仓库本地语言覆盖配置

> 本文件不是阶段门控文件，但当用户希望“补某个暂未适配的语言”时，后续 agent 应优先参考本文件，而不是直接改核心脚本。

---

## 目标

让目标仓库可以用 **repo 本地配置** 扩展语言支持，而不是把每次新增语言都做成核心脚本补丁。

配置文件位置默认是：

```text
.nexus-mapper/language-overrides.json
```

也可以在运行 `extract_ast.py` 时显式传入：

```bash
python extract_ast.py <repo_path> --language-config <repo_path>/.nexus-mapper/language-overrides.json
```

---

## Schema

```json
{
  "extensions": {
    ".templ": "templ"
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

### `extensions`

- 键：文件扩展名，可写成 `.templ` 或 `templ`
- 值：`tree-sitter-language-pack` 可识别的语言名，如 `templ`

### `queries`

- 键：语言名
- 值：对象，允许两个字段：
  - `struct`
  - `imports`

捕获名必须继续遵守现有约定：

- 类：`@class.def` / `@class.name`
- 函数：`@func.def` / `@func.name`
- 导入：`@mod`

### `unsupported_extensions`

- 用于显式声明“仓库里确实有这种扩展名，但当前仍不支持”
- 这比静默跳过更好，因为后续输出会把它写进 metadata 和 provenance

---

## 覆盖诚实度规则

repo 本地覆盖配置不会改变覆盖分层标准。所有语言一视同仁：

1. `structural coverage`
   条件：parser 可加载，且存在 `struct` query

2. `module-only coverage`
   条件：parser 可加载，但没有 `struct` query

3. `configured-but-unavailable`
   条件：仓库配置要求支持该语言，但当前环境无法加载 parser

4. `unsupported`
   条件：仓库或内建规则明确标记为未支持

禁止把 `configured-but-unavailable` 写成 `module-only`。
禁止把 `unsupported` 伪装成“仓库里没出现”。

---

## 推荐工作流

当后续 agent 需要补一个新语言时，优先顺序应当是：

1. 先确认 `tree-sitter-language-pack` 是否已有该语言 grammar
2. 若已有 grammar：
   - 先在 `.nexus-mapper/language-overrides.json` 增加扩展名映射
   - 如果只需要 Module 级覆盖，到此为止
   - 如果要类/函数结构，再补 `queries`
3. 若没有 grammar：
   - 先把扩展名写入 `unsupported_extensions`
   - 在输出中显式保留 evidence gap
   - 不要直接伪造查询结果

---

## 设计原则

- repo 本地配置优先于内建映射
- 自定义 query 是正式输入，不是旁路 hack
- 所有新增语言都必须走同一套 metadata 和 provenance 规则
- 如果只是为了补一个仓库的本地语言，不要优先改核心脚本