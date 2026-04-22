---
title: Modular Metadata
abstract: |
  Proposes a reserved `metadata` property on every OXA node that carries structured information about the node's content — title, authors, affiliations, funding, licenses, identifiers, and other descriptive data. Metadata is modular, referenceable, and composable: it propagates from parent to child, can be overridden at any level of the tree, and supports cross-references between metadata entries. This RFC establishes the principles and structural conventions for metadata; the specific field definitions are deferred to future RFCs.
---

This RFC introduces a `metadata` property available on every OXA content node. The property provides a structured place for descriptive information — authorship, licensing, identifiers, titles, affiliations, funding, and similar concerns — that applies to the node and, by default, to all of its descendants.

The design is motivated by modular scientific publishing, where individual components of a document (a figure panel, an embedded image, a chapter) may have distinct authorship, licensing, or provenance from the containing document. Rather than requiring all metadata to live at the document root, OXA treats metadata as a contextual property that flows down the tree and can be narrowed or replaced at any node.

This RFC lays out the principles of the approach. It does not define the specific metadata fields (e.g. the shape of an author object or the license vocabulary) — those will be specified in subsequent RFCs that can draw on this structural foundation.

## Motivation

Scientific documents are not monolithic. A single article may contain:

- **Figures** contributed by a collaborator who is not a document author
- **Panels within a figure** created by a subset of the figure's authors
- **Embedded images** sourced from external works with different licenses
- **Chapters** in a collection, each written by different author groups
- **Datasets** with their own DOIs, funders, and data-availability statements

Current document formats handle this poorly. JATS places all metadata in a single `<front>` section at the document level; there is no standard mechanism for per-component authorship or licensing. LaTeX has no native metadata model at all. Pandoc's YAML frontmatter is document-level only.

OXA needs metadata that is **modular** — it must be possible to describe _any_ node in the tree with its own metadata context, independent of the document root.

## Proposal

### The `metadata` Property

Every OXA node MAY include a `metadata` property. This is a reserved key, distinct from `data` (the general extension bucket defined in RFC0002). Where `data` is an unstructured escape hatch for tool-specific or experimental fields, `metadata` is a structured, well-defined space for descriptive information about the node's content.

```typescript
interface Node {
  type: string;
  children?: Node[];
  value?: string;
  data?: Record<string, unknown>;
  metadata?: Metadata;
}
```

The `Metadata` type will be defined in detail by subsequent RFCs. For the purposes of this RFC, it is an object that may contain fields such as:

```typescript
interface Metadata {
  title?: InlineContent[];
  subtitle?: InlineContent[];
  authors?: (AuthorData | MetadataReference)[];
  license?: LicenseData | MetadataReference;
  identifiers?: Record<string, string>;
  affiliations?: (AffiliationData | MetadataReference)[];
  funding?: (FundingData | MetadataReference)[];
  // ... additional fields defined by future RFCs
}
```

### Metadata Context and Propagation

The top-level node in a tree establishes the **metadata context** for all of its descendants. Children inherit the parent's metadata unless they provide their own `metadata` property, in which case the child's metadata becomes the new context for that subtree.

This is analogous to how the JATS `<front>` section describes the document — but generalized to any node in the tree.

```yaml
type: Document
metadata:
  title:
    - type: Text
      value: 'Seismic Observations of the 2024 Noto Peninsula Earthquake'
  authors:
    - identifier: rowan
      name: Rowan Cockett
      orcid: 0000-0002-7859-8394
    - identifier: tracy
      name: Tracy K. Teal
      orcid: 0000-0002-9180-9598
  license:
    id: CC-BY-4.0
children:
  - type: Heading
    level: 1
    children:
      - type: Text
        value: 'Introduction'
  - type: Paragraph
    children:
      - type: Text
        value: 'This document demonstrates modular metadata...'
  - type: Image
    src: 'https://example.com/seismic-map.png'
    metadata:
      authors:
        - xref: '@rowan'
          roles:
            - Visualization
      license:
        id: CC-BY-4.0
```

In this example:

- The `Document` node establishes authorship and licensing for the entire tree.
- The `Image` node overrides the metadata context: it credits a specific author with a specific role, and declares its own license. The `Heading` and `Paragraph` nodes inherit the document-level metadata.
- The image's author entry uses a cross-reference (`xref: '@rowan'`) to point back to the full author definition in the document metadata, rather than duplicating the data.

### Principles

#### 1. Modular

Metadata can be attached to any node. A figure, a panel within a figure, a chapter, an embedded dataset — any node that needs its own descriptive context can carry `metadata`. This supports modular science, where components are authored, licensed, and identified independently.

**Example:** A figure composed of four panels, where panel (b) was created by a different research group:

```yaml
type: Figure
metadata:
  authors:
    - xref: '@rowan'
    - xref: '@tracy'
children:
  - type: Image
    src: 'panel-a.png'
  - type: Image
    src: 'panel-b.png'
    metadata:
      authors:
        - identifier: external-collab
          name: J. Martinez
          orcid: 0000-0001-2345-6789
          affiliations:
            - name: Universidad Nacional
      license:
        id: CC-BY-SA-4.0
  - type: Image
    src: 'panel-c.png'
  - type: Image
    src: 'panel-d.png'
```

Panels (a), (c), and (d) inherit the figure-level metadata. Panel (b) has its own authorship and a different license.

#### 2. Referenceable

Metadata entries can be **defined once and referenced elsewhere** in the document. Authors, affiliations, funders, and other entities are given identifiers within the metadata and can be referenced using cross-reference (`xref`) syntax from other metadata sections or from inline content.

**Example:** An author defined in the document metadata and referenced in an acknowledgements section:

```yaml
type: Document
metadata:
  authors:
    - identifier: rowan
      name: Rowan Cockett
      orcid: 0000-0002-7859-8394
children:
  # ... document content ...
  - type: Paragraph
    children:
      - type: Text
        value: 'In this manuscript, '
      - type: CrossReference
        xref: '@rowan'
        kind: Person
        children:
          - type: Text
            value: 'R. C.'
      - type: Text
        value: ' conceived the study and wrote the initial draft.'
```

The `@` prefix distinguishes metadata references from content references (e.g. `#fig1` for a figure, `@rowan` for a metadata reference, like authors). The exact cross-reference mechanics will be defined in a future RFC on cross-references.

#### 3. Composable

Metadata references can be **composed** — a new node can reference existing metadata entries while adding or overriding specific fields. This avoids duplication and keeps the source of truth in one place.

**Example:** An image that references an existing author but adds a role specific to this context:

```yaml
type: Image
src: 'visualization.png'
metadata:
  authors:
    - xref: '@rowan'
      roles:
        - Visualization
        - Software
```

The image does not redefine Rowan's name, ORCID, or affiliations — it references the canonical entry and layers on context-specific roles. This composition pattern means that updating the author's ORCID in the document metadata automatically propagates to all references.

### Metadata Identifiers

All identifiable entries within metadata (authors, affiliations, funders, grants, venues, etc.) carry an `identifier` field. These identifiers:

- MUST be unique across all metadata in the document (i.e. you cannot have an author and an affiliation with the same identifier)
- Need NOT be unique across the content of the document — a section with identifier `csf` and a metadata affiliation with identifier `csf` occupy different namespaces (content and metadata respectively)
- Are referenced using the `@` prefix in cross-references (e.g. `@rowan`, `@csf`)

The `@` prefix is a convention proposed by this RFC to distinguish metadata references from content references. A future cross-reference RFC will formalize the full syntax, including how to disambiguate when content and metadata identifiers overlap.

### Title and Subtitle

Titles and subtitles are included in `metadata` as inline content arrays (`InlineContent[]`), allowing rich formatting (e.g. math, emphasis, superscripts in titles).

A node's metadata title and its content are distinct concepts. A figure may have a caption (in its `children`) that differs from the metadata title inherited from the image's original source. Both can coexist:

- The **metadata title** describes the node for indexing, citation, and metadata propagation purposes
- The **content title** (e.g. a caption, heading) is what appears in the rendered document

This distinction is useful when embedding components from external sources. An image sourced from a different publication carries its original metadata title, but the containing figure may present it with a different caption in context.

Titles in metadata do need to be traversable by tree algorithms for transformations (e.g. resolving cross-references within a title). Because `metadata.title` is an array of inline nodes — the same types that appear in `children` — existing tree walkers can be extended to traverse metadata content with minimal additional complexity.

### Unknown and Experimental Metadata

Metadata that does not fit a defined field SHOULD be placed in the node's `data` property (RFC0002), not in `metadata`. The `metadata` property is reserved for structured, well-defined fields specified by RFCs. This keeps `metadata` predictable for tooling while preserving `data` as the extension point for experimental or tool-specific information.

## Relationship to JATS

The document-level `metadata` is analogous to the JATS `<front>` element, which contains `<article-meta>` with title, authors, affiliations, funding, licenses, and identifiers. Future RFCs that define the specific metadata fields SHOULD aim for mostly lossless mapping to and from JATS `<front>`, with the understanding that some JATS elements may be omitted where open alternatives exist (e.g. preferring ROR over Ringold for organization identifiers, or ORCID over proprietary author IDs)[^jats-lossless].

[^jats-lossless]: There are elements of JATS that we may choose to not include in this metadata, for example, support for non-open identifiers that have open alternatives (e.g. Ringold).

The key difference from JATS is that OXA metadata is not restricted to the document root. Any node can carry `metadata`, enabling per-component attribution and licensing that JATS does not natively support.

| Concern           | JATS                                                | OXA                                |
| ----------------- | --------------------------------------------------- | ---------------------------------- |
| Metadata scope    | Document-level only (`<front>`)                     | Any node in the tree               |
| Author per-figure | Not natively supported                              | `metadata.authors` on any node     |
| License per-asset | `<license>` in `<permissions>`, document-level only | `metadata.license` on any node     |
| Identifiers       | `<article-id>`, fixed vocabulary                    | `metadata.identifiers`, extensible |
| Extension         | Custom XML elements or processing instructions      | `data` property (RFC0002)          |

## Alternatives Considered

### Backmatter Node

We considered a `Backmatter` block-level node that would live exactly once as the last child of any tree and contain contributor definitions, affiliations, funding information, and supporting sections (data availability, acknowledgements, etc.).

This approach was rejected because:

- It conflates **metadata** (descriptive information about the content) with **content** (sections like acknowledgements that are part of the narrative). Acknowledgements are content that happen to appear at the end; they belong in the tree as regular nodes, not in a special container.
- It does not support per-component metadata. A `Backmatter` on the document root cannot express that a specific image has different authorship.
- It raises awkward questions about where to define new metadata entries that are first introduced mid-document (e.g. an author who only contributed one figure). With the `metadata` property approach, the author can be defined where they are first relevant — either on the document node (if they should be discoverable at the top level) or on the specific component.

### Metadata on CrossReference Nodes

An alternative for mid-document author definitions would be to allow `CrossReference` nodes to carry `metadata` that defines new entries inline:

```yaml
type: CrossReference
metadata:
  authors:
    - identifier: someone
      name: 'A. Helpful Person'
xref: '@someone'
children:
  - type: Text
    value: 'Person'
```

While this works mechanically, it adds complexity to cross-reference semantics — a `CrossReference` would sometimes _define_ metadata rather than just _reference_ it. The simpler approach is to define all metadata entries on the appropriate container node (typically the document root) and reference them from content. This keeps the definition site predictable and avoids special-casing `CrossReference` for metadata propagation.

## Open Questions

- **Identifier scoping:** Should metadata identifiers be required to start with a special prefix (e.g. `person:rowan`, `org:csf`), or is the `@` reference prefix sufficient to prevent conflicts with content identifiers? Starting without type prefixes keeps the syntax lighter, but may need revisiting if collision patterns emerge.
- **Propagation semantics:** When a child node provides `metadata`, does it _replace_ the parent context entirely, or _merge_ with it? Full replacement is simpler and more predictable; merging risks ambiguity about which fields are inherited vs. overridden. This RFC proposes full replacement as the default — a child with `metadata` establishes a new, complete context for its subtree.
- **Metadata field definitions:** The specific shapes of author, affiliation, funding, license, and identifier objects are intentionally deferred. Future RFCs should define these, drawing on JATS, schema.org, DataCite, and CRediT for established vocabularies.
- **Tree traversal of metadata content:** Metadata fields like `title` contain inline node arrays. Should tree-walking algorithms traverse `metadata` by default, or require explicit opt-in? Traversing by default ensures transformations (e.g. resolving cross-references in titles) work transparently, but increases the surface area that algorithms must handle.
- **Inline metadata references:** Can metadata entries be referenced freely from inline content (e.g. an author callout in the acknowledgements)? This RFC proposes yes — a `CrossReference` node with `xref: '@rowan'` can appear anywhere in the document. The rendering of such references (e.g. expanding to the author's full name, linking to their ORCID) is a renderer concern.

## Implications

If accepted, this RFC:

- Reserves `metadata` as a property on all OXA nodes, alongside `type`, `children`, `value`, and `data`
- Establishes metadata propagation as a core tree semantic: parent metadata applies to children unless overridden
- Introduces the `@` prefix convention for metadata cross-references, to be formalized in a future cross-reference RFC
- Provides the structural foundation for future RFCs to define specific metadata fields (authors, licenses, identifiers, etc.)
- Enables per-component attribution and licensing, supporting modular scientific publishing
- Maintains a clear separation between structured metadata (`metadata`) and unstructured extensions (`data`)

## Decision

Acceptance of this RFC establishes the `metadata` property as a reserved, structured extension point on every OXA node, enabling modular, referenceable, and composable metadata throughout the document tree. Subsequent RFCs will define the specific metadata vocabularies (authorship, licensing, funding, identifiers) within this framework.
