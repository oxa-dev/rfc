---
title: OXA RFC Process and Initial Scope
abstract: |
  This document establishes the **Request for Comment (RFC) process** for the Open Exchange Architecture (OXA) and defines the **initial goals and scope** for early OXA work. It is intentionally procedural rather than technical. Future RFCs will introduce and evolve specific schemas, containers, and linking models. The intent of this RFC is to set expectations for _how_ we work together before prescribing _what_ the architecture looks like.
---

## Motivation

OXA aims to support **interoperable, modular, and reusable scientific content** across tools, institutions, and time. Achieving that goal requires more than a technically correct specification — it requires a process that prioritizes:

- Adoption over theoretical completeness
- Iteration over perfection
- Implementations over committee-driven abstractions
- Practical convergence over ideological purity

The RFC process is designed to support _continuous design_: small, reviewable decisions that are grounded in real use, tested in software, and revised as experience accumulates.

## Goals of the RFC Process

The OXA RFC process aims to:

- **Provide clarity** on what is being proposed, why, and with what implications
- **Enable participation** without requiring deep prior context
- **Create shared understanding** across tool builders, publishers, and researchers
- **Document decisions** and their rationale in a durable, citable form
- **Support iteration** as tools, needs, and understanding evolve

RFCs are _not_ meant to represent the final truth. They are working agreements that may be revised or superseded as OXA matures.

## What an RFC Is (and Is Not)

An RFC in OXA:

- Is a written proposal for a **significant change or addition**
- Is scoped narrowly enough to be reviewed and implemented within a reasonable timeframe
- Is grounded in real or anticipated use cases
- May include open questions or known limitations

An RFC is **not**:

- A full system design
- A minor change that is low-risk or has little to no impact
- A comprehensive ontology
- A guarantee of permanence
- A substitute for implementation

## High-Level Scope of Early OXA Work

This RFC does _not_ define the technical structure of OXA. However, for the benefit of orienting early discussions, early RFCs are expected to focus on three foundational layers:

### 1. Container

OXA is intended to represent whole and complete research artifacts, not just narrative text and metadata (e.g. JATS). A container should be able to express a research output as it was produced and intended to be understood: narrative, figures, methods, code, and computational elements together, while still allowing components to link out to external datasets, repositories, or services where appropriate. Rather than treating computation, data, and code as supplementary or external by default, OXA should treat them as first-class, addressable components of the research artifact.

Examples of how a scientific artifact is packaged for exchange and preservation include:

- Bundling related files and exposing metadata
- Addressing versioning and technical access to the components (e.g. file system, cloud-buckets)
- Compatibility with existing packaging approaches (e.g. FTP, Zip, Git, MECA)

### 2. Structure

OXA will describe how content and metadata are represented in a structured form through schemas. Schema design in OXA will explicitly navigate the trade‑offs between **semantic purity** and **structural utility**. Highly semantic models improve conceptual clarity and alignment across systems, but can become brittle or impractical for real authoring, conversion, and rendering workflows. Highly structural models are easier to implement and transform, but risk losing meaning.

OXA prioritizes schemas that are _semantically meaningful enough to support interoperability and reuse_, while remaining _structurally practical for tools to implement, round‑trip, and evolve_. Where trade‑offs exist, OXA favors approaches that enable adoption, iteration, and working software over perfectly normalized representations.

Guiding preferences:

- Strict enough to be useful, flexible enough to be usable
- Small, composable pieces rather than monolithic schemas

Existing sources of inspiration include:

- JATS (conceptual scope, not structural complexity)
- schema.org (naming, reuse of established concepts)
- unified-style ASTs (tree-based document models)
- Pandoc JSON, Stencila JSON and MyST Markdown AST

### 3. Linking

**Granular identification** and linking is a core motivation for OXA: components within a document should be individually identifiable, referenceable, and reusable across systems, even as documents evolve. This enables citation, attribution, review, and reuse at the level of _parts_, not just whole articles.

Examples:

- Granular identification of document components, enabling stable references to figures, tables, equations, sections, code blocks, or other addressable parts of a document
- Infrastructure that enables hover-references to document components while preserving attribution, licensing and context
- Integration with persistent identifiers (DOIs and beyond), including relationships such as _isPartOf_, _references_, _cites_
- Reuse, attribution and licensing at the component level

## How We Will Work

The RFC process is designed to be intentionally **lightweight and action-oriented**.

Key norms:

- Validate ideas through implementations and demos, not discussion alone
- Prefer existing names, patterns, and concepts where possible
- Look outward before inventing inward (JATS, schema.org, etc.)
- Expect early drafts in the process to be incomplete
- Treat revisions as a success, not a failure

Consensus is built through _use_, not unanimity.

## RFC Process

Proposals begin as lightweight issues to test scope and motivation, graduate to draft RFCs via pull requests, and are refined through open discussion. Once an RFC is marked active, the Steering Council facilitates a time‑boxed review and approval process. Accepted RFCs move directly into implementation. See the RFC documentation for full procedural details, templates, and acceptance criteria.

## Governance

OXA is stewarded and hosted by the **Continuous Science Foundation (CSF)**, which provides organizational support and ensures the project remains aligned with its mission of advancing open, interoperable scientific communication.

### Steering Council

OXA is guided by a small **Steering Council** responsible for providing high‑level direction, safeguarding the project’s principles, and supporting timely decision‑making. Drawing inspiration from governance models used by organizations such as the Python Software Foundation and Project Jupyter, the Steering Council’s role is _directional rather than operational_.

Responsibilities of the Steering Council include:

- Setting and maintaining the overall technical and strategic direction of OXA
- Overseeing the RFC process and serving as final approvers for RFCs
- Steward the community and be responsible for the code of conduct
- Help to resolve conflicts or blocking issues when consensus cannot be reached
- Ensuring the project remains open, inclusive, and aligned with its stated goals
- Representing OXA in external collaborations and partnerships, when appropriate
- Supporting grant and funding efforts for OXA as a project, without implying organizational ownership or exclusive control

Steering Council members serve as individuals, not as formal representatives of their organizations.

The initial Steering Council consists of:

- **Tracy Teal** (openRxiv)
- **Rowan Cockett** (Continuous Science Foundation, Curvenote, Jupyter)
- **Nokome Bentley** (Stencila)

The Steering Council may expand over time as the project grows and as additional perspectives are needed.

### Core Team

OXA also will create a **Core Team** of developers who are actively involved in designing, implementing, and maintaining the technology. Core Team members are expected to:

- Contribute code, examples, documentation, and tooling to the OXA ecosystem
- Help review technical proposals and implementations
- Support the practical evolution of schemas, containers, and tooling
- Create and maintain issues and pull request practices and review and approve PRs

Core Team membership is expected to emerge through sustained contribution and demonstrated commitment to the project. Members may come from different organizations and tool ecosystems.

### Contributors and Community

OXA is an open project. Anyone may create issues, propose RFCs, or contribute code and documentation, provided they follow the project’s Code of Conduct.

The project values broad participation and encourages contributions from researchers, tool builders, publishers, and infrastructure providers. Governance structures exist to enable progress and stewardship — not to restrict contribution.

## Conclusion

This RFC defines the _social and procedural contract_ for OXA development. Lays out the scope of the project and some areas of initial focus (containers, structure, linking).

Future RFCs will:

- Introduce concrete schemas
- Define container formats
- Specify linking and identifier strategies
- Address versioning, hashing, and validation

OXA is built by doing. This RFC process exists to make that work visible, collaborative, and durable.

## Acknowledgements

The Open Exchange Architecture was started in October 2025 at the _From Tools to Adoption Workshop_ [@10.62329/kcep6732], run by Tracy Teal and Rowan Cockett supported by The Navigation Fund [@10.71707/GN91-KA32]. There were significant contributions from participants of the workshop including Adam Hyde, Agah Karakuzu, Anton Molina, Carlos Scheidegger, Carol Willing, Chris Wilkinson, Franklin Koch, Gabe Stein, Jason Priem, John Bohannon, John Kaye, Kevin-John Black, Matt Akamatsu, Michael Markie, Milton Pividori, Monica Granados, Nokome Bentley, Paul Shannon, Raj Palleti, Rose Reatherford, Rowan Cockett, Steinn Sigurdsson, Taylor Campbell, Ted Roeder, Tom Scott, Tracy Teal.
