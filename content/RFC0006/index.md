---
title: Images and Media
abstract: |
  Defines four OXA node types for visual and media content: `Image` (a block-level still image), `InlineImage` (an inline still image), `Video` (a block-level video or animation), and `InlineVideo` (an inline video). These nodes provide a minimal, URL-based representation of media objects aligned with Markdown, HTML, JATS, and schema.org conventions.
---

This RFC introduces four node types (`Image`, `InlineImage`, `Video`, and `InlineVideo`) for representing images and video in OXA documents. Media objects are fundamental to scientific and technical writing — figures, diagrams, plots, animations, and video supplements are integral to how research is communicated and understood.

The design follows the naming convention established in RFC0003 (block-level default, `Inline` prefix for inline variants) and keeps the initial property set deliberately minimal: a URL, an encoding format, and alternative text. Future RFCs may introduce richer media containers (e.g. `Figure` with captions, labels, and numbering) that wrap these primitive media nodes.

## Motivation & Background

Every document format supports embedded media, but the abstraction level varies:

- **Markdown** uses `![alt](url)` for images — simple, inline-capable, no video support
- **HTML** separates `<img>` (images) from `<video>` (video/animation), with attributes for `src`, `alt`, `type`, `width`, `height`
- **JATS** distinguishes `<graphic>` / `<inline-graphic>` (still images) from `<media>` / `<inline-media>` (video, audio, animations), with `@mimetype`, `@mime-subtype`, and `@xlink:href`
- **schema.org** models these as [`ImageObject`](https://schema.org/ImageObject) and [`VideoObject`](https://schema.org/VideoObject), subtypes of `MediaObject`, with properties like `contentUrl`, `encodingFormat`, and `caption`

Across these systems, a consistent pattern emerges:

1. **Still images and video/animation are distinct** — they have different rendering requirements, accessibility concerns, and player semantics
2. **Block and inline placement matter** — a full-width figure image behaves differently from an inline icon or equation graphic
3. **The core data is a URL and a format** — everything else (captions, labels, sizing, positioning) belongs to the containing structure

OXA follows this pattern by defining four nodes that serve as the primitive media references, separate from the higher-level containers (like `Figure`) that will provide captions, labels, and layout semantics in a future RFC.

### Why Separate Image and Video Types

JATS uses distinct elements for still images (`<graphic>`) and time-based media (`<media>`) because they have fundamentally different rendering and accessibility requirements:

- Images are rendered immediately and completely; videos require player controls, buffering, and temporal navigation
- Images have a single visual representation; videos have duration, frame rate, and potentially audio tracks
- Screen readers describe images with alt text; video accessibility involves captions, transcripts, and audio descriptions

While a single "media" node with a MIME type could theoretically cover both, this conflates presentation semantics that tooling needs to distinguish. Separate types make the tree self-describing — a walker can find all images or all videos without inspecting MIME types.

### Why Not `Graphic` and `Media`

JATS uses `<graphic>` and `<media>` — names inherited from SGML-era publishing workflows. OXA prefers `Image` and `Video` because:

- They align with HTML (`<img>`, `<video>`), the dominant rendering target
- They align with schema.org (`ImageObject`, `VideoObject`), the dominant structured data vocabulary
- They are immediately understood by developers and authors — `Image` unambiguously means a still picture; `Graphic` could mean a vector illustration, a chart, or a design asset
- `Media` is overly broad — in JATS it covers video, audio, datasets, and arbitrary binary objects. OXA benefits from precise types — `Image`, `Video`, and in the future `Audio`, as well as computational media types (e.g. interactive visualizations, notebooks, executable figures) that have their own distinct rendering, execution, and accessibility requirements

## Proposed Node Types

### Image

A **block-level** node representing a still image (photograph, diagram, chart, illustration, etc.).

```typescript
interface Image extends Node {
  type: 'Image';
  url: string;
  alt?: string;
  encodingFormat?: string;
}
```

**Fields:**

- `url` — the URL or path to the image file. This corresponds to `contentUrl` in schema.org, `@xlink:href` in JATS, and `src` in HTML. URLs may be fully qualified (`https://cdn.example.com/images/fig1.png`) or relative to the document (`figures/scatter.png`). Relative URLs are preferred for portability — they allow the same document tree to be served through different URL resolution strategies at render time. For example, a deployment pipeline may resolve relative paths through a CDN function (e.g. a Cloudflare Worker that maps `figures/scatter.png` to a versioned object in a storage bucket), while a local preview tool resolves them against the filesystem. The document should not embed deployment-specific URL schemes; resolution is a rendering concern.
- `alt` — alternative text describing the image for accessibility (screen readers) and fallback display. Corresponds to the `alt` attribute in HTML `<img>` and `<alt-text>` in JATS. Alt text should convey the _meaning_ or _purpose_ of the image, not merely describe its visual appearance.
- `encodingFormat` — the MIME type of the image file (e.g. `"image/png"`, `"image/svg+xml"`, `"image/jpeg"`). Corresponds to `encodingFormat` in schema.org and the combination of `@mimetype` / `@mime-subtype` in JATS. When omitted, the format may be inferred from the URL file extension or HTTP response headers.

`Image` is a leaf node — it has no `children` or `value`. The image content is external, referenced by `url`. This follows the same pattern as JATS `<graphic>`, where the element is a pointer to external content, not a container for it.

In most documents, `Image` will appear inside a higher-level container such as `Figure` (to be defined in a future RFC) that provides captions, labels, and positioning. A bare `Image` node — without a containing `Figure` — represents an unlabeled image embedded directly in the document flow, analogous to a Markdown `![alt](url)` not wrapped in a figure directive, or a JATS `<graphic>` appearing directly in `<body>` or `<p>`.

### InlineImage

An **inline** node representing a still image that participates in inline text flow.

```typescript
interface InlineImage extends Node {
  type: 'InlineImage';
  url: string;
  alt?: string;
  encodingFormat?: string;
}
```

**Fields** are identical to `Image`.

`InlineImage` is used for small images that appear within prose — icons, inline equations rendered as images, small logos, or decorative glyphs. It corresponds to JATS `<inline-graphic>` and an HTML `<img>` used within a `<p>` or `<span>`.

The distinction between `Image` and `InlineImage` is structural, not visual: `Image` is a block-level node that occupies its own position in the document tree (a sibling of `Paragraph`, `Heading`, etc.), while `InlineImage` is an inline node that appears within the `children` array of a `Paragraph` or other inline container.

:::{tip .dropdown} Why Both `Image` and `InlineImage`

A single `Image` node used in both block and inline positions would be simpler, but it creates real problems for tooling and round-tripping:

1. **Tree validation becomes context-dependent.** With a single type, whether an `Image` is valid depends on _where_ it appears — is it a direct child of the document body (block) or nested inside a `Paragraph` (inline)? Separate types make validity checkable locally: an `InlineImage` inside a `Paragraph` is correct by construction; an `Image` there is a type error. This is the same reason HTML has both block and inline elements rather than making all elements context-dependent.

2. **JATS requires the distinction.** JATS uses `<graphic>` (block) and `<inline-graphic>` (inline) as separate elements. Round-tripping through JATS without losing the block/inline distinction requires that OXA preserve it structurally. A single node with a "placement hint" would need to be inferred during JATS export — fragile and lossy.

3. **Markdown parsing produces the distinction naturally.** In CommonMark, `![alt](url)` as the sole content of a paragraph creates a block-level image (the paragraph is typically unwrapped by renderers), while the same syntax mid-sentence is inline. Parsers already know which case they are in — encoding that knowledge in the node type is cheaper and more reliable than reconstructing it later.

4. **Renderers need to know without inspecting parents.** A block image may be rendered as a standalone `<figure>` or full-width `<img>` with margin handling. An inline image is rendered as an `<img>` inside a `<span>` with `vertical-align` and constrained sizing. These are different code paths. A renderer visiting an `InlineImage` knows immediately what to do; a renderer visiting a generic `Image` would need to walk up the tree to determine context.

5. **Consistent with the OXA naming convention.** RFC0003 established the `Code` / `InlineCode` pattern precisely for this reason — block and inline variants are structurally different nodes even when they share the same properties. `Image` / `InlineImage` follows the same precedent.

Markdown gets away with a single syntax because it delegates the block/inline distinction to context and renderer heuristics. OXA, as a structured schema, cannot afford that ambiguity — the tree must be self-describing.

:::

### Video

A **block-level** node representing a video or animation.

```typescript
interface Video extends Node {
  type: 'Video';
  url: string;
  alt?: string;
  encodingFormat?: string;
}
```

**Fields:**

- `url` — the URL or path to the video file. Corresponds to `contentUrl` in schema.org, `@xlink:href` in JATS `<media>`, and `src` in HTML `<video>`.
- `alt` — alternative text describing the video content for accessibility. For video, alt text should describe what the video shows or demonstrates. Richer video accessibility (captions, transcripts, audio descriptions) is out of scope for this RFC and may be addressed alongside a `Figure` container or dedicated accessibility RFC.
- `encodingFormat` — the MIME type of the video file (e.g. `"video/mp4"`, `"video/webm"`, `"video/ogg"`). Corresponds to `encodingFormat` in schema.org and `@mimetype` / `@mime-subtype` in JATS.

Like `Image`, `Video` is a leaf node — a pointer to external content. It corresponds to JATS `<media>` with a video MIME type, and schema.org `VideoObject`.

### InlineVideo

An **inline** node representing a video or animation that participates in inline text flow.

```typescript
interface InlineVideo extends Node {
  type: 'InlineVideo';
  url: string;
  alt?: string;
  encodingFormat?: string;
}
```

**Fields** are identical to `Video`.

`InlineVideo` is used for small, inline video content — animated icons, short looping demonstrations, or GIF-like animations embedded within prose. It corresponds to JATS `<inline-media>` with a video MIME type.

## Explicitly Deferred

The following concerns are intentionally out of scope for this RFC:

- **Figures** — a container node (`Figure`) that wraps media nodes with captions, labels, numbering, and positioning semantics. This is a separate structural concern and will be addressed in a dedicated RFC.
- **Width, height, and sizing** — dimensions, aspect ratios, and responsive sizing are rendering concerns that may be addressed as optional properties in a future RFC or handled by the containing `Figure`.
- **Alternative formats** — JATS supports `<alternatives>` to provide the same content in multiple formats (e.g. a PNG and an SVG of the same diagram, or AVI and MP4 of the same video). This is a valid concern but adds complexity that should be addressed alongside `Figure`.
- **Audio** — audio content (podcasts, sound clips, narration) has distinct rendering and accessibility requirements. A future `Audio` / `InlineAudio` node pair may be introduced following the same pattern.
- **Supplementary material** — JATS distinguishes "integral" media (`<graphic>`, `<media>`) from "supplementary" material (`<supplementary-material>`). This distinction is better handled at the container or document-section level.
- **Thumbnails and poster images** — `VideoObject` in schema.org supports `thumbnail`; HTML `<video>` supports `poster`. These are rendering hints that may be added as optional properties later.
- **Embedding and streaming** — `embedUrl` (schema.org) and streaming protocols are out of scope; `url` points to a file, not a player.
- **Licensing and attribution** — media objects frequently carry their own licenses (e.g. a CC-BY photograph in an otherwise CC-BY-SA document) and authorship distinct from the document's authors. JATS handles this with `<permissions>` and `<attrib>` children on `<graphic>` and `<media>`; schema.org uses `license`, `creator`, and `copyrightHolder` on `MediaObject`. A future RFC will define how licensing, attribution, and provenance metadata attach to nodes — these properties will be designed consistently across all node types that need them (images, videos, figures, code, tables, etc.), not as media-specific fields.

## Examples

### Block-Level Image

> A simple image in the document flow.

Markdown: `![A scatter plot showing correlation between variables X and Y](figures/scatter.png)`

```yaml
{
  type: 'Image',
  url: 'figures/scatter.png',
  alt: 'A scatter plot showing correlation between variables X and Y',
}
```

### Inline Image (Icon in Prose)

> Click the settings icon {icon} to configure.

```yaml
{
  type: 'Paragraph',
  children:
    [
      { type: 'Text', value: 'Click the settings icon ' },
      {
        type: 'InlineImage',
        url: 'icons/settings.svg',
        alt: 'settings icon',
        encodingFormat: 'image/svg+xml',
      },
      { type: 'Text', value: ' to configure.' },
    ],
}
```

### Block-Level Video

> A video showing the experimental procedure.

```yaml
{
  type: 'Video',
  url: 'supplementary/experiment-v1.mp4',
  alt: 'Video of the droplet formation process under varying pressure conditions',
  encodingFormat: 'video/mp4',
}
```

### Inline Video (Animated Demonstration)

> The particle follows a helical path {animation} under the applied field.

```yaml
{
  type: 'Paragraph',
  children:
    [
      { type: 'Text', value: 'The particle follows a helical path ' },
      {
        type: 'InlineVideo',
        url: 'animations/helix.webm',
        alt: 'particle tracing a helical path',
        encodingFormat: 'video/webm',
      },
      { type: 'Text', value: ' under the applied field.' },
    ],
}
```

### Image with Encoding Format

```yaml
{
  type: 'Image',
  url: 'https://example.com/diagram.svg',
  alt: 'System architecture diagram',
  encodingFormat: 'image/svg+xml',
}
```

## Mapping to Existing Formats

| OXA Node      | Markdown           | HTML                | JATS                     | schema.org    |
| ------------- | ------------------ | ------------------- | ------------------------ | ------------- |
| `Image`       | `![alt](url)`      | `<img>`             | `<graphic>`              | `ImageObject` |
| `InlineImage` | `![alt](url)` [^1] | `<img>` (in flow)   | `<inline-graphic>`       | `ImageObject` |
| `Video`       | —                  | `<video>`           | `<media>` (video)        | `VideoObject` |
| `InlineVideo` | —                  | `<video>` (in flow) | `<inline-media>` (video) | `VideoObject` |

### Property Mapping

| OXA Property     | Markdown | HTML   | JATS                          | schema.org        |
| ---------------- | -------- | ------ | ----------------------------- | ----------------- |
| `url`            | `(url)`  | `src`  | `@xlink:href`                 | `contentUrl`      |
| `alt`            | `[alt]`  | `alt`  | `<alt-text>`                  | `description`[^2] |
| `encodingFormat` | —        | `type` | `@mimetype` + `@mime-subtype` | `encodingFormat`  |

## Implications

If accepted, this RFC:

- Introduces `Image`, `InlineImage`, `Video`, and `InlineVideo` as standard OXA node types
- Establishes the minimal property set (`url`, `alt`, `encodingFormat`) for media references
- Provides a clear mapping path from Markdown, HTML, JATS, and schema.org
- Creates the primitive media nodes that a future `Figure` RFC can wrap with captions, labels, and layout semantics
- Follows the block/inline naming convention from RFC0003

## Decision

Acceptance of this RFC establishes the media vocabulary for OXA schemas, providing the building blocks for representing visual and video content in a structured, interoperable way.

## References

- **JATS `<graphic>`** — <https://jats.nlm.nih.gov/archiving/tag-library/1.3/element/graphic.html>
- **JATS `<inline-graphic>`** — <https://jats.nlm.nih.gov/archiving/tag-library/1.3/element/inline-graphic.html>
- **JATS `<media>`** — <https://jats.nlm.nih.gov/archiving/tag-library/1.3/element/media.html>
- **JATS `<inline-media>`** — <https://jats.nlm.nih.gov/archiving/tag-library/1.3/element/inline-media.html>
- **schema.org `ImageObject`** — <https://schema.org/ImageObject>
- **schema.org `VideoObject`** — <https://schema.org/VideoObject>
- **HTML `<img>`** — <https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img>
- **HTML `<video>`** — <https://developer.mozilla.org/en-US/docs/Web/HTML/Element/video>
- **CommonMark Images** — <https://spec.commonmark.org/0.31.2/#images>

[^1]: Markdown does not syntactically distinguish block and inline images — the same `![alt](url)` syntax is used in both contexts. The block vs. inline distinction is determined by the parser based on whether the image is the sole content of a paragraph.

[^2]: schema.org `ImageObject` does not have a dedicated `alt` property. The closest mapping is `description` (from `Thing`). The `caption` property on `ImageObject` serves a different purpose — it is a visible caption, not accessibility alt text.
