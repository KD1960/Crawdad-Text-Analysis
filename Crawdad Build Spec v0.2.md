# Crawdad 2026 — Build Specification v0.2

**Date:** 2026-07-09
**Sources:** `Crawdad workflow.docx` (requirements, "WF"); Corman, S., Kuhn, T., McPhee, R., & Dooley, K. (2002). Studying complex discursive systems: Centering Resonance Analysis of organizational communication. *Human Communication Research*, 28(2), 157–206 ("JHC", cited by journal page); `Visualizer examples/` (reference renderings).
**Status:** v0.2 — all open decisions ruled on by Kevin 2026-07-09 (§12). Ready for M0.

---

## 1. Purpose and scope

Rebuild the Crawdad text-analysis product as a stand-alone, pure client-side web application implementing Centering Resonance Analysis (CRA) per JHC. Modules in scope: Generator, Visualizer, Comparator, Classifier, Theme Identifier, Latent Theme Identifier, Summarizer. Out of scope (per WF): Browser, Finder.

Design principles (WF "General"):

1. Stand-alone web site; also downloadable and runnable locally with no install.
2. Build one module at a time; validate/verify each before proceeding (§11).
3. The deterministic CRA pipeline is the product; LLM features are optional add-ons (§10).
4. Every output file carries the citation footer: *"Please cite: Corman, S., Kuhn, T., McPhee, R., & Dooley, K. (2002). Studying complex discursive systems: Centering Resonance Analysis of organizational communication. Human Communication Research, 28(2), 157–206."*

## 2. Architecture (decided 2026-07-09)

**Pure client-side single-page application.** All parsing, network math, and visualization run in the browser. No server, no user data leaves the machine (except optional LLM calls, §10).

| Concern | Choice | Rationale |
|---|---|---|
| Distribution | Static site (any host) + downloadable single-directory bundle; target: open `index.html` locally with no build step | WF General-1, -2 |
| POS tagging / NP chunking | **wink-nlp** (`wink-eng-lite-web-model`) — selected in M1 benchmark 2026-07-09 | F1 0.970 vs compromise 0.971 on the 20-sentence gold fixture, but wink's errors are modifier misses only, while compromise lost head nouns ("supply", "reports") and leaked determiners — worse failure modes for CRA. MIT license; vendored as `vendor/wink-bundle.js` |
| Graph metrics | graphology + graphology-metrics (Brandes betweenness), or a small audited in-house implementation | Betweenness must match Eq. 1 exactly; verify against reference values (§11) |
| Visualization | d3-force (force-directed = "spring" layout, WF 4c) | Standard, canvas/SVG export to PNG |
| PCA / K-means / MDS | ml.js family (ml-pca, ml-kmeans) + classical MDS (small in-house, ~50 lines) | Client-side, no backend |
| Spreadsheets (tokenization, latent codes) | SheetJS (xlsx, csv) | WF 2f, 8b |
| Optional file extraction | pdf.js (PDF→text), mammoth (docx→text) | Enhancement E-1, §9 |

Constraint check: Brandes betweenness is O(N·M). Book-length texts yield CRA networks of 10³–10⁴ nodes; acceptable in-browser (verify in M2 with *Out of the Crisis* ch. 1-scale text; run in a Web Worker to keep UI responsive).

URL ingestion (WF 1c): **dropped in v1** (decided 2026-07-09). Browser JavaScript cannot fetch arbitrary URLs due to CORS restrictions regardless of how the app is hosted; supporting URLs would require a server-side proxy. Users download the file first, then select it. Roadmap note: a hosted-only fetch proxy (e.g., Cloudflare Worker) could restore this feature later without changing the client-side architecture.

## 3. CRA formal definitions (normative)

These definitions govern all modules. Deviations are defects.

### 3.1 Unitizing and selection (JHC pp. 173–175)

- Utterance = sentence (or conversational equivalent). Sentence segmentation by the NLP library, validated in M1.
- From each utterance, extract noun phrases: a noun plus zero or more additional nouns and/or adjectives serving as subject or object. Determiners are dropped. Verb phrases are excluded. Pronouns are dropped by default (JHC p. 175); provide a toggle to retain them.
- Codable units = the nouns and adjectives within noun phrases, after tokenization substitutions (§4.2), stop-word removal, and stemming (§3.5).

### 3.2 Linking (JHC p. 176)

Within each utterance:

- L1. All center words are linked sequentially in order of occurrence.
- L2. Within any noun phrase of ≥3 words, link all pairs (e.g., "complex discursive system" → complex–discursive, discursive–system, complex–system).

- L3. **Inter-sentence linking (decided 2026-07-09, D-1):** the last center word of sentence *s* is linked to the first center word of sentence *s+1* (per WF 2h-ii "adjacent within or between sentences"). JHC p. 176 describes within-utterance linking only; inter-sentence linking was added to the Crawdad product post-2002 (Dooley, pers. comm., 2026-07-09), so this default matches the shipped product. An optional "JHC 2002" toggle disables inter-sentence links for paper-faithful replication.

### 3.3 Network (decided 2026-07-09)

Accumulating links over all utterances yields a **symmetric, undirected, valued** network (JHC p. 176). Edge value F_ij = number of times words i and j were linked. This supersedes WF 2h-iii ("unweighted"): weights are required by Eq. 4 and confirmed by Kevin.

Influence (betweenness) is computed on the **binarized** graph (an edge exists iff F_ij ≥ 1), consistent with Eq. 1's unweighted geodesics. F_ij is used only in pair resonance.

### 3.4 Indexing — influence (JHC Eq. 1, p. 177)

Influence of word i in text T:

```
I_i^T = [ Σ_{j<k} g_jk(i) / g_jk ] / [ (N−1)(N−2) / 2 ]
```

where g_jk = number of shortest paths between words j and k, g_jk(i) = number of those passing through i, N = number of nodes. This is normalized betweenness centrality. Disconnected graphs are expected (JHC p. 177); pairs with no path contribute 0.

### 3.5 Text normalization

- Stop words: applied after POS filtering (POS filtering already removes most function words; the stop list catches mis-tagged tokens). Ship a standard English list, user-editable.
- Stemming (decided 2026-07-09, D-2): **Porter stemming by default**; minimal plural stemming (strip -s/-es, JHC p. 175 style) available as a user option. Porter was adopted in the Crawdad product post-2002 (Dooley, pers. comm., 2026-07-09), so the default matches the shipped product; the option supports 2002-paper-faithful replication. No lemmatization (WF 2e). Display forms use the most frequent unstemmed surface form so output stays readable.
- Case-fold to lowercase. Retain the most frequent surface form for display.

### 3.6 Resonance (JHC Eqs. 2–6, pp. 178–179)

Word resonance (Eq. 2) and standardized word resonance (Eq. 3):

```
WR_AB  = Σ_i Σ_j  I_i^A · I_j^B · α_ij        α_ij = 1 iff word i of A = word j of B
WR'_AB = WR_AB / sqrt( Σ_i (I_i^A)² · Σ_j (I_j^B)² )
```

Frequency-weighted pair influence (Eq. 4), pair resonance (Eq. 5), standardized (Eq. 6):

```
P_ij^T  = I_i^T · I_j^T · F_ij^T
PR_AB   = Σ_{i<j} Σ_{k<l}  P_ij^A · P_kl^B · β_ijkl
PR'_AB  = PR_AB / sqrt( Σ_{i<j} (P_ij^A)² · Σ_{k<l} (P_kl^B)² )
```

β_ijkl = 1 iff word set {i,j} in A equals {k,l} in B (order-free) and both pairs are connected (F ≥ 1). Note: JHC states the β condition as "F_ij^A and F_kl^B both are equal to one"; this spec reads that as *connected* (F ≥ 1), the only reading consistent with frequency-weighted Eq. 4 (confirmed 2026-07-09, D-3).

All four values (WR, WR', PR, PR') are computed and reported wherever "resonance" is required. Modules that need a single standardized measure default to WR' (JHC p. 178 recommends standardization when positioning documents relative to one another).

## 4. File formats

### 4.1 `filename_cra.md` (decided 2026-07-09: full data + display threshold)

One per input text. Two sections:

**A. Human-readable summary** (WF 2j):

- Header: source filename, processing date, app version, parameter settings (stemming mode, pronoun toggle, linking mode, tokenization file used).
- Network summary: node count, edge count, and for degree, betweenness, closeness, eigenvector centrality: network-level (Freeman) centralization plus mean of node values (confirmed 2026-07-09, D-4). Per-node degree + influence appear in the node table.
- Node table, **display-filtered to influence > 0.005** (WF 2j-iii), sorted by influence descending: word, degree, influence, connected words.
- Citation footer (§1.4).

**B. Machine-readable block** — a fenced ```json code block containing the **complete** network: schema version, parameters, full node list (id, word, display form, degree, influence), full edge list (i, j, F_ij). All downstream modules parse only this block; the 0.005 threshold never affects computation.

Rationale: single self-describing file (WF intent), exact downstream math, human-skimmable. A `_cra.md` without a valid JSON block is rejected with a clear error.

### 4.2 Tokenization file (WF 2f)

Spreadsheet (.xlsx or .csv), no header row required: column 1 = string to find, column 2 = replacement (e.g., "supply chain management" → "SCM"). Applied to raw text before parsing, case-insensitive, longest-match-first, whole-word boundaries. Multi-word replacements effectively protect phrases from being split by the parser.

### 4.3 Latent codes file (WF 8b–c)

Spreadsheet: column 1 = theme name, column 2 = word. One row per (theme, word). Words are stemmed with the same pipeline settings before matching. (WF says "second columns contains words" — this spec normalizes to one word per row; also accept comma-separated words in one cell.)

### 4.4 Analysis outputs

Each module writes a Markdown report (with embedded JSON block for machine reuse) plus PNG for visuals. All carry the citation footer.

## 5. Application shell and flow (WF 1, 3)

1. Landing screen: two paths — **Generate new CRA file(s)** or **Analyze existing CRA file(s)** — plus the Crawdad logo.
2. File selection: native file picker (multi-select) + drag-and-drop (no URL field in v1, §2). Accepted for generation: .txt, .md, .pdf, .docx (PDF/Word converted to text in-browser, E-1). Accepted for analysis: `*_cra.md`.
3. Analysis menu with one-phrase descriptions and enforced file-count constraints (WF 3a–b):
   - **Visualizer** (≥1 file): "Draw the CRA network of a text."
   - **Comparator** (≥2 loaded; user picks the two to compare, WF 5a): "Measure resonance between two texts."
   - **Classifier** (≥5): "Cluster texts by mutual resonance."
   - **Theme Identifier** (≥5): "Find manifest themes across texts (PCA)."
   - **Latent Theme Identifier** (≥1): "Score texts against your own theme dictionary."
   - **Summarizer** (≥2): "Rank the most influential words across texts."

## 6. Generator (WF 2)

Pipeline, in order:

1. **Ingest** file(s) or URL; convert to UTF-8 text. Markdown accepted natively (WF 2c).
2. **Clean** (WF 2b): strip headers/footers/page numbers (repeated-line detection), image/table markup, boilerplate. Show a before/after diff preview; user approves or edits before proceeding. Deterministic rules by default; LLM-assisted cleaning optional (§10).
3. **Tokenization substitutions** (§4.2) if a file is supplied.
4. **Sentence segmentation → POS tagging → noun-phrase extraction** (§3.1).
5. **Stop-word removal, pronoun handling, stemming, case-folding** (§3.5).
6. **Linking** (§3.2) → accumulate valued network (§3.3).
7. **Influence** via Brandes betweenness on binarized graph, normalized per Eq. 1.
8. **Emit** `filename_cra.md` (§4.1). Batch mode: one output per input file, plus a batch log (files processed, node/edge counts, warnings).

Errors are surfaced per file (e.g., "no noun phrases found — is this prose?"); a bad file does not abort the batch.

## 7. Module specifications

### 7.1 Visualizer (WF 4)

- Input: ≥1 `_cra.md`; one rendering per file.
- Show top-K nodes by influence, K = 30 default, user-adjustable (WF 4d). Edges drawn among displayed nodes; edge thickness ∝ F_ij.
- Layout: d3-force (spring; charge + link forces tuned to reduce crossings, WF 4c). Node label = word; node size ∝ influence; nodes with influence > 0.05 highlighted in a distinct color class (WF 4e), others neutral — matching the look of the reference GIFs in `Visualizer examples/`.
- Canvas: 16:9 landscape (1920×1080 export) so it drops onto a PowerPoint slide (WF 4f). Filename rendered as title; Crawdad logo in a corner; citation line in footer.
- Export: PNG download (WF 4g). Interactive on screen (drag nodes, hover for exact influence).

### 7.2 Comparator (WF 5)

- Input: exactly 2 `_cra.md`.
- Compute WR, WR', PR, PR' (§3.6).
- Word lists: up to 10 most influential **common** words (present in both; ranked by min(I^A, I^B) so both texts must value the word) and up to 10 most influential **unique** words *per text* (ranked by influence in the owning text) (confirmed 2026-07-09, D-5).
- Output: `comparison_A_vs_B.md` — resonance table, the word lists with influence values, parameters, citation footer.

### 7.3 Classifier (WF 6)

- Input: ≥5 `_cra.md` (n files).
- Compute the n×n matrix of pairwise **WR'** (standardized; JHC p. 178 rationale) — PR' matrix also computed and included in output for reference.
- K-means on rows of the WR' matrix. K selection (revised 2026-07-09, D-8): target K₀ = round(√n); accept K₀ if its mean silhouette is within 0.10 (absolute) of the maximum over K = 2 … min(8, n−1); otherwise fall back to the best-silhouette K, preferring solutions without singleton clusters. Rationale: raw silhouette maximization favors degenerate splits (e.g., 8-vs-1) with small n. Full silhouette-by-K table with cluster sizes and singleton flags is always reported, the applied rule is stated in the output, singleton solutions carry a warning, and the UI allows manual K override.
- Output: `classification_<timestamp>.md` — resonance matrix (heat-shaded HTML table), cluster membership, silhouette table, parameters, citation footer.
- Post-analysis checkboxes (WF 6e): (a) aggregate each cluster's source texts into one file, (b) re-run Generator on aggregates, (c) emit `cluster_k_cra.md`, (d) run Visualizer on results. Requires original source texts still loaded in the session; if only `_cra.md` files were supplied, options (a)–(b) are disabled with an explanatory tooltip.
- Optional 2-D plot (WF 6f): classical **MDS** (multidimensional scaling) on distance d = 1 − WR', points colored by cluster (confirmed 2026-07-09, D-6).

### 7.4 Theme Identifier (WF 7)

- Input: ≥5 `_cra.md` (n texts).
- For every word appearing in any text, compute **summed total influence** across texts (not averaged; zeros count as zero — WF 7b example). Take top 50 words; build the 50×n word-by-text influence matrix.
- PCA on that matrix: **words are observations, texts are variables** — components are extracted from the n×n correlation matrix of texts; each component (theme) has an eigenvalue and loadings, and word scores identify which words define each theme (confirmed 2026-07-09, D-7). The alternative orientation (words as variables) is degenerate when texts number fewer than words.
- Output (WF 7d): `themes_<timestamp>.md` — all components ordered by eigenvalue descending; eigenvalue magnitudes with **eigenvalue > 1.0 color-coded**; loading table with **|loading| > 0.4 color-coded**; the 50-word list with summed influences; parameters; citation footer.

### 7.5 Latent Theme Identifier (WF 8)

- Input: ≥1 `_cra.md` + latent codes file (§4.3).
- For each theme and each text: summed total influence of the theme's words in that text (words absent from a text contribute 0).
- Report words in the codes file that were not found in any text (coverage check — silent zero-matching is a validity trap).
- Output: `latent_themes_<timestamp>.md` — theme × text table of summed influences, coverage report, parameters, citation footer.

### 7.6 Summarizer (WF 9)

- Input: ≥2 `_cra.md`.
- Executes Theme Identifier step (b) only: top-50 words by summed total influence, word-by-text influence matrix, output as a table.
- Note: as specified, this duplicates a subset of Theme Identifier. Optional LLM narrative summary is the feature that would differentiate it (§10, E-4). Retained as specified pending Kevin's view.

## 8. UI/UX requirements

- Every module screen shows: selected files, parameter controls with the defaults above, a Run button, progress indicator (Web Worker), and results pane with Download buttons.
- All defaults visible and changeable; every output records the parameters that produced it (reproducibility).
- Accessibility: keyboard navigable; color coding paired with symbols/bold so meaning survives grayscale printing (relevant to WF 7d color codes).

## 9. Enhancements beyond the workflow (suggestions, WF General-5)

- **E-1** PDF/DOCX ingestion in-browser (pdf.js, mammoth) so users skip manual conversion. **Pulled into v1 scope (decided 2026-07-09); delivered with M0.** Libraries vendored locally so the app stays offline-capable. Text-layer extraction only: scanned/image PDFs are detected and rejected with a warning (no OCR in v1).
- **E-2** Project bundle: save/load an entire session (all `_cra.md` + settings) as one .zip.
- **E-3** Batch Visualizer export (one PNG per file in one click).
- **E-4** See §10 LLM features.
- **E-5** Reference-corpus regression tests shipped with the app ("Verify install" button) — runs the M-series validation suite (§11) locally.
- **E-6** URL ingestion via a hosted fetch proxy (deferred from v1; see §2).

## 10. LLM features (decided 2026-07-09: optional, user-supplied API key)

**Status update (2026-07-09, D-10): dropped from the v1 UI per Kevin** — requiring users to
obtain and fund an Anthropic API key is too much friction. The module (`js/llm.js`, L-1–L-4)
is built, tested with a mock transport (8/8), and retained in the codebase unwired, so
re-enabling is a matter of restoring the UI hooks. The remainder of this section describes
the built-but-dormant design.

Off by default. User pastes an API key (stored in memory for the session only; never persisted, never sent anywhere but the model provider). Each feature labels its output as LLM-generated and non-deterministic. The CRA pipeline itself never depends on an LLM.

- **L-1 Cleaning assistant** (Generator step 2): propose removal of boilerplate/headers; user approves diff.
- **L-2 Pronoun disambiguation** (JHC p. 175 anticipates substituting disambiguated nouns): optional pre-processing pass replacing pronouns with referents; always shown as a reviewable diff.
- **L-3 Theme labeling**: suggest names for PCA components (from high-loading words) and Classifier clusters (from high-resonance shared words).
- **L-4 Narrative summary** for Summarizer/Comparator reports, generated strictly from the computed tables (prompt includes only Crawdad's own output — no raw text — to keep the summary grounded in the CRA results).

## 11. Build sequence, validation, and verification (WF General-3, -4)

One milestone at a time; each has machine checks (mine) and acceptance checks (Kevin's). No milestone starts until the prior one passes.

| M | Deliverable | Machine validation | Kevin's acceptance check |
|---|---|---|---|
| M0 | App shell, file I/O, `_cra.md` schema | Schema round-trip tests | Flow matches WF 1, 3 |
| M1 | Parser: sentence/POS/NP extraction | Hand-annotated 20-sentence fixture: precision/recall of noun-phrase words vs. manual annotation, both candidate libraries; pick winner | Review fixture disagreements; approve parser choice |
| M2 | Network + influence | Toy-text unit tests (incl. JHC "complex discursive system" linking example, p. 176); betweenness vs. published reference values (e.g., Freeman's star graph; cross-check vs. an independent implementation); performance on book-chapter text | Spot-check a `_cra.md` for a text Kevin knows well |
| M3 | Visualizer | PNG export dimensions; top-K and color threshold logic | Side-by-side vs. `Visualizer examples/` GIFs (Aesop, Macbeth, etc.) for qualitative equivalence |
| M4 | Comparator | Resonance unit tests: hand-computed WR/WR'/PR/PR' on two 3-sentence toy texts; identity check WR'_AA = 1 | Run on two texts with known relationship; face validity |
| M5 | Classifier | K-means determinism (fixed seed); silhouette correctness on synthetic clusters | Cluster a corpus with known groups (e.g., 3 authors × 3 texts) |
| M6 | Theme + Latent Theme | PCA vs. R/Python reference output on the same matrix (tolerance 1e-6); latent coverage report | Themes face-valid on a familiar corpus |
| M7 | Summarizer, LLM features, E-series | L-features degrade gracefully with no key | End-to-end walkthrough |
| M8 | Packaging: static deploy + downloadable bundle | Runs from `file://` with no network | Download, double-click, full workflow offline |

Golden-file principle: M2–M6 fixtures and expected outputs are committed with the app and re-runnable by anyone (E-5).

## 12. Decision log (all ruled on by Kevin, 2026-07-09)

| ID | Question | Ruling |
|---|---|---|
| D-1 | Link center words across sentence boundaries? (WF vs. JHC p. 176) | **Yes** — matches post-2002 Crawdad product; "JHC 2002" toggle available |
| D-2 | Stemming depth | **Porter by default** — matches post-2002 Crawdad product; minimal plural (JHC 2002) as option; no lemmatization |
| D-3 | β condition in Eq. 5: F "equal to one" read as "connected (F ≥ 1)" | **Agreed** |
| D-4 | `_cra.md` centralities: network-level centralization indices + per-node degree/influence table | **Agreed** |
| D-5 | Comparator unique words: 10 per text | **Agreed** |
| D-6 | "MDA plot" = classical MDS on 1 − WR' | **Yes** |
| D-7 | Theme Identifier PCA: words = observations, texts = variables | **Yes** |
| D-8 | Classifier K selection: √n target with silhouette tolerance check (0.10), singleton-aware fallback, manual override | **Kevin's proposal, 2026-07-09** — replaces raw silhouette maximization |
| D-9 | Theme Identifier: varimax rotation (Kaiser-normalized) of components with eigenvalue > 1, for simple structure; unrotated solution still reported; validated against independent numpy implementation to 1e-6 | **Kevin's request, 2026-07-09** |
| D-10 | LLM features (L-1–L-4) dropped from the v1 UI; module built, tested, and retained unwired in `js/llm.js` | **Kevin, 2026-07-09** |

Earlier rulings (same date): pure client-side architecture; valued edges (F_ij) with influence on the binarized graph; full data + display threshold in `_cra.md`; LLM features optional via user-supplied API key; URL ingestion dropped from v1 (E-6).

---

*Please cite: Corman, S., Kuhn, T., McPhee, R., & Dooley, K. (2002). Studying complex discursive systems: Centering Resonance Analysis of organizational communication. Human Communication Research, 28(2), 157–206.*
