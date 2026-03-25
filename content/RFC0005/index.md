---
title: Citations
abstract: |
  Defines three OXA node types for scholarly citations: `Cite` (an inline reference to a bibliographic entry), `CiteGroup` (a container that groups multiple Cite nodes with shared display semantics), and `Reference` (a block-level bibliographic record).
---

This RFC introduces three node types (`Cite`, `CiteGroup`, and `Reference`) for representing citations and bibliographic references in OXA documents. Citations are fundamental to scientific and technical writing and require careful structural treatment; they are not simple cross-references and can carry rich metadata about _how_ and _why_ a source is being cited.

The design draws on established practice in LaTeX/natbib, Pandoc, JATS, CSL, schema.org, and CiTO, aiming to capture the full expressiveness of scholarly citation while keeping the node shapes consistent with OXA conventions established in RFC0002 and RFC0003.

## Motivation & Background

Citations serve multiple purposes in technical documents:

- **Referencing evidence** — pointing the reader to a source
- **Attribution** — crediting prior work
- **Contextualization** — qualifying how a source relates to the current argument (e.g. "see", "cf.", "extending")
- **Navigation** — linking to specific locations within a referenced work

Existing systems handle these concerns differently:

- **LaTeX/natbib** distinguishes narrative (`\citet`) and parenthetical (`\citep`) styles, supports author expansion (`*`), and allows prefix/suffix text and locators
- **Pandoc** uses `[@key]` syntax with prefix, suffix, and locator support; bare `@key` produces narrative citations
- **JATS** separates the inline reference (`<xref ref-type="bibr">`) from the bibliographic record (`<element-citation>` inside `<ref>`)
- **CSL** defines a comprehensive vocabulary for locator types (chapter, page, figure, etc.) and citation formatting

A single `Cite` node is insufficient — citations often appear in groups with shared punctuation and display semantics (e.g. `(Jones, 2020; Smith, 2021)`). The grouping also matters for UIs that expand grouped citations into interactive lists.

## Proposed Node Types

### Cite

An **inline** node representing a single citation to a bibliographic entry.

```typescript
interface Cite extends Parent {
  type: 'Cite';
  xref: string;
  children?: [Inline];
  prefix?: [Inline];
  suffix?: [Inline];
  display?: 'author' | 'date' | 'full';
  locator?: string;
  url?: string;
  intent?: string;
}
```

**Fields:**

- `xref` — reference to the `identifier` of a `Reference` node (e.g. `"jones2022"`). A `Reference` node with a matching `identifier` MUST be present in the containing document.
- `prefix` — inline content preceding the citation within its group (e.g. `[{ type: 'Text', value: 'see ' }]`)
- `suffix` — inline content following the citation within its group (e.g. `[{ type: 'Text', value: ' and references therein' }]`)
- `display` — controls what the citation renders from the referenced `Reference` node[^natbib-support]:
  - `'author'` — abbreviated author only, e.g. "Jones et al." (equivalent to natbib `\citeauthor`, CSL `author-only`)
  - `'date'` — date only, e.g. "1990" (equivalent to natbib `\citeyear`, CSL `suppress-author`)
  - `'full'` — display the full bibliographic reference inline, as it would appear in a reference list; useful for first-mention expansions or in-body reference rendering without requiring the reader to navigate to the bibliography
  - If omitted, the default citation rendering applies — `author-date` (e.g. "Jones et al., 1990" or "Jones et al. (2022)" depending on narrative/parenthetical context), or numeric (e.g. "[1]") depending on the citation style of the renderer.

  The `'author'` and `'date'` values map directly to CSL's `author-only` and `suppress-author` flags respectively. `'full'` is an OXA extension beyond what CSL supports per-citation.

- `locator` — a human-readable print-style locator within the referenced work (e.g. `"chap. 2"`, `"pp. 12–15"`, `"fig. 3"`); follows CSL locator conventions and is typically rendered after the citation with a comma separator
- `url` — an optional deep link to a specific location in the referenced work (e.g. `"https://doi.org/10.1234/jones2022#fig3"`); the web-native counterpart to `locator`
- `intent` — the citation intent using CiTO ([Citation Typing Ontology](https://sparontologies.github.io/cito/current/cito.html)) vocabulary (e.g. `"cites"`, `"extends"`, `"usesMethodIn"`, `"agreesWith"`, `"disputes"`); enables machine-readable citation semantics.
- `children` — optional inline content that **overrides** the dynamically rendered citation text. When present, the `Cite` node renders its children directly instead of generating display text from the referenced `Reference` node. **Children should be avoided where possible** — prefer letting the renderer resolve display text dynamically from the `xref`, `display`, `prefix`, `suffix`, and `locator` fields. This allows citations to respond to style changes (e.g. switching from author-year to numeric). Use `children` only when the citation requires fully custom display text that cannot be derived from the reference data (e.g. `"My PhD Thesis"` or a non-standard alias).

Using inline arrays for `children`, `prefix`, and `suffix` (rather than plain strings) allows rich formatting — for example, italicized titles or emphasized author names within citation text.

`Cite` is an inline node and can appear directly within a `Paragraph` or other inline container. A standalone `Cite` node (not wrapped in a `CiteGroup`) represents a **narrative** citation — one that is grammatically part of the sentence (equivalent to `\citet` in natbib or a bare `@key` in Pandoc).

Note that `Cite` uses `xref` to point to a `Reference` node's `identifier`. The `identifier` field itself is reserved as a base property available on all OXA nodes — any node in the tree can be identified and referenced. The `xref` field is the citing side of that relationship. The referenced node MUST be a `Reference` and MUST exist in the same document — `xref` is an internal document reference, not an external lookup.

#### Why Both `locator` and `url`

Traditional citation systems use locators (e.g. `"chap. 2"`, `"pp. 12–15"`, `"fig. 3"`) to reference specific parts of a work. These are print-era concepts tied to physical pagination and CSL locale formatting, but they remain essential for round-tripping with LaTeX, Pandoc, and CSL-based tooling — and they are what readers expect to see in rendered output, even online.

The `url` field adds a web-native counterpart: a direct link to the specific section, figure, or element within a referenced document. This enables programmatic navigation and richer interactive experiences.

Having both fields bridges print and web conventions. A citation can render `"(Jones, 2022, fig. 3)"` for print while simultaneously linking to `#fig3` in the referenced document. When only one is present, renderers use what is available — `locator` for display text, `url` for linking. When both are present, the locator provides the human-readable label and the URL provides the navigation target.

#### Why `intent` and CiTO

Most citation systems treat all citations as equivalent — a reference is a reference. But citations carry meaning: one paper may _extend_ another, _dispute_ it, or _use its methodology_. CiTO (Citation Typing Ontology) provides a controlled vocabulary for these relationships, enabling:

- Machine-readable citation graphs with typed edges
- Richer metadata for indexing and discovery
- Better understanding of how knowledge builds on prior work

The `intent` field is optional, it is metadata about the _relationship_ between the citing and cited work.

### CiteGroup

An **inline** container that groups multiple `Cite` nodes with shared display semantics.

```typescript
interface CiteGroup extends Parent {
  type: 'CiteGroup';
  kind: 'narrative' | 'parenthetical';
  children: [Cite];
}
```

**Fields:**

- `kind` — the citation display style:
  - `'narrative'` — authors appear as part of the sentence, with the year in parentheses (equivalent to `\citet` in natbib)
  - `'parenthetical'` — the entire citation appears in parentheses (equivalent to `\citep` in natbib)
- `children` — one or more `Cite` nodes

`CiteGroup` is never directly authored in markup — it is the structural result of parsing grouped citation syntax (e.g. `[@key1; @key2]` in Pandoc or `` \citep{key1,key2}` `` in LaTeX). The group holds the shared context (parenthetical vs. narrative) while each child `Cite` retains its own prefix, suffix, and identifier.

This separation is important for UIs: a parenthetical group like `(Jones, 2020; Smith, 2021)` can be rendered as a single parenthesized string _or_ expanded into an interactive list showing each reference with its metadata (such as a hover reference). The display of these can also be shortened by the renderer, for example, `(Jones et al., 1990, 1991)` or `(Jones et al., 1990a,b)`, at the discretion of the rendering engine.

#### Why `kind` Over Boolean Flags

Using `kind: 'narrative' | 'parenthetical'` rather than `parenthetical: boolean` allows for future extension. Additional citation styles may be introduced (e.g. `'numeric'`, `'note'`) without breaking the existing schema. The nomenclature follows established citation-studies terminology.

:::{note}
The `CiteGroup` pattern of grouping inline references with shared display semantics may be generalized in a future RFC to cover other cross-reference types — for example, grouped figure references like "(Figures 1a, 2)" or combined figure-and-table references like "(Figure 1; Table 3)". The current scope is limited to bibliographic citations.
:::

### Reference

A **block-level** node representing a bibliographic record.

```typescript
interface Reference extends Parent {
  type: 'Reference';
  children?: [Inline];
  csl: CSL.Item;
}
```

**Fields:**

- `identifier` — the unique key used by `Cite` nodes via `xref` to reference this entry (e.g. `"jones2022"`). The node's `identifier` MUST match the `citation-key` field in the `csl` object.
- `children` — optional inline content for the rendered display of this reference (e.g. `[{ type: 'Text', value: '1' }]` for numeric styles, or formatted author-year text); if omitted, renderers generate display text from `csl`
- `csl` — a single CSL-JSON item object conforming to the [CSL-JSON schema](https://resource.citationstyles.org/schema/v1.0/input/json/csl-data.json). This is a well-established, widely-supported standard for representing bibliographic data, used by Zotero, Mendeley, Pandoc, Typst, CrossRef, and most citation-processing tools.

#### CSL-JSON

The `csl` field contains a single [CSL-JSON](https://citeproc-js.readthedocs.io/en/latest/csl-json/markup.html) item — not an array. The schema defines 47 item types (e.g. `"article-journal"`, `"book"`, `"chapter"`, `"dataset"`, `"software"`) and provides structured fields for:

- **Identifiers** — `id` (required), `citation-key`, `DOI`, `ISBN`, `ISSN`, `PMID`, `URL`
- **Name variables** — `author`, `editor`, `translator`, etc., each an array of `{ family, given, literal, ... }` objects
- **Date variables** — `issued`, `accessed`, `submitted`, etc., each supporting EDTF strings or structured `date-parts`
- **Publication details** — `title`, `container-title`, `publisher`, `volume`, `issue`, `page`, `edition`, etc.
- **Additional metadata** — `abstract`, `language`, `keyword`, `note`, `custom`

By adopting CSL-JSON directly rather than defining a custom bibliographic schema, OXA benefits from:

- **Interoperability** — tools that already consume or produce CSL-JSON (Zotero, Pandoc, citeproc-js, Citation.js) work without transformation
- **Completeness** — the schema covers the full range of publication types and metadata fields across disciplines
- **Rendering** — CSL processors can use the `csl` object directly to format references in any of the thousands of available CSL styles

The node's `identifier` and the CSL item's `citation-key` MUST be kept in sync; they are the shared key that connects `Cite` nodes (via `xref`) to their bibliographic data.

`Reference` nodes are typically collected in a reference list at the end of a document, analogous to the `<ref-list>` in JATS or a bibliography section in LaTeX. This RFC does not describe where these are collected or placed in a document similar to JATS `<back>` section; that may be introduced in a future RFC.

## Relationship to Cross-References

OXA nodes can carry an `identifier`, and any node that points to another node uses `xref` to do so. `Cite` uses `xref` just like any other referencing node would — the mechanism is intended to be shared, but citations carry additional semantics that justify a distinct node type:

- Display varies by citation style (author-year, numeric, narrative vs. parenthetical)
- Citations participate in bibliographies and reference lists
- Citations can carry intent, prefix/suffix, and deep links
- Citation groups have punctuation and display rules that differ from simple reference lists

A future RFC on cross-references may define a general-purpose node (e.g. `CrossReference`) that also uses `xref` for internal document links to figures, tables, and equations. The `xref` field is the shared primitive; the node type determines the semantics.

## Examples

### Narrative Citation (Standalone Cite)

> Jones et al. (2022) demonstrated that...

```yaml
# Preferred: no children — display text resolved dynamically from the Reference
{
  type: 'Paragraph',
  children: [{ type: 'Cite', xref: 'jones2022' }, { type: 'Text', value: ' demonstrated that...' }],
}
```

### Author-Only and Date-Only Citations

> As Jones et al. showed in 2022...

```yaml
{
  type: 'Paragraph',
  children:
    [
      { type: 'Text', value: 'As ' },
      { type: 'Cite', xref: 'jones2022', display: 'author' },
      { type: 'Text', value: ' showed in ' },
      { type: 'Cite', xref: 'jones2022', display: 'date' },
      { type: 'Text', value: '...' },
    ],
}
```

### Full Inline Reference

> The framework was introduced in: Jones, A. & Chen, B. (2022). A Framework for Open Science. _Journal of Open Research_, 15(3), 112–130.

```yaml
{
  type: 'Paragraph',
  children:
    [
      { type: 'Text', value: 'The framework was introduced in: ' },
      { type: 'Cite', xref: 'jones2022', display: 'full' },
    ],
}
```

### Parenthetical Citation (CiteGroup)

> ... as shown in prior work (see Jones, 2022; Smith, 2021).

```yaml
{
  type: 'Paragraph',
  children:
    [
      { type: 'Text', value: '... as shown in prior work ' },
      {
        type: 'CiteGroup',
        kind: 'parenthetical',
        children:
          [
            { type: 'Cite', xref: 'jones2022', prefix: [{ type: 'Text', value: 'see ' }] },
            { type: 'Cite', xref: 'smith2021' },
          ],
      },
      { type: 'Text', value: '.' },
    ],
}
```

### Citation with Locator, URL, and Intent

> We extend the framework proposed by Jones et al. (2022, fig. 3).

```yaml
{
  type: 'Paragraph',
  children:
    [
      { type: 'Text', value: 'We extend the framework proposed by ' },
      {
        type: 'Cite',
        xref: 'jones2022',
        locator: 'fig. 3',
        url: 'https://doi.org/10.1234/jones2022#fig3',
        intent: 'extends',
      },
      { type: 'Text', value: '.' },
    ],
}
```

### Reference Entry

```yaml
{
  type: 'Reference',
  identifier: 'jones2022',
  csl:
    {
      type: 'article-journal',
      id: 'jones2022',
      citation-key: 'jones2022',
      title: 'A Framework for Open Science',
      author: [{ given: 'Alice', family: 'Jones' }, { given: 'Bob', family: 'Chen' }],
      issued: { date-parts: [[2022]] },
      container-title: 'Journal of Open Research',
      volume: '15',
      issue: '3',
      page: '112-130',
      DOI: '10.1234/jones2022',
    },
}
```

## Mapping to Existing Formats

| OXA Node    | LaTeX/natbib      | Pandoc        | JATS                           | schema.org                          |
| ----------- | ----------------- | ------------- | ------------------------------ | ----------------------------------- |
| `Cite`      | `\cite`, `\citet` | `@key`        | `<xref ref-type="bibr">`       | —                                   |
| `CiteGroup` | `\citep{a,b}`     | `[@a; @b]`    | (multiple `<xref>`)            | —                                   |
| `Reference` | BibTeX entry      | CSL-JSON item | `<ref>` + `<element-citation>` | `CreativeWork` / `ScholarlyArticle` |

### Natbib Command Coverage

The following table maps every natbib citation command to OXA node representations:

| natbib                        | Output                                   | OXA representation                                        |
| ----------------------------- | ---------------------------------------- | --------------------------------------------------------- |
| `\citet{jon90}`               | Jones et al. (1990)                      | Standalone `Cite`                                         |
| `\citet[chap.~2]{jon90}`      | Jones et al. (1990, chap. 2)             | `Cite` + `locator`                                        |
| `\citep{jon90}`               | (Jones et al., 1990)                     | `CiteGroup` parenthetical                                 |
| `\citep[chap.~2]{jon90}`      | (Jones et al., 1990, chap. 2)            | `CiteGroup` + `locator`                                   |
| `\citep[see][]{jon90}`        | (see Jones et al., 1990)                 | `CiteGroup` + `prefix`                                    |
| `\citep[see][chap.~2]{jon90}` | (see Jones et al., 1990, chap. 2)        | `CiteGroup` + `prefix` + `locator`                        |
| `\citet*{jon90}`              | Jones, Baker, and Williams (1990)        | `Cite` with `children` override, or CSL style config      |
| `\citep*{jon90}`              | (Jones, Baker, and Williams, 1990)       | `CiteGroup` with `children` override, or CSL style config |
| `\citealt{jon90}`             | Jones et al. 1990                        | Standalone `Cite` (default author-date display)           |
| `\citealt*{jon90}`            | Jones, Baker, and Williams 1990          | `Cite` with `children` override, or CSL style config      |
| `\citealp{jon90}`             | Jones et al., 1990                       | Standalone `Cite` (default author-date display)           |
| `\citealp*{jon90}`            | Jones, Baker, and Williams, 1990         | `Cite` with `children` override, or CSL style config      |
| `\citenum{jon90}`             | 11                                       | Style concern (numeric rendering)                         |
| `\citetext{...}`              | (...)                                    | Out of scope                                              |
| `\citeauthor{jon90}`          | Jones et al.                             | `display: 'author'`                                       |
| `\citeauthor*{jon90}`         | Jones, Baker, and Williams               | `Cite` with `children` override, or CSL style config      |
| `\citeyear{jon90}`            | 1990                                     | `display: 'date'`                                         |
| `\citeyearpar{jon90}`         | (1990)                                   | `CiteGroup` parenthetical + `display: 'date'`             |
| `\citet{jon90,jam91}`         | Jones et al. (1990); James et al. (1991) | `CiteGroup` narrative, two `Cite`                         |
| `\citep{jon90,jam91}`         | (Jones et al., 1990; James et al. 1991)  | `CiteGroup` parenthetical, two `Cite`                     |
| `\citep{jon90,jon91}`         | (Jones et al., 1990, 1991)               | Two `Cite` — year-collapsing is a render concern          |
| `\citep{jon90a,jon90b}`       | (Jones et al., 1990a,b)                  | Two `Cite` — suffix-collapsing is a render concern        |

## Implications

If accepted, this RFC:

- Introduces `Cite`, `CiteGroup`, and `Reference` as standard OXA node types
- Introduces `xref` as a node property, that should be shared with other cross-references and internal document linking
- Enables structured, machine-readable citations with typed intent
- Supports both print-era citation conventions and web-native deep linking
- Embeds CSL-JSON directly for bibliographic data, enabling immediate interoperability with existing citation tools
- Provides a clear mapping path from LaTeX, Pandoc, JATS, and CSL-JSON
- Establishes a foundation for citation-aware tooling, rendering, and search

## Decision

Acceptance of this RFC establishes the citation and bibliography vocabulary for OXA schemas, enabling implementations to represent the full range of scholarly citation practice in a structured, interoperable way.

## References

- **CSL-JSON Schema** — <https://resource.citationstyles.org/schema/v1.0/input/json/csl-data.json>
- **Natbib Reference Sheet** — <https://gking.harvard.edu/files/natnotes2.pdf>
- **Pandoc Citation Syntax** — <https://pandoc.org/MANUAL.html#citation-syntax>
- **JATS `<element-citation>`** — <https://jats.nlm.nih.gov/archiving/tag-library/1.3/element/element-citation.html>
- **rehype-citation parse-citation.js** — <https://github.com/timlrx/rehype-citation/blob/main/src/parse-citation.js#L139>
- **CiTO (Citation Typing Ontology)** — <https://sparontologies.github.io/cito/current/cito.html>

[^natbib-support]: Expanded author lists (e.g. "Jones, Baker, and Williams" instead of "Jones et al.") are intentionally not modeled as a `display` value. Per-citation author expansion adds complexity to the schema for a relatively rare need. Instead, use `children` to override the display text when full author names are required for a specific citation, or configure the CSL style's `et-al-min` / `et-al-use-first` settings to control author abbreviation thresholds at render time.
