---
title: OXA Schema Node Structure
abstract: |
  This RFC proposes the **base structural shape** for OXA schemas: a small set of common fields shared by all nodes that enables consistent traversal, transformation, and validation across tools. The goal is to define a predictable "tree contract" for OXA content while leaving room for semantic growth over time.
---

## Context

OXA aims to represent scientific content as structured, interoperable objects that can be converted, validated, rendered, and recomposed across systems. To do that reliably, tool builders need a **consistent way to traverse** document structures and identify where content and metadata live.

Many ecosystems converge on a small set of structural conventions:

- A **typed node** model, where each node has a type and optional fields
- A **tree** structure, where nodes contain children
- A **value** field for leaf nodes (text, code, math)
- A flexible **data** bucket for attaching extra information without breaking traversal

The [unifiedjs](https://unifiedjs.com/) ecosystem (via unist) has demonstrated that a small common shape dramatically reduces friction for tooling: visitors, transforms, linters, extractors, renderers, and converters can all agree on the traversal contract even while node types evolve.

## Proposal

All OXA schema nodes SHOULD conform to a shared "base node" shape with the following core fields:

- `type` (required): a capitalized string (`PascalCase`) identifying the node type
- `children`: an array of child nodes (for container nodes)
- `value`: a scalar payload for leaf/content nodes
- `data` (optional): an extensible object for non-core metadata

One of `children` or `value` is required. Additional fields may exist on specific node types (e.g., `depth` on `Heading`, `language` on `CodeBlock`), but the above fields define the common traversal and extension contract.

### 1. `data`: the extension bucket

`data` is the **escape hatch** for unknown, tool-specific, or early-stage fields. It enables experimentation and forward compatibility without requiring the core schema to anticipate every concept.

Principles:

- `data` MAY contain any JSON-serializable object.
- Consumers ignore unknown keys in `data`.
- Producers SHOULD prefer `data` for new or uncertain fields.
- As patterns stabilize, fields SHOULD be **promoted** from `data` into first-class, well-specified properties using the RFC process.

This "promote from data" pattern is how OXA evolves while remaining usable.

A key example of promotion is the distinction between **content fields** and **extension fields**:

- `children` and `value` are promoted, standardized ways to represent content.
- Everything else starts in `data` until it becomes common enough to standardize.

### 2. `children`: the traversable content tree (unist-inspired)

`children` is the primary mechanism for representing structured content as a tree.

Principles:

- Nodes with `children` represent containers (e.g., `Article`, `Section`, `Paragraph`, `List`).
- The tree SHOULD contain the narrative and actionable structure directly (headings, paragraphs, code blocks, figures), rather than hiding content in opaque blobs or the data fields.
- Visitors and transforms SHOULD be able to operate generically by walking `children`.

This approach enables consistent tooling:

- Extract all figures, code blocks, or citations
- Render to HTML/PDF
- Apply linting and validation rules
- Support recomposition and modular reuse

### 3. `value`: leaf payloads

`value` carries the "payload" for leaf nodes where a string (or other scalar) is the primary content.

Examples:

- `Text.value`: literal text
- `InlineMath.value`: math expression
- `CodeBlock.value`: code content
- `Raw.value`: format-specific raw content (if supported)

Principles:

- Nodes SHOULD NOT hide their primary content inside `data`.
- If a node’s main content is a single string payload, it SHOULD use `value`.

## Examples

### Base node patterns

```yaml
# Container node with children
- type: Paragraph
  children:
    - type: Text
      value: 'Hello '
    - type: Emphasis
      children:
        - type: Text
          value: 'world'

# Leaf node with value
- type: InlineMath
  value: "\\pi"

# Node with extension data
- type: CodeBlock
  value: |
    print("Hello")
  data:
    language: python
    executable: true
```

### Promotion example

```yaml
# Early stage: store a new field in data
- type: Image
  data:
    alt: 'A phase diagram'
    src: 'figures/phase.png'

# Later stage: promote common fields to first-class properties
- type: Image
  src: 'figures/phase.png'
  alt: 'A phase diagram'
```

## Implementation implications

This RFC is primarily about **tooling compatibility**.

Benefits:

- Tool builders can implement generic visitors and transforms by relying on `children`.
- Leaf payloads become predictable via `value`.
- The `data` bucket enables extension without schema churn.
- Converters can round-trip content more reliably across ecosystems.

Constraints / expectations:

- Content intended for traversal SHOULD live in the tree (`children` / `value`), not only in `data`.
- Schemas SHOULD define clear node-type-specific properties over time as patterns stabilize.

## Open questions

- Should `value` be limited to strings, or allow other scalars (numbers/booleans) where appropriate or objects?
- How do we represent "mixed payload" nodes where both `children` and `value` are meaningful?

## Decision

If accepted, this RFC establishes the minimal, shared node structure for OXA schemas and informs subsequent RFCs that define:

- Node identity and component-level addressing
- The common base node fields beyond `type/children/value/data` (e.g., `id`, `classes`)
- Document-level roots and metadata conventions
- Validation rules and schema versioning

## Acknowledgements

The Open Exchange Architecture was started in October 2025 at the _From Tools to Adoption Workshop_ [@10.62329/kcep6732], run by Tracy Teal and Rowan Cockett supported by The Navigation Fund [@10.71707/GN91-KA32]. There were significant contributions to this content structure and the decisions articulated in this RFC from Carlos Scheidegger, Franklin Koch, Rose Reatherford, Rowan Cockett, and Nokome Bentley.
