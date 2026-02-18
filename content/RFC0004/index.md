---
title: Executable Code Nodes
abstract: |
  Defines OXA node types for executable code at both block and inline levels, enabling computational documents with embedded, runnable code that produces outputs.
---

This RFC defines **executable code node types** for OXA documents: block-level (`CodeCell`) and inline-level (`CodeExpr`), along with their properties and semantics.

Executable code is fundamental to computational documents, notebooks, and literate programming. Unlike static code blocks (addressed in RFC0003 as `Code` and `InlineCode`), executable code nodes carry execution semantics: they can be run by a kernel or runtime, produce outputs, and track execution state.

Across existing systems (e.g. Jupyter notebooks, MyST Markdown, Quarto, and Stencila) there is broad convergence on the semantics of these constructs, though implementations vary in naming, and property structure.

# Out of Scope

This RFC focuses on establishing type names and core **source properties** for executable code nodes. The following are explicitly out of scope and deferred to future RFCs:

- **Execution-derived properties**: Properties that are populated by an execution engine rather than authored directly, including:
  - `outputs` / `output`: Results from executing code
  - `errors`: Errors encountered during execution
  - `executionCount`: The number of times the node has been executed
  - `executionDuration`: The duration of the last execution

- **Labeling and caption properties**: Properties for cross-referencing and describing outputs, including:
  - `label`: Short label for cross-referencing
  - `caption`: Descriptive caption

  These properties are shared with other captionable node types (e.g., `Table`, `Figure`) and will be defined consistently in RFCs addressing those types.

- **Kernel and environment specification**: How to specify the execution environment

- **Execution caching and dependency tracking**: Mechanisms for reactive execution

These deferred properties are either implementation-dependent or shared across multiple node types. By establishing the core source properties first, this RFC provides a stable foundation that future RFCs can extend.

# Proposed Executable Code Node Types

## Block-Level: `CodeCell`

A `CodeCell` represents an executable block of code that can produce outputs, similar to a cell in a Jupyter notebook.

## Inline-Level: `CodeExpr`

A `CodeExpr` represents an executable expression embedded within prose, evaluating to a single value that appears inline with surrounding text.

# Naming Rationale

The names `CodeCell` and `CodeExpr` are chosen to:

1. **Distinguish from static code**: `Code` and `InlineCode` (RFC0003) represent non-executable, display-only code. The distinct names prevent confusion.

2. **`CodeCell` aligns with ecosystem convergence**: The term "cell" has become the dominant terminology for executable code blocks across computational document systems. Jupyter notebooks established `code_cell` as the standard, MyST adopted `code-cell` for its directive, and Quarto's internal model uses cell-based terminology. While R Markdown historically used "chunk", its successor Quarto moved toward cell-based naming, reflecting broader ecosystem convergence.

3. **`CodeExpr` is concise and reflects purpose**: The abbreviated "Expr" clearly conveys that the node evaluates an expression to produce a single inline value. This naming is consistent with Quarto's inline executable syntax (`{lang} expr`), where "expr" denotes an evaluated expression. The shorter form also balances well with `CodeCell`, keeping both type names compact.

4. **`language` favors brevity over disambiguation**: The property `language` is used rather than the more explicit `programmingLanguage` (as used by schema.org and Stencila). While `language` could theoretically be confused with the natural language of prose content, the context of executable code nodes makes the meaning unambiguous. The shorter form improves readability and aligns with common usage in Jupyter (`kernelspec.language`) and Quarto (fence info strings).

Alternative names considered:

- `ExecutableCode` / `InlineExecutableCode`: verbose
- `CodeChunk`: historical R Markdown terminology, and used by Stencila, but superseded by cell-based naming in newer systems (Quarto, MyST)
- `CodeBlock`: conflicts with common static code terminology

# Node Definitions

## `CodeCell`

```json
{
  "type": "object",
  "required": ["type", "code"],
  "properties": {
    "type": {
      "const": "CodeCell"
    },
    "code": {
      "type": "string",
      "description": "The source code to execute"
    },
    "language": {
      "type": "string",
      "description": "Programming language identifier (e.g., python, r, javascript)"
    },
    "isEchoed": {
      "type": "boolean",
      "description": "Whether the source code is displayed to readers",
      "default": false
    },
    "isHidden": {
      "type": "boolean",
      "description": "Whether outputs are hidden from readers",
      "default": false
    }
  }
}
```

**Properties:**

| Property   | Type      | Description                                                         |
| ---------- | --------- | ------------------------------------------------------------------- |
| `code`     | `string`  | The source code to execute (required)                               |
| `language` | `string`  | Programming language identifier (e.g., `python`, `r`, `javascript`) |
| `isEchoed` | `boolean` | Whether the source code is displayed to readers (default: `true`)   |
| `isHidden` | `boolean` | Whether outputs are hidden from readers (default: `false`)          |

**Example:**

```yaml
type: CodeCell
language: python
code: |
  import matplotlib.pyplot as plt
  plt.plot([1, 2, 3], [1, 4, 9])
  plt.title("Square numbers")
  plt.show()
isEchoed: true
```

## `CodeExpr`

```json
{
  "type": "object",
  "required": ["type", "code"],
  "properties": {
    "type": {
      "const": "CodeExpr"
    },
    "code": {
      "type": "string",
      "description": "The expression to evaluate"
    },
    "language": {
      "type": "string",
      "description": "Programming language identifier (e.g., python, r, javascript)"
    }
  }
}
```

**Properties:**

| Property   | Type     | Description                                                         |
| ---------- | -------- | ------------------------------------------------------------------- |
| `code`     | `string` | The expression to evaluate (required)                               |
| `language` | `string` | Programming language identifier (e.g., `python`, `r`, `javascript`) |

**Example:**

```yaml
type: Paragraph
children:
  - type: Text
    value: "The dataset contains "
  - type: CodeExpr
    language: python
    code: len(df)
  - type: Text
    value: " records."
```

# Crosswalk Tables

The following tables map OXA node types and properties to their equivalents across existing systems.

## Block-Level Executable Code

| Aspect          | OXA                  | Jupyter                                    | MyST                            | Quarto                   | Stencila              |
| --------------- | -------------------- | ------------------------------------------ | ------------------------------- | ------------------------ | --------------------- |
| **Type name**   | `CodeCell`           | `code_cell`                                | `code-cell` directive           | code block with `{lang}` | `CodeChunk`           |
| **Code**        | `code`               | `source`                                   | directive body                  | fenced content           | `code`                |
| **Language**    | `language`           | (per notebook `kernelspec`)                | directive argument              | fence info string        | `programmingLanguage` |
| **Show code**   | `isEchoed`           | `metadata.jupyter.source_hidden` (inverse) | `:tags: [hide-input]` (inverse) | `echo: true/false`       | `isEchoed`            |
| **Show output** | `isHidden` (inverse) | `metadata.jupyter.outputs_hidden`          | `:tags: [hide-output]`          | `output: true/false`     | `isHidden` (inverse)  |

## Inline Executable Code

| Aspect           | OXA        | Jupyter | MyST                   | Quarto           | Stencila              |
| ---------------- | ---------- | ------- | ---------------------- | ---------------- | --------------------- |
| **Type name**    | `CodeExpr` | -       | `{eval}` role          | `{lang} expr`    | `CodeExpression`      |
| **Code content** | `code`     | -       | role content           | backtick content | `code`                |
| **Language**     | `language` | -       | (from notebook kernel) | fence info       | `programmingLanguage` |

# Implications

If accepted, this RFC:

- Establishes the type names and core source properties for executable code in OXA documents
- Enables interoperability with Jupyter notebooks, MyST, Quarto, and Stencila
- Complements RFC0003's static code nodes with executable counterparts
- Provides a foundation for future RFCs addressing execution-derived properties

Future RFCs may address:

- Execution-derived properties (`outputs`, `errors`, `executionStatus`, etc.)
- Labeling and caption properties (shared with `Table`, `Figure`, and other captionable types)
- Environment and kernel specification
- Execution caching and dependency tracking

# Decision

Acceptance of this RFC establishes `CodeCell` and `CodeExpr` as the standard OXA node types for executable code, enabling computational documents within the OXA ecosystem.
