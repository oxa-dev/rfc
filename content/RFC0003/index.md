---
title: Core Node Types and Naming Conventions
---

This RFC defines the **initial set of non-controversial OXA node types** and establishes a **naming convention** for distinguishing block-level and inline-level content.

The goal of this RFC is not to be exhaustive, but to lock down the _boring, obvious, and widely shared_ parts of document structure so that tooling can rely on a stable core while more complex or contested structures are addressed in later RFCs.Context

Across authoring and publishing systems — including word processors, Markdown dialects, HTML, Pandoc, MyST, Quarto, Stencila, JATS — there is strong convergence around a small set of structural elements. These elements form the backbone of narrative documents and are consistently represented as a traversable tree of blocks and inline content.

By defining these nodes early, OXA establishes a predictable baseline that:

- Enables generic traversal and transformation
- Supports round-tripping across ecosystems
- Minimizes early bikeshedding
- Creates a clear extension path for future node types

## Proposed Core Node Types

### Block-Level Nodes

These nodes represent structural units that occupy their own place in the document tree.

- **Heading**
- **Paragraph**
- **Code**
- **ThematicBreak**

These block nodes are sufficient to represent the majority of narrative scientific documents in a structured, tool-agnostic way.

### Inline-Level Nodes

These nodes represent content that appears _within_ block nodes and participates in inline flow.

- **Text**
- **Emphasis**
- **Strong**
- **Superscript**
- **Subscript**
- **InlineCode**

Inline nodes are always expected to appear within the `children` array of a block or inline container (See RFC0002).

## Explicitly Out of Scope

The following structures are intentionally excluded from this RFC because they introduce additional complexity best handled later:

- **Quotes / BlockQuotes** — deferred due to attribution, provenance, and citation considerations
- **Tables** — complex structure, dedicated RFC
- **Lists** — ordered, unordered, and definition lists introduce additional hierarchy and semantics
- **Figures, images, and media** — handled in later media-focused RFCs
- **Citations and bibliographies** — require identifier and relationship models

Excluding these nodes at this stage is a deliberate choice to keep the initial core small, stable, and easy to implement.

## Naming Conventions and Design Trade-offs

A key decision in this RFC is the **naming pattern used to distinguish block-level and inline-level nodes**.

Several conventions were considered:

- `BlockCode` / `InlineCode`
- `CodeBlock` / `CodeInline`
- `Code` / `InlineCode`

Each option has trade-offs.

### Considerations

1. **Consistency for future extensions**
   - New node types should not require revisiting naming decisions later.
   - The naming scheme should gracefully handle unforeseen additions.

2. **Avoiding semantic overreach**
   - Some existing nodes (e.g. `BlockQuote`) blur the line between semantic meaning and layout, complicating symmetric naming.

### Chosen Pattern: `Code` / `InlineCode`

This RFC proposes the `Code` / `InlineCode` naming pattern as the default convention:

> When there is not a clear default (e.g. `Text`) and both block and inline variants exist, inline-only nodes are explicitly prefixed with `Inline`.

Benefits of this approach:

- Defaults new node types to their default (either block-level or inline), matching common document structure
- Avoids forcing symmetric pairs where they are awkward or misleading (e.g. `Quote` / `InlineQuote`)
- Leaves room for future inline-only extensions without renaming existing block types

For example:

- `Code` (block-level)
- `InlineCode` (inline-level)

This pattern also enables future extensions such as:

- `List` (block-level)
- `InlineList` (inline-level, e.g. for structured a/b/c points within a sentence)

Importantly, this avoids a breaking change if an inline variant is introduced later — we do not need to redefine an existing block type as a "block list" retroactively.

The trade-off is that block and inline variants are not grouped adjacently in file listings (e.g. `Code` vs `InlineCode`). This is considered an acceptable and minor cost.

## Node Definitions and Examples

This section provides more concrete, implementation-oriented examples of the proposed core nodes. The examples below are are meant to clarify intent and guide implementers.

### Abstract Node Shapes

#### `Literal`

```typescript
interface Literal {
  value: string;
}
```

**Literal** represents a leaf node whose primary content is a scalar value. In OXA, `Literal` nodes carry their content via the `value` field rather than `children`.

Examples of `Literal` nodes include `Text` and `InlineCode`.

#### `Parent`

```typescript
interface Parent {
  children: [Node];
}
```

**Parent** represents a node that contains other nodes. Parent nodes define the traversable document tree and are the primary mechanism by which structure and ordering are expressed.

### Block-Level Nodes

#### `Heading`

```typescript
interface Heading extends Parent {
  type: 'Heading';
  level: number;
  children: [Inline];
}
```

**Heading** represents the heading for a section of content.

- `level` indicates the heading depth (e.g. 1–6)
- Content is expressed via inline children

Example:

```yaml
{ type: 'Heading', level: 1, children: [{ type: 'Text', value: 'Introduction' }] }
```

#### `Paragraph`

```typescript
interface Paragraph extends Parent {
  type: 'Paragraph';
  children: [Inline];
}
```

**Paragraph** represents a unit of prose. It contains inline content such as text, emphasis, and inline code.

Example:

```yaml
{
  type: 'Paragraph',
  children:
    [
      { type: 'Text', value: 'Run ' },
      { type: 'InlineCode', value: 'make build' },
      { type: 'Text', value: ' to compile the project.' },
    ],
}
```

#### `Code`

```typescript
interface Code extends Literal {
  type: 'Code';
  language: string;
}
```

**Code** represents a block of preformatted text, typically source code.

- `language` (optional) indicates the programming language
- content is stored in the `value` field

Example:

```yaml
{ type: 'Code', language: 'python', value: 'print("Hello, world")' }
```

This node is conceptually paired with `InlineCode`, but is block-level.

#### `ThematicBreak`

```typescript
interface ThematicBreak {
  type: 'ThematicBreak';
}
```

**ThematicBreak** represents a thematic or structural division between sections of content.

Example:

```yaml
{ type: 'ThematicBreak' }
```

### Inline-Level Nodes

#### `Text`

```typescript
interface Text extends Literal {
  type: 'Text';
}
```

**Text** represents unformatted character data.

Example:

```yaml
{ type: 'Text', value: 'Hello world' }
```

#### `Emphasis`

```typescript
interface Emphasis extends Parent {
  type: 'Emphasis';
  children: [Inline];
}
```

**Emphasis** represents stressed emphasis of its contents.

Example:

```yaml
{ type: 'Emphasis', children: [{ type: 'Text', value: 'important' }] }
```

#### `Strong`

```typescript
interface Strong extends Parent {
  type: 'Strong';
  children: [Inline];
}
```

**Strong** represents strong importance or emphasis.

#### `InlineCode`

```typescript
interface InlineCode extends Literal {
  type: 'InlineCode';
}
```

**InlineCode** represents short fragments of code appearing within prose.

- `language` (optional) indicates the programming language
- content is stored in the `value` field

Example:

```yaml
{ type: 'InlineCode', language: 'bash', value: 'ls -la' }
```

## Implications

If accepted, this RFC:

- Establishes a minimal, interoperable set of node types for early OXA implementations
- Provides a clear naming convention for block vs inline nodes
- Sets a precedent for how future node types should be introduced

Subsequent RFCs can build on this foundation to introduce:

- Lists and tables
- Quotes and attribution-aware structures
- Media and figures
- Citations and identifiers

## Decision

Acceptance of this RFC establishes the initial core vocabulary and naming conventions for OXA schemas, enabling early implementations to converge while leaving space for future evolution.
