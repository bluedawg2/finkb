## Project

**finkb — Financial Knowledge Companion.** A local desktop app that collects, evaluates and searches
high-quality personal-finance education (agency pages, PDFs, expert articles, video and podcast
metadata) and shows ranked, explainable results with provenance. It never gives advice.

`PRD.md` is the single source of truth. Read its "Phased build plan" and "Open questions" sections
before any work; the four Phase 0 decisions there are settled.

## Constraints (from PRD.md)

- Rust-only core: Cargo workspace `finkb-core` (all domain logic, Tauri-free), `finkb-cli`, `src-tauri` (thin shell). No Python sidecar before the Phase 1 PDF spike decides it.
- One SQLite file `finkb.sqlite`: FTS5 + sqlite-vec; migrations numbered in `finkb-core/src/migrations/`, additive, never edit a shipped one.
- Content rights: every `source` row has a `posture` (`full_text` / `metadata_link` / `unclear` / `blocked`) with an evidence URL; `unclear` is treated as `metadata_link`. No chunk from document text unless posture is `full_text`.
- Corpus budgets are enforced in code, never warned. Store extracted text, not pages.
- AI: local-only for anything that touches user queries or displayed text. Nothing ships without an eval row.
- SQL is parameter-bound. Fetches honor robots.txt, block SSRF, and cap size and pages.

## Workflow

- Phase 0 and 2 are Tier 1 (`/quick`); Phases 1, 3 and 4 are Tier 2 (`/phase` with one plan-adversary pass). Tier-2 paths: `fetch/`, `extract/`, `ai/`, `migrations/`.
- Each phase closes only when its scripted gate passes on `finkb eval`.
- Commits: no real financial data, no PII; synthetic fixtures only.
- Prefix shell commands with `rtk`; use plain `npx tsc` / `npm test`.

## Phase 0 scope (current)

Workspace scaffold; migrations 001–003 (`source`, `eval_query`, `eval_label`); seed registry with
posture evidence URLs, `serves_questions`, budgets and the expert registry (verified feeds and
YouTube channel IDs); 70 labeled queries; `finkb eval` scoring a TREC run file.
Gate: two labeling passes agree ≥ 80%; every registry row has a posture evidence URL or `unclear`
with a review date.
