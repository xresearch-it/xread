# xread · spec

`@xresearch/xread` · identity + metadata in the [repo README](../README.md)

> **Specs = Docs.** This is xread's own spec, carried in xread's own repo. It is **self-contained**: an external contributor sees this package and the public packages it composes over (`xsubstrate`, `xcontract`, `xkit`, `xresearch-std`) — nothing else. The package is described as a standalone box with a typed interface; system wiring is never its concern.

---

## Responsibility

xread is **the reader + citation-browser surface**: it builds a paper-reference graph from metadata and projects a workspace subgraph to a sourced draft. It is **pure logic, environment-agnostic** (no Cloudflare/Workers/Pages imports). As a **surface** (a declared composition, `render ∘ citationGraph`) it composes over the **base** (`xsubstrate` / `xcontract`) plus the shared render component (`xkit`, the `render` of its algebra) — via published `@xresearch/<name>` entry points only, never an engine peer's internals (C-18). That is what keeps it independently extractable.

The surface IS a composition, so its own responsibility is stated as one:

```
xread = render ∘ citationGraph(xsubstrate)
```

Read literally:

- **`xsubstrate`** is the base — the append-only claim-graph + event-log it reads from. xread holds no store of its own; it is a **functor over the substrate**.
- **`citationGraph`** is the projection xread *adds*: the paper-reference network (paper nodes, `cited_works`, `cites` edges, resolution states, OA enrichment), built from **metadata alone** — no reasoning, no claim-extraction in the path.
- **`render`** is the surface: the shared render-component (from `xkit`) showing a subgraph selection, plus the one-direction **compose** projection (`graph → Markdown draft`) that reads what the graph holds and writes nothing back.

Two faculties, one package:

| Faculty | Direction | Reasoning? |
|---|---|---|
| **Citation graph** (§ 3.1) | builds graph from parsed metadata | none |
| **Compose / export** (§ 3.2) | projects graph → draft, **read-only over the graph** | none (deterministic templates) |

**No reasoning lives here.** xread renders structure that already exists in the store and projects it out; it never calls a retrieval or reasoning operation. The citation-graph build and the compose/export projection are its **own**.

---

## Interface

xread reads its shapes from the store and `xcontract`; it defines no new tables. What crosses its boundary:

**In**
- `paper.parsed` — the trigger to build a paper node + reference list from parser output (§ 3.1.3).
- The substrate's `papers` rows, `cited_works` rows, and the polymorphic `edges` table — **read** via shapes conforming to `xcontract`.
- A workspace subgraph (working-set refs + workspace-authored edges + the exporter's gaps/corroborations) — the compose input (§ 3.2).
- External enrichment responses (OpenAlex / Semantic Scholar / Unpaywall / parser) — Zod-validated at the seam (C-10).

**Out**
- The rendered citation-graph subgraph selection (the left-column map, § UI).
- `POST /api/cited-works/:id/resolve` — a human resolution action, writes a `cited_work.resolved` event.
- `POST /api/workspaces/:id/export` — renders the compose working set to a **Markdown draft** (or `format: 'bib'` to the reference list); always async, result via `GET /api/exports/:id`.
- `POST /api/citations/render` — the one citation renderer (inline `[@key]`, reference list, `.bib`, copy-as-cited).
- Export events appended to the store (the export records itself; the substrate is never mutated).

xread **owns no content** and **writes nothing back** to the substrate beyond append-only export/resolution events.

---

## Behaviour

### 3.1 · The citation / reference graph

A navigable **paper-reference network** built from paper metadata + the parsed reference list — **no claim-extraction required**. It is the left column of the surface (the library / citation map).

#### 3.1.1 · Goals

- A navigable **paper-reference network** built from metadata alone.
- Cited works are first-class clickable entities — **including ones not yet ingested**, enriched from publisher-neutral open infrastructure.
- An OA-available cited work is **one click from ingested** — the citation map is also the intake funnel.
- In-text `[N]` citations resolve to their reference entity.

#### 3.1.2 · Graph shape

The graph is three node/edge kinds over the substrate. xread does **not** define new tables; it reads the substrate's `cited_works` rows and the polymorphic `edges` table (the field-level schema is `xsubstrate`'s; xread's contract is what it *reads*, conforming to `xcontract`).

| Element | What it is | Why this shape, not another |
|---|---|---|
| **Paper node** | An ingested paper (has a PDF, has pipeline state). | The substrate's `papers` row. |
| **`cited_works` row** | A work *cited by* a paper — its own kind, **not** an overloaded `papers` row (no PDF, no pipeline state) and **not** an `entity_cluster` (a work, not a concept). | A cited work may never be ingested; it still needs an id and a node. Overloading either neighbour corrupts that neighbour's invariants. |
| **`cites` edge** | `paper → cited_work`, a **declared dependency** in the polymorphic `edges` table: the `dependency` edge-family, `dependency_subtype = cites`. | A citation *is* a dependency declaration; it rides the existing edge family rather than minting a parallel relation kind (one canonical way per concern). |

**External nodes are real nodes.** A not-yet-ingested cited work is a first-class `cited_works` row with a stable id — the map reflects the researcher's *real* reference network, not just what is loaded.

#### 3.1.3 · Build from metadata

On `paper.parsed`, xread parses the **reference list** (parser output) and creates, per paper:

1. the **paper node**,
2. one **`cited_works` row per reference**,
3. one **`cites` edge** per reference.

This runs on **metadata alone**, so the citation graph is built **without any dependency on claim-extraction having run.**

#### 3.1.4 · Resolution — conservative, with honest states

Reference resolution is a hard problem wearing an easy costume, so the surface admits it. Every `cited_works` row renders **exactly one** of three states, stored as `resolution_state` — a discriminated union, never a set of optional flags that "shouldn't both be set":

| `resolution_state` | Meaning | How it is reached |
|---|---|---|
| **`resolved`** | Matched to a clean entity. | DOI / arXiv match, **or** normalized-title match above a conservative confidence. Sets `resolved_paper_id`. |
| **`ambiguous`** | Competing matches, shown as such. | Multiple candidates in `resolution_candidates_json`; the pick is a **human action** — `POST /api/cited-works/:id/resolve` — writing a `cited_work.resolved` event. |
| **`unresolved`** | Raw citation string, still a **first-class clickable node**. | No confident match. The node is never dropped for being unresolved. |

**Dedup on create**, in order: by DOI, else arXiv id, else normalized title.

**Resolution never rewires an edge.** A resolved work gets `resolved_paper_id` set; **edges keep pointing at the `cited_works` row and resolve through that indirection.** When an external work is later ingested, the same indirection resolves its node to the now-ingested paper — without breaking, moving, or re-pointing a single edge. The indirection is the mechanism, not a hope.

**Ship order inside the surface:** the **raw reference list renders first** (zero resolution risk, immediately useful); resolution + enrichment layer on top. The resolution-quality floor gates the *resolution layer's default-on*, never the column itself — the raw list ships regardless.

#### 3.1.5 · Navigate

The left column renders the reference network: every paper a node, every cited work clickable. Selecting one **switches the right render-component to that entity's view** — one link model, one shared render component (§ UI). In-text `[N]` citations resolve to their reference entity, so a citation is a link, not dead text.

#### 3.1.6 · Enrich + one-click OA ingest

A `cited_works` row is **enriched asynchronously** from publisher-neutral open infrastructure:

- **OpenAlex** (CC0 metadata: authors, year, venue-neutral identifiers, OA status) — primary;
- **Semantic Scholar** — secondary;
- **Unpaywall** — a legal OA full-text URL.

Enrichment writes `oa_url` + `enriched_at` (and `retraction_status`). A cited work with an `oa_url` gets a **one-click "ingest this work"** affordance routing the OA link through the normal source intake — **the citation map is the intake funnel.** The graph stays **metadata-first**: enrichment makes external nodes *rich* nodes **without owning any content**. Enrichment failure leaves a plain, still-clickable node — **the graph never waits on the enrichment layer.**

#### 3.1.7 · Layout + ranking — deterministic

- **Layout is deterministic** — seeded by node id, so the **same library renders the same layout across refreshes** (C-16: same inputs, same bytes out).
- When a neighbourhood exceeds the visible limit, **truncate by a neighbour-ranking composite** (validated-confidence, edge-weight, recency, centrality — runtime-configurable), with an **honest truncation UX**: *"top N by connection strength · filter to see all M."* A cap is named in the UI, never silently applied (C-22: nothing unbounded).
- **Global node ranking is personalized PageRank** — not a bespoke named algorithm.

#### 3.1.8 · Citation-graph acceptance

- A parsed paper produces its node + a cited-work entity per reference + `cites` edges — with **no** dependency on extraction having run.
- A `[N]` in-text citation resolves to the correct reference entity.
- The same library renders the **same** deterministic layout across refreshes.
- A cited work later ingested resolves its external node to the ingested paper **without breaking edges**.
- An enriched cited work with an `oa_url` ingests in **one click** through the normal intake; enrichment failure leaves a plain, clickable node.

#### 3.1.9 · Citation-graph non-goals

- Richer bibliometric overlays (co-citation clustering, author networks, impact metrics) — later. **No prestige / venue prior:** authority is earned by validation, not journal name.
- Full-text fetch of external cited works — they stay metadata entities until ingested.
- Paper recommendations / curated-graph discovery (the playlist model) — later; rides this graph plus the ranking substrate once real usage signals exist.

### 3.2 · Compose (export) — the projection out

A researcher composes on the graph and needs a draft to carry to their paper. The draft is a **projection of the workspace subgraph, not an AI-written paper** — every exported sentence traces to a claim the author accepted, so authorship and provenance stay intact. Compose **reads the graph and writes nothing back to it.**

#### 3.2.1 · Goals

- A workspace subgraph → a **Markdown draft** that preserves provenance.
- **Projection, not generation** — no AI prose enters at the boundary of what the author validated.
- **Non-destructive** — an export is a snapshot; re-export shows what changed, and the graph stays bit-identical.

#### 3.2.2 · One direction: `graph → draft`

Compose reads the working-set refs + the edges authored in the workspace + the exporter's gaps/corroborations, and emits Markdown. **It never writes to the graph** — so it cannot corrupt the substance, and re-export is always safe (the export itself is recorded as events; the graph stays bit-identical). This is the single most important property of the surface: **read-only over the substrate.**

#### 3.2.3 · Export — `POST /api/workspaces/:id/export`

Renders the **compose working set** to Markdown, or with `format: 'bib'` to the working set's reference list.

- **The working set:** the `in_composition` refs (the add-to-cart); the **whole accepted set** when the cart is empty. **Canonical-resolved and lifecycle-honest** — deliberately-kept superseded refs export **with their tombstone badge** (a pinned citation is content, not an accident); **invalidated never.**
- **Always async.** The export job runs on a background `export-queue` — **never in the request path**, because a whole-workspace render is unbounded work (C-22, C-23: bounded, observable background work). `GET /api/exports/:id` fetches the result; large bodies via object storage. **The render is a deferred user mutation** (the export carries its initiator): the consumer revalidates the initiator is `users.status='active'` before finalizing; an initiator anonymize-revoked mid-render terminates the export `status='cancelled'` (reason `actor_revoked`, the `user_scoped_revalidates` consumer posture) and lands no artifact — the async arm of the actor-status guard, mirroring extraction persist.
- **Resolved at render time, not request time.** The cart can mutate between the `202` and the queue run. The consumer records the resolved scope in `exports.params_json` (bounded); the exact rendered ref/claim ids land in the **provenance sidecar** (the audit record, off the request payload). `prev_export_id` is set **server-side** to the workspace's last `ready` export of the **same format + scope + exporter** — the draft renders the *exporter's* gaps and corroborations, so a diff against another member's export would show phantom deltas; the re-export diff (§ 3.2.8) is only meaningful **like-for-like.**
- **Every exported artifact carries an `"as of <timestamp>, export <id>"` line** — a snapshot ages, and the artifact says so itself. The re-export diff helps only the author; the "as of" line travels with the document.

#### 3.2.4 · Claims → prose stubs (provenance follows kind)

Each claim becomes a **prose stub whose provenance follows its kind** — the export **never flattens the kinds into one look**:

| Claim kind | Stub renders as | Provenance carried |
|---|---|---|
| **`extracted`** | machine-extracted prose | its `source_span` (and sources / equations where present) |
| **`user_authored`** | an explicit **author assertion** | when it carries the chunk-anchored pair (the review-queue split path), the stub carries its `source_span` as an **author-anchored quote** — never dressed as machine-extracted (losing the second half of a split quote would defeat the split) |
| **`synthesis`** | prose citing its inputs | its **input claims**, **permission-filtered with the exporter's identity** — an invisible input renders the "input not in your view" marker and contributes **nothing** to list or `.bib` |

**Every stub carries its citation(s) inline as Pandoc-style `[@key]`:**

- *extracted* → its citation record;
- *authored* → one per `sources_json` entry — **zero entries renders a bare attributed assertion, nothing invented**;
- *synthesis* → its inputs' records, permission-filtered as above.

The keys come from the **one citation renderer** (the same implementation as copy-as-cited, § UI) — which is what makes "every inline key resolves 1:1 against its own `.bib`" true **by construction**, not by instruction.

**The honesty marks survive into the stub** — the draft is the product's honesty on paper, not a laundered rendering of it:

- a stub never loses its claim's hedges;
- an `auto_accepted` ref's stub carries the **"machine-reviewed, not validated"** mark;
- a disputed claim (`validated_confidence = 0`) carries its **disputed** mark;
- a `retracted` / `concern` source carries its **retraction note**.

#### 3.2.5 · Edges → structure

Edges order and connect the stubs into argumentative structure:

- `depends_on` / `supports` / `refines` **sequence the build**;
- `contradicts` **surfaces a stated tension.**

#### 3.2.6 · Gaps → limitations

Each of the **exporting member's** open gaps over the working set (gaps are per-member) becomes an **explicit stated limitation** in the draft — a hole is **named, never hidden.** This is the property that makes a compose export different from a copy block: copy carries no gaps; mandatory limitations are what make the export a finished argument rather than a fragment.

#### 3.2.7 · Citations → reference list

The draft's reference list is **exactly the set of citation records its stubs cite inline** (§ 3.2.4) — rendered and keyed by the **one** citation implementation, **never the papers' own bibliographies** (`cites`-family edges are the *graph's* structure, not the *draft's* references).

- **`format: 'bib'`** exports this **same list** — which is why "every inline key resolves 1:1 against its own `.bib`" holds by construction.
- **Corroborations:** the exporter's **open** corroborations over the working set (state `new` or `seen`; `dismissed` / `resolved` never render) state their support **as a clause on the stub's own sentence** — the sentence keeps its single-claim mapping; never a free-standing sentence. The export **widens its citation scope by exactly those matched claims the exporter can view** (the renderer runs the same per-claim `can_view` as the route, with the exporter's identity); a corroboration whose matched claim the exporter cannot view is **omitted entirely** — clause and citation both. An artifact never names what its exporter cannot see.
- Citations to **superseded** claims carry their tombstone badge.
- **One work, one `.bib` entry:** entries dedupe over the citation identity; `cited_works` rows **never render** into copy or `.bib` at MVP — the reference list is exactly the inline-cited records, and the annotated-bibliography seam (§ 3.2.10) re-opens the question. Bibliographic **fields come from one place — the citation record itself** (enrichment refresh upgrades the underlying row's fields at write time, precedence enrichment > parser), so key and fields can never disagree at render time.

#### 3.2.8 · Re-export + diff

Re-export is **non-destructive** and shows a **diff since the last version** — the export is a snapshot view, **not a commit**; the workspace keeps evolving underneath it. Re-export leaves the workspace **bit-identical** and produces a correct diff against the prior snapshot.

#### 3.2.9 · Projection guarantee — mechanized

The composer is **deterministic templates over claims and edges — no LLM in the export path at all.** Every sentence in the output is template-generated from **exactly one claim** (or one edge connective), so *"every sentence traces"* is true **by construction**, not by instruction. This is what makes the 100%-traceability gate **testable instead of aspirational** (C-16: deterministic by construction; C-12: no silent fabrication).

The **provenance sidecar's domain, pinned:**

- a **body sentence** maps to its **claim** or **edge**;
- a **limitation sentence** (§ 3.2.6) maps to its **gap** (`gap_id` — the sidecar's third arm);
- **structural lines** — title, the "as of" line, section headings — are **not sentences** and carry **no mapping**.

Totality is over the body; exemption is **by class, never by omission.**

A later **optional LLM "smooth" pass is gated behind a validator** that emits `sentence → claim_ids` and **rejects any sentence without one** — it cannot ship before that validator exists. The deterministic composer ships regardless.

#### 3.2.10 · Stable export interface — formats and scopes

Markdown is the MVP target. The renderer is a **projection engine behind a stable export interface**, so future shapes plug into the *same* Compose stage **without touching the graph or any caller**:

- **Formats:** LaTeX and PDF rendering plug in later; their `format` values are **already reserved** in the underlying `exports` CHECK. The reference list is additionally exportable as **`.bib`**, rendered by the **same `POST /api/citations/render`** implementation as copy-as-cited — one key scheme everywhere by construction.
- **Scopes:** MVP ships **exactly one** — the working set. The named future projections (a paper's claims, a gaps report, an annotated bibliography, a claim dossier) are **seams**, additive on `exports.params_json`, **never a second pipeline.** (C-4: no speculative generality — the seams are reserved, not built.)

#### 3.2.11 · Two egress tempos, deliberately

| Tempo | Scope | Mechanism | Role |
|---|---|---|---|
| **copy-as-cited** | per claim / selection / answer (multi-select included) | one read-class `POST /api/citations/render` call — **no queue**, instant | feeds drafts continuously |
| **compose pipeline** (this surface) | whole working set | **async, provenance-complete** | projects the **finished argument** |

A multi-claim copy block **deliberately never poses as a workspace export** — copy carries no gaps; mandatory limitations are Compose-only (§ 3.2.6).

#### 3.2.12 · Compose acceptance

- Every sentence in an export is **template-traceable to exactly one** working-set claim (per its provenance kind) **or one edge connective**; **zero** free sentences. Author-asserted and synthesis content is **visibly marked**; hedges, the auto-accepted mark, the disputed mark, and retraction notes survive into the stubs.
- The template connectives are subject to the **forbidden-verbs** copy-snapshot fixture: a deterministic template asserting *"the system understands…"* is as much a credibility-ender on paper as in the UI.
- **Every** open gap of the exporter over the working set appears as a stated limitation; **none** silently dropped.
- **Re-export leaves the workspace bit-identical** and produces a correct diff.
- A real workspace exports to a Markdown draft: a paper went in as sources; a draft comes out.

#### 3.2.13 · Compose non-goals

- **AI paper-generation** — deliberately out. The substance is settled by the author on the graph; the wording is a thin, sourced rendering, not a model writing the paper.
- **LaTeX / PDF rendering** for MVP — deferred behind the stable export interface (Markdown ships first).
- **Any write-back** from the export to the graph — compose is read-only over the graph.

---

## Invariants

xread is the code-level enforcement of its slice. The constraints below are cited by `C-n` from [`xresearch-std/CODING-STYLE.md`](https://github.com/xresearch-it/xresearch-std).

| Property | Where it binds | Constraint |
|---|---|---|
| **Composes over the base + the `xkit` render substrate** (published entry points) — no engine-peer dependency, no peer-internals import. | the whole package | C-18 |
| **Pure core** — no Cloudflare/Workers/Pages import, no ambient clock / randomness / id. | the whole package | C-15 |
| **Deterministic by construction** — same library → same layout; same working set → same bytes. | § 3.1.7 layout · § 3.2.9 projection | C-16 |
| **Read-only over the graph** — compose appends export *events*, never mutates the substrate. | § 3.2.2 | C-17 |
| **Resolution states are a discriminated union**, exhaustively handled (`assertNever` default). | § 3.1.4 | C-6 · C-7 |
| **Branded ids** — `WorkspaceId`, `ClaimId`, `CitedWorkId` are never bare `string`. | every id boundary | C-9 |
| **Parse at the boundary** — every external response (OpenAlex / S2 / Unpaywall / parser) is Zod-validated at the seam. | § 3.1.6 enrichment | C-10 |
| **Errors are values at the seam** — `Result<T, E>`; an enrichment / resolution failure is a named state, never a swallow. | § 3.1.4 · § 3.1.6 | C-11 · C-12 |
| **Nothing unbounded** — whole-workspace export is async + budgeted; neighbourhoods truncate to a named cap. | § 3.1.7 · § 3.2.3 | C-22 · C-23 |
| **Additive evolution only** — reserved `format` values and export scopes extend the contract, never re-mean it. | § 3.2.10 | C-4 · C-25 |
| **The contract is the only source of shapes** — every field / route / column xread reads is read from `xcontract` or the schema first. | the whole package | C-2 |

---

## UI

xread's UI is built on **`xkit`** (the shared render component + design system + API client). Like every package repo, xread ships **its own** dev / inspector / demo surface, so the module's behaviour is visible **in isolation**.

- **One render component, selection-state, not a bespoke screen.** The citation graph is **one selection-state** of the shared render component (a subgraph) — selecting a node switches the **right render-component** to that entity's view (§ 3.1.5). There is no second viewer.
- **One citation renderer everywhere.** Inline keys (§ 3.2.4), the reference list and `.bib` (§ 3.2.7), and copy-as-cited (§ 3.2.11) all key off the **same** `POST /api/citations/render` implementation — one key scheme by construction.
- **Compose is an expand-mode, not a screen.** Compose **expands the right render-component into a focused editor** while the left map collapses to a rail. Same component, focused state — consistent with the one-render-component rule above.
- **Honest UX is non-negotiable:** the truncation affordance (§ 3.1.7) names its cap; resolution state (§ 3.1.4) is shown, never hidden; the export's "as of" line and surviving honesty marks (§ 3.2.4) render visibly. xread's UI never launders the substrate's honesty.

> **UI development convention.** All UI work on this package uses the `superpowers` `brainstorming` / `visual-companion` skill — a standing convention, no exceptions. No UI work starts without it.

---

## Build & identity

- **Identity** is the workspace name `@xresearch/xread`, stable regardless of where the repo lives — a package can change location or visibility without a single import changing.
- **Pure logic, environment-agnostic** — no Cloudflare/Workers/Pages imports; the edge/deploy layer is never a package's concern.

```
              citationGraph                          render
  store  ──────────────────▶  paper-reference graph  ──────▶  reader UI
  (read)                       · paper nodes                   (xkit, one
                               · cited_works (+ resolution)     render component)
                               · cites edges
                               · OA enrichment / one-click ingest
                                       │
                                       │ read-only
                                       ▼
                               compose (graph → Markdown draft)
                                · deterministic templates, no LLM
                                · provenance per claim-kind
                                · gaps → limitations
                                · inline [@key] == own .bib
                                · async export-queue, snapshot + diff
```

xread **builds** a graph from metadata and **projects** it to a sourced draft. It reasons about nothing, owns no content, and writes nothing back to the store — `render ∘ citationGraph(xsubstrate)`, exactly.
