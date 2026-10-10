# Financial Knowledge Companion — PRD v0.2 (Build-Ready)

Oct 10, 2026

## Summary

v0.2 turns the v0.1 principles into decisions: build a Rust core on one SQLite file, ingest only allowlisted government HTML and PDFs for MVP, keep the source registry in SQLite (not Obsidian, not NotebookLM), and gate every phase on a hand-labeled eval set that exists before the ranker does.

| v0.1 gap | v0.2 decision | Section |
| --- | --- | --- |
| No MVP source choice | Government and regulator HTML + PDF from a 15-site allowlist; everything else metadata + link or deferred | 5 |
| No users or success criteria | One user (the founder); P@5 ≥ 0.6, provenance 100%, zero silent drops | 2, 12 |
| Crawler unspecified | Allowlist walker, 1 req/s, RFC 9309, conditional GET + text hash, every outcome recorded with an enum reason | 7 |
| No stack | reqwest, dom\_smoothie, htmd, pdfium-render, FTS5 + sqlite-vec + RRF, fastembed bge-small, local LLM via loopback provider; no spider, no Python unless PDFs force it | 6 |
| Quality model was a list | 12 named dimensions, each with a method and a rendered reason; 4 human-curated tiers; freshness by subject class; no composite score | 10 |
| "Why relevant" undefined | Template over stored fields, never generated prose in MVP; AI output lives only in `ai_annotation` with a review state | 9, 11 |
| No schema | Ten tables; posture enforced by trigger; transport, content and curation identity kept separate | 8 |
| No content-rights policy | `full_text` only with a fetched terms page on record; default `metadata_link`; FINRA, Morningstar, YouTube blocked | 5 |
| Eval deferred | 70 labeled queries in Phase 0; ablations every run; fixed hazard regression suite | 12 |
| Obsidian or NotebookLM? | Neither as system of record. Obsidian is a Phase 3 one-way export; NotebookLM has no role; Zotero optional for papers in Phase 4 | 4 |
| Build vs buy | Assemble: no tool ships explainable source scoring, so that layer is the product; capture and extraction are commodities | 3 |

Added after review: a corpus-governance section (admission by seed question, hard budgets, text-not-pages, dormancy), an expert track (named Tier 2–3 registry, indexed surrogates for metadata-only sources, legal intake via RSS, clipper, YouTube Data API and podcast RSS, and a read / watch / listen format field with coverage gates), and a learning-experience section (six-rung answer ladder, evidence-labeled design rules, number and probability presentation, WCAG checklist).

Four owner decisions block Phase 0 (section 14): repo name, Python sidecar policy, hosted-model policy, and handling of `unclear` posture.

Research caveats: several agency terms pages (SSA, investor.gov, medicare.gov) and most commercial publishers returned 403 to the fetcher, so their posture is `unclear` until a human opens them; 2026 limit figures quoted in section 10 come from search summaries and are placeholders for the eval set only.

## Product definition

A local research tool that answers a personal-finance question with ranked, explained, cited sources, and never with advice.

**Vision.** A user asks "How much do I need to retire?" and gets the five most useful public sources, each with why it ranks where it does, what kind of thing it is (fact, analysis, opinion), how fresh it is, and a link to the original. Workflow: question → intent → retrieve → rank → explain → provenance → dig deeper.

**Primary user (MVP).** The founder, as a long-time Quicken user researching his own retirement questions. One user, one machine, Windows desktop. No accounts, no sharing.

**Second user (post-MVP).** A CompassWorth user who taps "research this" from a dashboard; private context (age, assets, goals) arrives through an authenticated, data-minimizing interface and never enters the knowledge store.

**Non-goals for v1.**

- No financial advice, projections or calculators; the product ranks sources, it does not compute answers.
- No YouTube, podcasts, books or paywalled content. Source types beyond HTML and PDF wait for Phase 4.
- No claim extraction, contradiction detection or research synthesis.
- No multi-user, sync or hosted service.
- No hand-built UI polish beyond what the eval loop needs; the MVP UI is a search box and a result list.

**Success criteria (acceptance gate for MVP, section 12 has the method).**

| Measure | Target |
| --- | --- |
| Precision@5 on the 10 seed questions, graded by hand | ≥ 0.6 mean; ≥ 0.4 on every question |
| Provenance correctness (URL, title, publisher, dates resolve to the real page) | 100% of shown results |
| Explanation present and specific (names the dimension and the evidence) | 100% of shown results |
| Corpus size at gate | ≥ 300 documents from ≥ 12 registered sources |
| Fresh re-crawl of the whole corpus | < 30 min on the dev desktop, fully offline after fetch |
| Search latency, warm | < 500 ms lexical; < 2 s with rerank |
| Ingestion failure visibility | every failed fetch or parse has a row with a reason; zero silent drops |

The numbers above are targets set now, not measurements. They are revised once the first labeled set exists; the doc records both.

## Alternatives research: build, buy or assemble

Assemble a thin layer; do not adopt a platform. No existing tool crawls, scores source quality on explainable dimensions, ranks and cites. The scoring and ranking are the product; fetch, extract, archive and embed are commodities.

### Capture and archive tools (self-hosted)

| Tool | License | What it adds | Why not the core |
| --- | --- | --- | --- |
| [Karakeep](https://github.com/karakeep-app/karakeep) (ex-Hoarder) | AGPL-3.0 | Clipper, yt-dlp video archive, AI tags via Ollama, REST API, importers | Docker stack (web + Meilisearch + Chrome); its tags are not quality dimensions |
| [ArchiveBox](https://github.com/ArchiveBox/ArchiveBox) | MIT | Best fidelity: WARC + `index.sqlite3`, yt-dlp with subtitles | REST API is alpha; Python stack |
| [Readeck](https://codeberg.org/readeck/readeck) | AGPL-3.0 | Single binary, SQLite, immutable ZIP per bookmark, FTS | API and PDF support not documented |
| [Linkwarden](https://github.com/linkwarden/linkwarden) | AGPL-3.0 | Screenshot + PDF + single-HTML per page | Human-facing; weak programmatic ingest |
| [SingleFile](https://github.com/gildas-lormeau/SingleFile) | AGPL-3.0 | One self-contained HTML per page, CLI | A primitive, not a system |

### Local RAG platforms

| Tool | License | Strength | Verdict |
| --- | --- | --- | --- |
| [Kotaemon](https://github.com/Cinnamon/kotaemon) | Apache-2.0 | Best citation UX: PDF viewer with highlights, per-citation relevance score, weak-relevance warning | Borrow the citation pattern, not the app |
| [AnythingLLM](https://github.com/Mintplex-Labs/anything-llm) | MIT | Local embedder + LanceDB, citations in chat | Generic RAG, no source-quality model |
| [RAGFlow](https://github.com/infiniflow/ragflow) | Apache-2.0 | Traceable citations, sitemap ingest | Needs Elasticsearch, MySQL, MinIO, NATS, ClickHouse; 16 GB RAM |
| [Open WebUI](https://docs.openwebui.com/features/chat-conversations/rag/) | — | BM25 hybrid + cross-encoder rerank | Chat UI only; no corpus pipeline |
| [Onyx](https://github.com/onyx-dot-app/onyx) (ex-Danswer) | MIT CE | 50+ connectors, own crawler | Redis + MinIO + inference servers; too heavy |
| [txtai](https://github.com/neuml/txtai) | Apache-2.0 | Sparse + dense in one embeddings DB, Python | Good library if a Python sidecar exists anyway |

### Hosted services (discovery or fallback only)

| Service | Price | Role here |
| --- | --- | --- |
| [Firecrawl](https://www.firecrawl.dev/pricing) | Free 1,000 credits/mo; Hobby $16/mo | Fallback for JS-heavy pages |
| [Jina Reader](https://jina.ai/reader/) | Free, 20 RPM keyless, 500 RPM with key | Zero-setup fallback; reads PDFs; sends URLs to a third party |
| [Exa](https://exa.ai/pricing) / [Tavily](https://www.tavily.com/pricing) | Exa $4–15 per 1k searches; Tavily 1,000 free credits/mo | Source discovery, never storage |
| [Readwise Reader](https://readwise.io/pricing) | $9.99/mo annual | Capture + highlights; hosted dependency |
| [Elicit](https://elicit.com/pricing) / Consensus | Elicit Pro $49/mo | Academic papers only |
| [Crawl4AI](https://github.com/unclecode/crawl4ai) | Apache-2.0, self-host | Best Python sidecar: clean markdown, JS rendering, v0.9.4 (2026-09-23) |
| [Trafilatura](https://github.com/adbar/trafilatura) | Apache-2.0 | Best-regarded main-text + metadata + date extractor; sitemap and feed discovery |

### Rust crates that cover the crawl layer

[spider](https://github.com/spider-rs/spider) (MIT, HTTP-first with optional Chrome), [dom\_smoothie](https://github.com/niklak/dom_smoothie) (MIT, Readability.js port, small community), [htmd](https://github.com/letmutex/htmd) (Apache-2.0, HTML to Markdown), [texting\_robots](https://github.com/Smerity/texting_robots) (robots.txt with crawl-delay and sitemaps, fuzz-tested).

### Finance-specific knowledge bases

Nothing close exists. One search for personal-finance RAG or Bogleheads-wiki pipelines returned only generic RAG lists. Treat the gap as likely, not proven.

### Three viable architectures

1. **Karakeep as capture layer + own ranker.** Fast clipper and mobile apps; cost is a second Docker stack beside a Tauri app and AGPL boundaries.
2. **Rust core + SQLite FTS5 + own ranker, thin Python sidecar for hard pages and PDFs (recommended).** Matches the CompassWorth stack; provenance and scoring live in the schema, so "why this ranked first" is columns, not a model. Cost: crawler glue, PDF and YouTube handling, citation UI are yours.
3. **Adopt a RAG platform.** Citations and chat free, but opaque relevance and no source-quality model; scoring would be bolted on from outside.

Licensing note: Karakeep, Linkwarden, Readeck, Khoj and SingleFile are AGPL-3.0. Calling their APIs is fine; embedding them in a closed app is not.

Not verified from vendor pages: Consensus and Perplexity pricing (403 on vendor page), Open WebUI license, Readeck API, ArchiveBox search backends. Extraction quality claims are the projects' own; none were tested here.

## Obsidian vs NotebookLM vs own store: decision

SQLite is the system of record. Obsidian is an optional human-facing view generated from it. NotebookLM is not a store, an API or a search backend for this product. Zotero is the one outside tool worth adding, and only for papers and PDFs.

|  | Obsidian | NotebookLM (Gemini Notebook) | Zotero | Own SQLite |
| --- | --- | --- | --- | --- |
| Programmatic write | Filesystem markdown + YAML (best); [CLI](https://obsidian.md/cli) and [Local REST API](https://github.com/coddingtonbear/obsidian-local-rest-api) need the app running | Enterprise-only [API v1alpha, Preview](https://docs.cloud.google.com/agentspace/notebooklm-enterprise/docs/api-notebooks); no consumer API; unofficial libs use undocumented endpoints | [Local API](https://www.zotero.org/support/dev/web_api/v3/local_api) writes need Zotero 10+ and a user-approved key; [Web API](https://www.zotero.org/support/dev/web_api/v3/write_requests) 50 items/request | Unlimited, transactional |
| Programmatic read/search | Filesystem; REST `/search/` JsonLogic over frontmatter | Chat only; Enterprise 500 queries/user/day | Local API, no rate limit | SQL + FTS5 |
| Provenance fields | Free-form [properties](https://obsidian.md/help/properties): text, list, number, checkbox, date, tags; no nesting | Little beyond the source itself | Strong bibliographic schema, access date, snapshot | Whatever the schema defines |
| Cost | [Free incl. commercial](https://obsidian.md/license); Sync $4/mo | Free tier; paid tiers unverified | Free; cloud storage $20–120/yr | Free |
| Limits | None found | [50–600 sources per notebook by plan](https://support.google.com/notebooklm/answer/16213268); 500,000 words/source | Local unlimited | Disk |
| Export / lock-in | Plain markdown; none | Export not documented; high | Open formats, local SQLite | None |
| Offline | Yes | No (Enterprise runs in your GCP project) | Yes | Yes |
| Fit as source registry | Good as a view; weak typing, no transactions | Poor | Poor for custom fields (tier, license, crawl rules) | Best |
| Fit as content store | Clipped pages and notes; not bulk crawl | Poor: hard source cap | Good for papers, PDFs, snapshots | Good; raw files on disk |
| Fit as search backend | Fair (Omnisearch BM25) | No control over ranking | Fair | Best; scoring is ours |

### Decisions

1. **Source registry lives in SQLite** (`source` table, section 8). It needs typed fields, transactions and joins to fetch history; Obsidian properties are untyped and unenforced.
2. **Obsidian export is a Phase 3 feature, off by default.** One markdown note per source with frontmatter (`tier`, `license_posture`, `crawl_rules`, `last_fetched`); [Bases](https://obsidian.md/help/bases) gives a browsable table. Export is one-way. If hand-editing tiers in Obsidian is wanted later, add an import step with explicit conflict handling.
3. **NotebookLM has no role in the build.** Source caps of 50–600 per notebook, cloud-only data, Preview-grade Enterprise API, no documented export. It stays a personal reading aid on hand-picked sets.
4. **Zotero is optional for the academic-paper source type.** The Connector captures metadata + PDF + snapshot; the app reads `zotero.sqlite` or the local API. Decide when the paper source type is scheduled (Phase 4).

Unverified: NotebookLM paid-tier prices and compute-based usage limits (secondary sources only); the July 2026 rename to Gemini Notebook (Google support pages use the name, no announcement found); Obsidian Web Clipper batch/PDF/YouTube support; whether Zotero 10 has shipped.

## MVP scope, seed source registry and content-rights policy

MVP ingests two source types, HTML pages and PDFs, from a hand-curated allowlist of about 15 sites. No link-following beyond the allowlist. Everything else waits.

### Source types

| Type | MVP | Later | Why |
| --- | --- | --- | --- |
| Government and regulator HTML + PDF | Yes |  | Public domain, static, no JS, highest authority for fact-type content |
| Research publishers (NBER, SSRN, arXiv, CRR, EBRI) | Metadata + link only | Full text where the paper's own license allows | Author-held copyright; arXiv metadata is CC0 |
| Practitioner blogs and RSS (Kitces, ERN, Bogleheads) | Metadata + link via RSS where offered | Full text after per-site terms check | Terms ban systematic retrieval; RSS is Kitces' explicit exception |
| Commercial research (Vanguard, Morningstar, Fidelity, Schwab) | No | Metadata + link | Terms prohibit scraping and storage |
| YouTube transcripts, podcasts | No | Only via official API or user-supplied files | [YouTube ToS](https://www.youtube.com/t/terms) bans automated access; yt-dlp caption pulls breach it |
| Books | No | Metadata only | Never store text |

### Seed source registry (MVP allowlist)

| Source | Posture | Evidence | Crawl rule |
| --- | --- | --- | --- |
| [irs.gov](https://www.irs.gov/about-irs/use-of-content-from-irsgov) | full\_text | Federal content "not subject to copyright"; credit requested; third-party material excluded | 1 req/s; sitemap-driven; `/pub/irs-pdf/` and `/retirement-plans/` first |
| [consumerfinance.gov](https://www.consumerfinance.gov/privacy/website-privacy-policy/) | full\_text | CFPB content "in the public domain"; citation requested | 1 req/s; `/consumer-tools/` and `/data-research/` |
| [sec.gov](https://www.sec.gov/about/privacy-information) | full\_text | May be copied or redistributed; cite SEC; declared User-Agent required; hard 10 req/s cap | 1 req/s max; declared UA with contact email |
| investor.gov | full\_text (assumed, SEC-operated) | Terms not fetched | Verify before first crawl |
| [federalreserve.gov](https://www.federalreserve.gov/disclaimer.htm) | full\_text | Public domain unless marked; no framing | 1 req/s |
| [bls.gov](https://www.bls.gov/opub/copyright-information.htm) | full\_text | Public domain except marked photos | 1 req/s; CPI and employment-cost pages |
| ssa.gov | full\_text (assumed, 17 USC §105) | Terms page returned 403 | Verify manually before first crawl |
| fdic.gov | full\_text (archive subsite verified; main site unclear) | [archive.fdic.gov](https://archive.fdic.gov/Content%20and%20Copyright) says public domain | Verify main-site terms |
| medicare.gov | metadata\_link until verified | CMS guidance says link, don't duplicate | Verify |
| [treasurydirect.gov](https://www.treasurydirect.gov/legal-information/terms) | metadata\_link | Terms: personal non-commercial use only, copying needs consent; conflicts with §105 but take the cautious reading | Rates via Treasury data APIs if their terms allow |
| [finra.org](https://www.finra.org/terms-of-use) | metadata\_link | Private SRO; no scraping, no database building, no ML/AI use | Manual bookmarking only |
| [nber.org](https://nber.org/policies.html) | metadata\_link | Author copyright; mass download not permitted | Abstract + link |
| [arxiv.org](https://info.arxiv.org/help/api/tou.html) | metadata (CC0); full text per-paper license | Legacy API: 1 request per 3 s | q-fin.\* via API only |
| [kitces.com](https://kitces.com/terms-of-use/) | metadata\_link; RSS allowed | Bans bots and systematic retrieval; RSS explicitly allowed | RSS feed only |
| Bogleheads wiki | unclear | Copyright page returned 402 | Check footer license; metadata\_link until known |

### Content-rights policy

`posture` is a required field on every source and is enforced at write time, not by convention.

| Posture | Stored | Shown | Allowed when |
| --- | --- | --- | --- |
| `full_text` | Raw bytes, extracted text, chunks, embeddings | Snippets and highlighted passages with attribution | A fetched terms page or statute grants it; the URL and fetch date of that page are recorded on the source |
| `metadata_link` | Title, author, publisher, dates, canonical URL, our own one-paragraph summary, hash of the page for change detection | Metadata and our summary; the original is one click away | Default for anything not proven `full_text` |
| `unclear` | Same as `metadata_link` | Same | Terms unreadable or contradictory; carries a review-by date |
| `blocked` | Nothing except the URL and the reason | Nothing | Terms forbid automated access (FINRA, Morningstar, YouTube) |

Rules: a source with `unclear` or `blocked` posture cannot have chunk rows; the schema enforces it with a trigger. Third-party material on a public-domain page (quoted reports, images) keeps its own copyright, so extracted images are never stored. A posture change from `full_text` downward deletes chunks and embeddings in the same transaction.

AI-generated summaries, classifications and explanations are stored in `ai_annotation` rows, never in `document` text columns, and are shown with an "AI" marker.

## Corpus governance: no dumping ground

Every source must name the seed questions it serves, every fetch is charged against a budget, and text is stored instead of pages. A document that nobody ever sees is pruned.

### Admission

- A `source` row carries `serves_questions` (seed-question IDs). A source serving zero questions is never crawled; the registry UI refuses to save it.
- Pull, not crawl: a URL under an allowed prefix is fetched only if its sitemap path, title or anchor text matches the topic vocabulary (`topic_terms` per source, e.g. `retirement`, `401(k)`, `IRA`, `Social Security`, `withdrawal`), or it is a listed seed URL. Everything else stays `discovered`.
- IRS has thousands of PDFs; the vocabulary rule keeps the MVP at the roughly 150 that cover retirement accounts, Social Security interaction, education savings and Medicare.

### Budgets (enforced in code; a crawl stops at the cap, it does not warn)

| Budget | Default | Where |
| --- | --- | --- |
| Documents per source | 200 | `source.max_documents` |
| Documents in corpus | 5,000 | `settings.max_documents` |
| Blob storage (PDF bytes) | 2 GB | `settings.max_blob_bytes` |
| Single fetch | 20 MB body, 200 PDF pages | section 7 |
| URLs fetched per crawl run | 2,000 per source | section 7 |

### What is stored

| Content | Stored | Not stored |
| --- | --- | --- |
| HTML page | Extracted Markdown, zstd-compressed, typically 10–40 KB | Raw HTML, images, CSS, scripts, screenshots, WARC |
| PDF | Raw bytes (cannot be re-extracted later) under the blob cap, plus extracted text | Rendered pages, embedded images |
| Video, podcast | Metadata surrogate only (next section) | Audio, video, thumbnails |
| `not_modified` fetch | One `fetch` row | Any blob |

Sizing estimate, not measured: 5,000 HTML documents at 30 KB compressed is about 150 MB; PDFs are the only thing that can hurt, hence the separate 2 GB cap.

### Retention and decay

- Keep the last 3 text versions of a document, plus any version an `eval_label` or user note points at. Older versions keep their `fetch` row and hash, lose their text.
- A document that has not appeared in any top-20 result for 12 months drops to metadata-only (`status = dormant`): text and chunks deleted, surrogate and provenance kept, re-fetched on demand if it surfaces again.
- A source disabled by the user keeps its rows for 90 days, then documents go dormant.

### Visibility

A monthly corpus report in the app: documents and GB per source, never-surfaced count, dormant count, budget headroom, pruned items with reasons. The eval runner fails if the corpus exceeds any budget.

## Expert track, surrogates and video

Agencies supply facts; named experts supply analysis and opinion. Both are in the MVP corpus, experts enter through legal intake paths as indexed surrogates rather than crawled text, and every item carries a format (read, watch, listen) so the result list can match how someone wants to learn.

### Who counts as an expert (Tier 2–3 admission)

Named author; stated credentials or institutional affiliation; method or data shown for analysis claims; conflicts disclosed; a corrections or update practice. Sources that sell the product they discuss are admitted with a `conflict_tags` entry, never silently.

### Starting registry (verify channel IDs, feeds and terms in Phase 0)

| Expert / outlet | Formats | Intake path | Posture | Conflict tag |
| --- | --- | --- | --- | --- |
| Michael Kitces, kitces.com and Kitces YouTube | read, watch | RSS (explicitly allowed by terms); YouTube Data API | metadata\_link + surrogate | Advisor platform |
| Wade Pfau, Retirement Researcher (site, YouTube), JFP papers | read, watch | RSS; YouTube Data API; paper metadata | metadata\_link + surrogate | Sells courses |
| Big ERN, Early Retirement Now SWR series | read | RSS; user clipper | metadata\_link + surrogate | None known |
| Bogleheads wiki and Bogleheads conference YouTube | read, watch | Wiki per license (verify); YouTube Data API | unclear until license read | None |
| Christine Benz and Jeff Ptak, Morningstar The Long View | listen, read | Podcast RSS; Morningstar articles blocked by terms | metadata\_link + surrogate | Morningstar products |
| Rob Berger (site, YouTube) | read, watch | RSS; YouTube Data API | metadata\_link + surrogate | Affiliate links |
| Mike Piper, Oblivious Investor | read | RSS; user clipper | metadata\_link + surrogate | Sells books |
| Allan Roth, Jonathan Clements (HumbleDollar) | read | RSS | metadata\_link + surrogate | None known |
| Ben Felix, Rational Reminder (YouTube, podcast) | watch, listen | YouTube Data API; podcast RSS | metadata\_link + surrogate | Canadian focus; advisor firm |
| CRR Boston College, EBRI, NBER authors (Bengen, Shoven, Coile) | read | Site RSS; paper metadata | metadata\_link; full text where the paper's license allows | None |

### Surrogates: how metadata-only sources still rank

A `surrogate` is the indexed stand-in for a document we may not store: title, publisher, author, date, format, duration, our own summary (≤ 150 words, AI-proposed, review state recorded), up to five extracted claims, and quotes of at most two sentences each, at most two per document. Surrogates are chunked, FTS-indexed and embedded exactly like full-text chunks, so an expert video ranks beside an IRS page. Snippets on expert results show the surrogate; the original is one click away.

### Intake paths (all legal-by-construction)

| Path | What arrives | Rules |
| --- | --- | --- |
| RSS / Atom | Title, link, summary, date | Only feeds the site publishes; honor feed TTL |
| User clipper ("add this page") | One page the user chose, extracted text kept privately as a personal copy; surrogate indexed | Never batch; never follow links; a personal reading copy, not redistribution. Judgment call, not legal advice |
| YouTube Data API v3 | Video title, description, channel, publish date, duration, chapters parsed from description timestamps | [Developer policies](https://developers.google.com/youtube/terms/developer-policies): API data stored no longer than 30 days unless refreshed (III.E.4.c–d); no scraping (III.E.6); no downloading or caching audiovisual content (III.E.1.a); only YouTube API Services may be used to retrieve data (III.I.14), so yt-dlp is out; YouTube must be shown as the source (III.F.2); embedded player with autoplay off. [captions.download](https://developers.google.com/youtube/v3/docs/captions/download) requires edit permission on the video, so transcripts are out unless the creator publishes one |
| Podcast RSS | Episode title, show notes, enclosure URL, publisher transcript when the feed carries `podcast:transcript` | Audio never stored; link to the publisher's player |

YouTube refresh job: every 28 days, `videos.list` on every stored video (about 1 unit each against the [10,000-unit daily quota](https://developers.google.com/youtube/v3/getting-started)); deleted or private videos are removed within the window. `search.list` is limited to 100 calls per day by default, so discovery runs from channel upload playlists, not search.

### Format as a first-class field

`document.format` ∈ {read, watch, listen}. Result cards show a format badge and duration; the list has Read / Watch / Listen chips, default all on, preference remembered. Chapter timestamps become deep links (`youtu.be/<id>?t=<sec>`) so a result can point at the minute where the expert covers the question. The learning-experience section below sets how formats are mixed and sequenced.

### Coverage gates

- For every seed question, the labeled set must contain at least one Tier 2–3 source and, where one exists, at least one watch or listen item.
- For analysis-intent queries, the top 5 must include at least one Tier 2–3 result; the regression suite fails if any seed question returns only agency documents.
- The corpus report shows the Tier 1 / Tier 2–3 ratio and the read / watch / listen mix per topic.

## Architecture and tech stack

Rust-only core, one SQLite file for corpus, index and provenance, and a Tauri + React shell. A Python sidecar is allowed for one job (table-heavy PDFs) and only after Rust extraction fails on real IRS and SSA PDFs.

&#91;embedded content: component architecture · 4 pipeline stages, 1 store, 4 consumers\]

The pipeline writes every stage's output and every failure to the same file the ranker reads; the eval runner and the Obsidian export are read-only consumers.

### Stack decisions

| Concern | Choice | Version | Why | Rejected |
| --- | --- | --- | --- | --- |
| Workspace | Cargo workspace: `finkb-core` (Tauri-free), `finkb-cli`, `src-tauri` shell |  | Same core/shell split as CompassWorth; core testable without an app | Python service |
| Fetch | [reqwest](https://crates.io/crates/reqwest) + small frontier table in SQLite | 0.13.5 | Polite, auditable, allowlisted crawl in less code than a framework | [spider](https://github.com/spider-rs/spider) 2.53.9: fine, but a framework for a 15-site allowlist |
| robots.txt | [texting\_robots](https://crates.io/crates/texting_robots) | 0.2.2 (2023) | Spec is stable (RFC 9309); fuzz-tested |  |
| HTML extraction | [dom\_smoothie](https://crates.io/crates/dom_smoothie) + [htmd](https://crates.io/crates/htmd) | 0.18.2 / 0.5.5 | Readability port + HTML-to-Markdown, both MIT/Apache | Trafilatura (Python, best-regarded; sidecar only if Rust output fails) |
| PDF | [pdfium-render](https://crates.io/crates/pdfium-render) | 0.9.4 | Mature text layer; ships a pdfium binary | [mupdf](https://crates.io/crates/mupdf) 0.8.0 is AGPL; [pdf\_oxide](https://crates.io/crates/pdf_oxide) is pre-1.0 with self-reported benchmarks |
| PDF tables (later) | [Docling](https://github.com/docling-project/docling) sidecar, isolated subprocess |  | MIT; layout + table structure; only if needed | marker (model weights under a restricted license) |
| Feeds | [feed-rs](https://crates.io/crates/feed-rs) | 3.0.0 |  |  |
| Lexical search | SQLite [FTS5](https://www.sqlite.org/fts5.html), `porter unicode61`, `bm25()` with column weights | built-in | One file, one transaction, `snippet()` for highlights | tantivy 0.26.2: separate index files outside the transaction |
| Vector search | [sqlite-vec](https://github.com/asg017/sqlite-vec) `vec0` | crate 0.1.9 | Same file as provenance; brute-force is fine at < 500k chunks | LanceDB, Qdrant: a second storage engine |
| Hybrid fusion | Reciprocal rank fusion in SQL, k = 60, [Garcia's recipe](https://alexgarcia.xyz/blog/2024/sqlite-vec-hybrid-search/index.html) |  |  |  |
| Embeddings | [fastembed](https://crates.io/crates/fastembed) with bge-small-en-v1.5 (384-dim, MIT) | 7.1.1 | Fast, small vector table; model name + dim stored in DB for re-embedding | Qwen3-Embedding-0.6B: test on the 50-query set in Phase 2; switch if it wins |
| Reranker | None in MVP |  | Add bge-reranker-v2-m3 or mxbai-rerank-base only if hybrid fails the eval |  |
| LLM | Existing local backend (llama.cpp / Ollama / LM Studio) through an OpenAI-compatible loopback provider, reusing the CompassWorth `local_backend` pattern |  | Model-agnostic; keyless; loopback-only | Hard-wiring one provider |
| DB access | [rusqlite](https://crates.io/crates/rusqlite) | 0.40.2 | Already used |  |
| UI | Tauri 2 + React/TypeScript, plain CSS |  | Same as CompassWorth |  |

Fallback if sqlite-vec breaks (it is pre-v1 with about 159 open issues): keep vectors as BLOBs and brute-force cosine in Rust. At MVP scale that is a few milliseconds.

Risk to retire in Phase 1: whether sqlite-vec loads into rusqlite 0.40 built with SQLCipher on Windows. Spike it before anything depends on it.

## Crawler and ingestion spec

The crawler is an allowlist walker, not a web crawler: it visits only URLs under registered source prefixes, one request per second per host, and records every outcome including failures.

### Discovery

- Seeds per source: `sitemap.xml` (preferred), RSS feed, or a hand list of start URLs. No off-site link following.
- On-site links are followed only if they match the source's `path_prefixes` and `allowed_extensions` (`.html`, no extension, `.pdf`).
- Frontier is a SQLite table (`fetch_queue`: url, source\_id, discovered\_from, priority, not\_before), not an in-memory queue, so a crash loses nothing.
- Cap per crawl run: 2,000 URLs per source, configurable; MVP corpus target is 300 documents, so most runs are small.

### Politeness

- [RFC 9309](https://www.rfc-editor.org/rfc/rfc9309.html): longest-match user-agent rule wins; robots.txt 4xx means allow all; 5xx means disallow all; cache robots.txt ≤ 24 h; parse at least 500 KiB. `Crawl-delay` is not in the RFC but is honoured when present.
- User-Agent: `FinKB/0.1 (+https://<project-url>; mailto:<contact>)`. SEC.gov requires a declared UA and caps at 10 req/s across all machines; we use 1 req/s everywhere.
- One in-flight request per host; global concurrency 4.
- Back off on 429 and 503 with `Retry-After` if present, else exponential from 60 s, max 3 retries, then `failed` with the reason.

### Fetch

- Conditional GET: send `If-None-Match` / `If-Modified-Since` from the last `fetch` row; a 304 writes a `fetch` row with `outcome = not_modified` and touches nothing else.
- Redirects: disabled in the client; each hop is validated against the SSRF rules and the allowlist, max 5 hops, every hop recorded.
- Limits: 20 MB body, 100 MB decompressed, 30 s connect + 60 s total, 200 PDF pages. Over any limit: `failed(reason = limit_exceeded)`.
- Raw bytes are stored on disk under `blobs/<sha256[0..2]>/<sha256>` with the content type; the `fetch` row holds the hash, status, headers, timing and the exact URL served.

### Change detection

1. HTTP validators (free).
2. SHA-256 of the raw body: identical bytes end the pipeline here.
3. SHA-256 of the normalized extracted text (whitespace collapsed, boilerplate stripped): identical text means a new `fetch` row only, no new `document_version`.
4. Different text creates a new `document_version`; every version is kept. Near-duplicate detection (MinHash) is for corroboration (section 9), never for deciding "unchanged": one changed number on an IRS page is a real change.

Recrawl cadence: start at 7 days; double after each unchanged fetch, cap 90 days; reset to 7 days on change; `lastmod` in the sitemap later than the last fetch forces a fetch. Sources tagged `hard_expiry` (IRS limits, SSA COLA, Medicare premiums) get a fixed calendar check in the weeks they are known to change (section 10).

### Extraction

| Content type | Path | Output |
| --- | --- | --- |
| HTML | dom\_smoothie (main content) → htmd (Markdown) | Markdown with headings preserved; title, author, dates, canonical URL, `rel=canonical` and OpenGraph metadata |
| PDF | pdfium-render text layer, page by page | Markdown-ish text with page markers; title from metadata or first heading; no OCR in MVP |
| PDF with tables (later) | Docling subprocess, no network, 120 s timeout, 2 GB memory cap; output treated as untrusted text | Markdown with tables |
| RSS | feed-rs | Item metadata only (posture `metadata_link`) |

Every extraction records `extractor_name`, `extractor_version` and a quality flag: `ok`, `thin` (< 200 words), `garbled` (non-dictionary ratio > 30%), `empty`. `thin`, `garbled` and `empty` stop the pipeline for that version and are visible in the ingestion inbox.

### Lifecycle and failure states

`discovered → queued → fetched → extracted → chunked → indexed → enriched → published`, plus `not_modified`, `failed(reason)`, `blocked_by_posture`, `removed`. Every transition writes `document_version.status` and a `pipeline_event` row (stage, outcome, reason, duration). Nothing is deleted on failure; `removed` is set only by the user or by a posture downgrade.

Reasons are an enum, not free text: `robots_disallow`, `http_4xx`, `http_5xx`, `timeout`, `limit_exceeded`, `ssrf_rejected`, `redirect_offsite`, `extract_empty`, `extract_garbled`, `parse_error`, `posture_blocked`.

### Not in MVP

JS rendering (government pages do not need it; Jina Reader or Firecrawl free tier is the manual fallback), OCR, YouTube, podcasts, link following off-site, distributed crawling.

## Data model

Ten tables carry the MVP; source material and AI output never share a column. Numbered SQL migrations, additive only, same rule as CompassWorth.

| Table | Key columns | Notes |
| --- | --- | --- |
| `source` | `id`, `name`, `publisher`, `host`, `path_prefixes` (JSON), `seed_urls` (JSON), `sitemap_url`, `feed_url`, `posture` (full\_text / metadata\_link / unclear / blocked), `posture_evidence_url`, `posture_checked_at`, `posture_review_by`, `tier` (1–4), `tier_reason`, `source_kind` (agency / regulator / academic / practitioner / vendor / community), `conflict_tags` (JSON), `crawl_delay_s`, `recrawl_min_days`, `enabled` | The registry. Hand-curated; `tier` and `posture` are human fields with a free-text reason |
| `fetch` | `id`, `source_id`, `url`, `final_url`, `requested_at`, `status_code`, `outcome` (ok / not\_modified / failed), `reason`, `etag`, `last_modified`, `content_type`, `body_sha256`, `blob_path`, `bytes`, `duration_ms`, `redirect_chain` (JSON), `user_agent` | One row per HTTP attempt, including 304s and failures |
| `document` | `id`, `source_id`, `canonical_url`, `alias_urls` (JSON), `title`, `author`, `published_at`, `updated_at`, `doc_kind` (page / pdf / paper / feed\_item), `subject_class` (hard\_expiry / annual\_data / market\_data / method\_evergreen / input\_dependent), `as_of_year`, `claim_type` (fact / analysis / opinion / mixed), `primary_or_secondary`, `primary_ref_document_id`, `status`, `removed_reason` | One per canonical URL. `claim_type`, `subject_class`, `as_of_year` are written only from `ai_annotation` rows marked `accepted` or by hand |
| `document_version` | `id`, `document_id`, `fetch_id`, `text_sha256`, `text_md` (nullable when posture ≠ full\_text), `word_count`, `extractor_name`, `extractor_version`, `quality_flag`, `status`, `created_at` | Every distinct text is kept; trigger rejects `text_md` unless the source posture is `full_text` |
| `chunk` | `id`, `document_version_id`, `ordinal`, `heading_path`, `page_no`, `char_start`, `char_end`, `text` | Heading-aware, 400–512 tokens, 15% overlap; `heading_path` and `page_no` become the citation |
| `chunk_fts` | FTS5 external-content table over `chunk.text` + `document.title`, `porter unicode61` | `bm25(chunk_fts, 2.0, 1.0)` weights title over body |
| `chunk_vec` | `vec0(chunk_id, embedding float[384])`, plus `embedding_model`, `embedding_dim` in a one-row `index_meta` table | Re-embed when `index_meta` changes; the fallback is a BLOB column |
| `ai_annotation` | `id`, `target_kind` (document / chunk / result), `target_id`, `kind` (summary / claim\_type / subject\_class / specificity / explanation / contradiction), `value` (JSON), `model`, `prompt_version`, `created_at`, `review_state` (proposed / accepted / rejected), `reviewed_by` | The only place AI output lives; shown with an AI marker; never copied into `document` text |
| `pipeline_event` | `id`, `document_version_id` or `fetch_id`, `stage`, `outcome`, `reason`, `duration_ms`, `at` | The observability log; answers "why was this ingested or rejected" |
| `eval_query`, `eval_label` | `query_id`, `text`, `category`; `query_id`, `chunk_id` or `document_id`, `grade` (0–3), `dimension_labels` (JSON), `labeler`, `labeled_at` | Section 12 |

Identity rules, borrowed from CompassWorth's DATA-18 lesson: transport identity (`fetch.url`) ≠ content identity (`document.canonical_url`) ≠ user curation (`source.tier`, `ai_annotation.review_state`). None is collapsed into another. `rel=canonical` and redirects set `canonical_url`; aliases are kept so a later fetch of an alias maps to the same document.

Invariants, each with a test:

- No `chunk` row from document text whose source posture is not `full_text`; a `metadata_link` document has exactly one `surrogate` row and its chunks derive only from that surrogate.
- Every `document_version` has exactly one `fetch` and that fetch has `outcome = ok`.
- Every `chunk_fts` and `chunk_vec` row has a live `chunk`; rebuild commands exist for both.
- `document.claim_type` is NULL until an accepted `ai_annotation` or a human sets it.

File layout: `%APPDATA%\com.finkb.app\finkb.sqlite` plus `blobs/`. Backups copy both. SQLCipher is a later toggle, not an MVP dependency; the public corpus is not secret.

Additions for governance, experts and formats:

| Table or column | Purpose |
| --- | --- |
| `source.serves_questions`, `source.topic_terms`, `source.max_documents`, `source.intake_path` (crawl / rss / clipper / youtube\_api / podcast\_rss) | Admission and budget rules |
| `document.format` (read / watch / listen), `document.duration_s`, `document.media_id`, `document.api_data_fetched_at` | Format field; YouTube 30-day refresh clock |
| `surrogate` (`document_id`, `summary_md`, `claims` JSON, `quotes` JSON, `chapters` JSON, `built_from` (rss / api / clipper), `review_state`) | Indexed stand-in for metadata-only documents; chunks for such documents point here |
| `document.status` adds `dormant`; `document.last_surfaced_at` | Retention and decay |
| `settings` (`max_documents`, `max_blob_bytes`) | Corpus budgets |

## Search and ranking

Relevance decides order; the other dimensions act as gates and labels, never as weights in a hidden sum. Every result card shows each dimension with a one-line reason.

### Query pipeline

1. **Intent.** Classify the question as `fact_lookup`, `analysis`, or `mixed` with a small local prompt; extract the topic tag (retirement\_savings, social\_security, withdrawal, tax\_accounts, housing, college, emergency\_fund, ltc, allocation). Cached per query text.
2. **Retrieve.** FTS5 `bm25()` top 100 and `chunk_vec` KNN top 100 on the query embedding.
3. **Fuse.** Reciprocal rank fusion in SQL, `1 / (60 + rank)` per list ([Cormack et al. 2009](http://cormack.uwaterloo.ca/cormacksigir09-rrf.pdf) found k = 60 near-optimal and not sensitive). Collapse to one row per document, keeping the best chunk as the snippet.
4. **Gate.** Apply hard rules by intent and source fields (table below). Gated-out results are listed under "hidden because" with the rule, so nothing disappears silently.
5. **Annotate.** Attach the pre-computed dimensions from `source`, `document` and accepted `ai_annotation` rows. Nothing runs an LLM at query time in MVP except step 1.
6. **Order.** Fused rank within gate class: official primary sources first for `fact_lookup`; otherwise fused rank with stale results demoted below fresh ones of the same source tier.
7. **Explain.** Each dimension renders a label and a reason string from the stored fields. The reason is a template filled from data ("IRS, official source for contribution limits"), not generated prose.

No rerank, no generated answer in MVP. A cross-encoder reranker and claim-level cited answers (ALCE-style entailment checks, section 11) are Phase 3, gated on the eval showing hybrid alone misses the target.

### Gate rules

| Rule | When | Effect |
| --- | --- | --- |
| Official-first | `fact_lookup` and a tier-1 agency document matches | Pin the agency document at rank 1 |
| Opinion filter | `fact_lookup` | Hide `claim_type = opinion`, list under "hidden" |
| Stale demotion | `subject_class = hard_expiry` and `as_of_year` < current rule year | Demote below all current results; label "Stale: cites {as\_of\_year} figure" |
| Pending flag | Topic has a known upcoming change (e.g. SS COLA before mid-October) | Label "Pending: 2027 figure not yet announced" on all results |
| Posture | `metadata_link` sources | Shown with title, our summary and link, no snippet |
| Corroboration collapse | Several results share one `primary_ref_document_id` | Show the primary once with "+ N articles that restate it" |

Two more rules from the expert and format tracks: **Expert coverage** (analysis intent: if the top 5 has no Tier 2–3 result, promote the best-ranked one into position 5 and label it "Expert view"); **Format mix** (when the user has not filtered, and a watch or listen item ranks in the top 20, surface the best one in the top 5 with its format badge). Both are visible rules, listed in "why this rank".

### Result card

| Field | Source of truth | Example |
| --- | --- | --- |
| Title, publisher, author, dates | `document` | SSA, Retirement planner: delayed retirement credits, updated 2026-01 |
| Match snippet | FTS5 `snippet()` on the best chunk | "...credits increase your benefit by 8% per year of delay up to age 70..." |
| Type | `claim_type` | Rule / Research / View |
| Authority | `source.tier`, `tier_reason` | Tier 1: official source for benefit rules |
| Evidence | `ai_annotation(kind = method)` when accepted | Statutory formula; no study |
| Freshness | `subject_class`, `as_of_year` vs current | Current (rule unchanged since 2023) |
| Independence | `primary_or_secondary`, `primary_ref_document_id` | Original source |
| Conflicts of interest | `source.conflict_tags` | None / Sells annuities / Affiliate links |
| Contradicts | Phase 3 | — |
| Why this rank | Template over gate and rank | "Pinned: official source for a fact question" |
| Open original | `canonical_url` + page/heading anchor |  |

All dimension values and reasons are plain data in the result payload, so the CLI, the UI and the eval runner see the same thing. The Database Test applies: no internal enum names in copy (`hard_expiry` renders as "changes every year").

## Source quality model

Quality is a set of named fields, each with a stated method and a visible reason. There is no composite score. Hard gates and relevance order results; the fields explain them.

### Dimensions

| # | Dimension | Signals | Method | Reason shown |
| --- | --- | --- | --- | --- |
| 1 | Claim type (fact / analysis / opinion) | Number or rule from a statute or agency = fact; method + data + stated assumptions = analysis; advice without method = opinion | Rule: tier-1 agency implies fact. Else LLM per document, human-accepted | "Rule from agency page" / "Method stated: Monte Carlo 1926–2023" |
| 2 | Primary vs secondary | Original publication (statute, notice, paper, dataset) vs restatement; outbound citation to a primary | Rule on `source_kind`; LLM proposes `primary_ref`; human accepts | "Original" / "Restates: \[SSA planner\]" |
| 3 | Authority | Official source for the topic; institutional publisher; named author with credentials. [Google's rater guidelines](https://static.googleusercontent.com/media/guidelines.raterhub.com/en//searchqualityevaluatorguidelines.pdf) treat finance as YMYL and "advice on how to invest for retirement" as expert-only | Human-curated `source.tier` 1–4 with `tier_reason` | "Tier 1: IRS, official source for limits" |
| 4 | Evidence quality | Peer review or working paper; data, sample, period and assumptions disclosed | Human tier by source kind; LLM extracts method/data/assumptions into an annotation | "NBER working paper; data 1926–2012; assumes tax-free account" |
| 5 | Conflict of interest | Sells the product discussed; affiliate links; sponsored label; funder. [NewsGuard](https://www.newsguardtech.com/ratings/rating-process-criteria/) scores ad labeling and ownership disclosure; Wikipedia treats sponsored content as non-independent | Human `conflict_tags` per source; rule detects affiliate-link patterns per page | "Sells annuities; affiliate links present" |
| 6 | Independence | Adds evidence or only repeats another source | Rule: citation lineage clustering (section 9) | "1 of 6 articles on this result adds no new evidence" |
| 7 | Freshness by subject | `subject_class` and `as_of_year` vs the current authoritative value | Rule table below | "Stale: cites 2025 limit; 2026 limit is $24,500" |
| 8 | Specificity | Stated applicability: country, horizon, account type, filing status | LLM rubric 0–3, Phase 3 | "Applies to: US, 30-year horizon, 50/50 portfolio" |
| 9 | Citation support | Cited passages entail the claim ([ALCE](https://arxiv.org/abs/2305.14627) citation recall and precision) | NLI check, Phase 3, only on generated answers | "4/5 claims backed by cited text" |
| 10 | Contradiction | Another result asserts an incompatible figure for the same normalized claim | Claim-level comparison, Phase 3 | "Disagrees with Morningstar 3.9% (90% success)" |
| 11 | Reputation | Corrections policy, regulator actions, error history | Human, rarely changed; folded into tier | Tier label |
| 12 | Relevance | Query-passage match | Hybrid retrieval | Snippet, not a number |

MVP ships dimensions 1–7, 11 and 12. 8–10 wait for a labeled set.

### Source tiers

| Tier | Definition | Examples |
| --- | --- | --- |
| 1 | Official source for the rule or data | IRS, SSA, CMS, CFPB, SEC, Federal Reserve, BLS, Treasury |
| 2 | Peer-reviewed or institutional research with disclosed method | NBER, CRR Boston College, EBRI, Journal of Financial Planning, Fed research |
| 3 | Expert practitioner with named author, method shown, conflicts disclosed | Kitces, Pfau, Bogleheads wiki, Morningstar research |
| 4 | Vendor education and general media | Fidelity, Vanguard, Schwab guides, Investopedia |

A tier is a human decision with a written reason and a date. The Wikipedia [perennial-sources](https://en.wikipedia.org/wiki/Wikipedia:Reliable_sources/Perennial_sources) statuses (reliable / no consensus / unreliable / deprecated) are the template for the review process; the list itself is not reused.

### Freshness by subject

| Subject class | Topics | Authority and cadence | Rule |
| --- | --- | --- | --- |
| hard\_expiry | 401(k)/IRA limits, catch-ups, brackets, standard deduction | IRS, announced each November for the next year ([2026 release](https://www.irs.gov/newsroom/401k-limit-increases-to-24500-for-2026-ira-limit-increases-to-7500)) | Stale if `as_of_year` < current rule year |
| hard\_expiry | SS COLA, wage base, earnings test | SSA, mid-October | Stale as above; "pending" label from Oct 1 until announced |
| hard\_expiry | Medicare Part B premium, IRMAA | CMS fact sheet, November | Stale as above |
| hard\_expiry | SECURE 2.0 effective dates | IRS final regs and notices; consumer pages and regs have disagreed | Regression test case for contradiction |
| annual\_data | I-bond rates (May 1, Nov 1, [TreasuryDirect](https://www.treasurydirect.gov/savings-bonds/i-bonds/i-bonds-interest-rates/)), LTC cost surveys, Fed SCF (triennial; [2025 SCF released 2026-10-09](https://www.federalreserve.gov/econres/scfindex.htm)) | Per source | Label with data vintage; stale after the next release |
| market\_data | Mortgage rates ([Freddie Mac PMMS](https://www.freddiemac.com/pmms), weekly), yields | Weekly or daily | Never cached as fact; shown as "as of {date}" |
| method\_evergreen | 4% rule and SWR research, SS claiming analysis, asset allocation theory, behavioral finance | Bengen 1994, Trinity 1998, NBER papers | Never stale; headline numbers from the same papers carry their own `as_of_year` |
| input\_dependent | Mortgage payoff vs invest, rent vs buy | Depends on rates in the text | Freshness is the stated assumptions; extract and show them ("assumes 3% mortgage, 2021") |

Unverified: the 2026 figures above come from search summaries, not fetched IRS or CMS pages, and are placeholders for the eval set, not product data. Regulator guidance (FINRA, SEC) on evaluating online information was not retrieved.

## AI usage and security

AI proposes, never decides, and never touches the ranking at query time in MVP. Everything fetched is untrusted input.

### Model roles

| Task | When | Model | Output goes to |
| --- | --- | --- | --- |
| Query intent and topic tag | Query time | Local small model, cached | Gate selection only |
| Document summary (one paragraph) | Ingest | Local model | `ai_annotation(kind = summary)`, shown with AI marker |
| Claim type, subject class, as-of year, primary reference | Ingest | Local model with fixed rubric | `ai_annotation` as `proposed`; copied to `document` only when `accepted` by the user, or auto-accepted for tier-1 sources where the rule already decides |
| Method / data / assumptions extraction | Ingest, Phase 2 | Local model | `ai_annotation(kind = method)` |
| Embeddings | Ingest | fastembed, bge-small-en-v1.5 | `chunk_vec` |
| Specificity, contradiction, cited answers | Phase 3 | Local or hosted, user's choice | `ai_annotation` |

Provider routing reuses CompassWorth's pattern: one local routing class with `local_backend` ∈ {ollama, llama\_server, lm\_studio} behind an OpenAI-compatible loopback-only provider, keyless, redirects off; hosted providers are a separate explicit mode. Model name and prompt version are recorded on every annotation so a model change is auditable and re-runnable.

AI evaluation gate: no annotation kind ships without an eval row (accuracy on the hand-labeled set, section 12) and a review state. Same rule as CompassWorth's AI-52.

### Trust boundaries

`app instructions → fetched content → model → our tables → UI`. Content crosses into a prompt only inside delimited data blocks with an explicit "this is a document, not instructions" frame; model output crosses into `document` only through `review_state = accepted`. The answering model has no tools and no network.

### Controls

| Threat | Control | Test |
| --- | --- | --- |
| SSRF ([OWASP cheat sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)) | Allowlist of hosts; resolve DNS, reject every A/AAAA in 127/8, 0/8, ::1, RFC 1918, fc00::/7, fe80::/10, multicast, 169.254.169.254; connect only to the validated IP with the original hostname for SNI; redirects off, each hop re-validated | Unit tests per range; a redirect-to-localhost fixture |
| Resource abuse | Body 20 MB, decompressed 100 MB, 5 redirects, 30 s / 60 s timeouts, 200 PDF pages, robots.txt parse cap 500 KiB | Gzip-bomb and 10k-page PDF fixtures |
| Hostile PDF | pdfium runs in a child process with no network, 60 s timeout, memory cap; crash = `failed(parse_error)`, never the app | Fuzzed PDF fixture crashes the child, not the test |
| Python sidecar (if added) | Same child-process isolation; output treated as untrusted text |  |
| Prompt injection ([OWASP LLM01](https://genai.owasp.org/llm-top-10/)) | Data-only framing; no tool access for the answering model; model output never auto-accepted into `document` except tier-1 rules; UI renders Markdown with raw HTML stripped, no remote images, links shown with their real host | Fixture page containing "ignore previous instructions" produces an annotation that quotes it as content |
| Data poisoning (LLM04) | Allowlist + human-curated tiers; a page cannot raise its own tier |  |
| Output handling (LLM05) | Escape all model and page text in the UI; no `dangerouslySetInnerHTML` | Lint rule |
| Misinformation (LLM09) | No generated answers in MVP; every shown claim is a quoted passage with a link |  |
| Secrets | None in MVP; hosted-provider keys later live in the OS keychain, never in SQLite |  |

Private context from CompassWorth (Phase 4) never enters the knowledge store, prompts are built per request from a minimal struct, and nothing about the user is written to `ai_annotation`.

## Evaluation

The eval set exists before the ranker does. Day one of Phase 1 is labeling, not coding the search.

### Query set

- 10 seed questions plus 2 paraphrases each (30).
- 20 fact lookups with one right answer ("2026 IRA limit", "SS full retirement age for 1960").
- 10 stale traps ("2025 401(k) limit" asked in 2026; a 2027 COLA query before mid-October).
- 10 conflict-prone questions ("is 4% safe?", "Roth catch-up required in 2026?").
- Total about 70 at MVP, grown to 100+ over time. [Voorhees and Buckley](https://www.nist.gov/publications/effect-topic-set-size-retrieval-experiment-error) showed small topic sets overstate differences, so system comparisons use paired bootstrap, not point estimates.

### Labels

Graded 0–3 per (query, document): 0 irrelevant; 1 related, does not answer; 2 answers with caveats or not authoritative; 3 authoritative and direct. Binary labels per dimension where it applies: claim type correct, stale flag correct, primary reference correct. 20% double-labeled by a second pass a week later; agreement reported before any number is trusted. Labels live in `eval_label` with labeler and date.

### Seed sources per question

The starting pool to label, from the research pass. "Verified" means the page was opened; the rest need a manual open before they enter the registry.

| Question | Candidate sources | Type | Status |
| --- | --- | --- | --- |
| 1 How much to retire | [Fed SCF 2025](https://www.federalreserve.gov/econres/scfindex.htm); [CRR Boston College](https://crr.bc.edu/); Fidelity savings-multiple guideline; EBRI Retirement Confidence Survey | fact, analysis, vendor rule of thumb | SCF and CRR verified; Fidelity and EBRI metadata-only |
| 2 When to claim SS | SSA retirement planner, delayed credits; [NBER w7318](https://www.nber.org/papers/w7318) Coile et al. 1999; NBER w19370 Shoven & Slavov 2013; CRR 2026 Trustees update | fact, analysis | NBER verified; SSA fetch blocked, open manually |
| 3 Mortgage payoff vs invest | [Wharton, Roberts 2021](https://knowledge.wharton.upenn.edu/article/think-twice-paying-off-mortgage-early/); [CFPB owning a home](https://www.consumerfinance.gov/owning-a-home/) | analysis (input-dependent: 3% rates), fact | Verified; gap: no Kitces or academic piece found |
| 4 College savings | [IRS Pub 970](https://www.irs.gov/publications/p970); investor.gov 529 page; studentaid.gov FAFSA | fact | Pub 970 verified; others blocked |
| 5 Emergency fund | [CFPB emergency fund guide](https://www.consumerfinance.gov/an-essential-guide-to-building-an-emergency-fund/) (no month target given); [CFPB Your Money Your Goals](https://www.consumerfinance.gov/consumer-tools/educator-tools/your-money-your-goals/) | education | Verified; gap: no analysis with a stated target |
| 6 Roth vs traditional | [IRS Roth IRAs](https://www.irs.gov/retirement-plans/roth-iras); IRS 2026 limits release; IRS catch-up page | fact | Verified; gap: tax-rate-arbitrage analysis (Kitces, Pfau) |
| 7 Safe withdrawal rate | Bengen 1994 (FPA copyright); Trinity 1998 (no official URL); Pfau JFP 2013; Morningstar State of Retirement Income 2026 (3.9%); Bogleheads SWR wiki | analysis, community | All metadata-only; Bogleheads blocked |
| 8 Long-term care | [ACL, how much care will you need](https://acl.gov/ltc/basic-needs/how-much-care-will-you-need) (page dated 2020); Genworth/CareScout Cost of Care 2024; medicare.gov LTC pages | fact, survey (insurer conflict) | ACL verified |
| 9 Rent vs buy | CFPB owning a home; Zillow break-even (vendor model) | fact, vendor analysis | Gap: no academic or Fed source found |
| 10 Asset allocation | Vanguard Principles for Investing Success; investor.gov asset allocation | vendor analysis, education | Gap: nothing verified |

The gaps are the point: questions 3, 5, 7, 9 and 10 have no clean tier-1 or tier-2 source yet, which is what the product must surface honestly rather than paper over.

### Metrics

| Metric | Use | MVP target |
| --- | --- | --- |
| Precision@5 | What the first screen looks like | ≥ 0.6 mean, ≥ 0.4 minimum per seed question |
| nDCG@10 | System comparison across ablations | Reported, no target until baseline exists |
| MRR | Fact lookups | ≥ 0.7 |
| Stale-flag precision and recall | Freshness rule | ≥ 0.9 / ≥ 0.8 on the stale-trap set |
| Provenance correctness | Every shown URL, title, date resolves | 100% |
| Claim-type accuracy | Annotation gate | ≥ 0.85 vs human labels before auto-accept on any tier |

Ablations run every time: BM25 only, vector only, RRF, RRF + gates. Later: + reranker, + LLM judge. Tooling: [ranx](https://github.com/AmenRa/ranx) (metrics, RRF, paired tests) or [ir\_measures](https://ir-measur.es/en/latest/); the eval runner exports TREC-format runs so either works.

### Regression suite

Fixed hazard queries that must keep passing: SECURE 2.0 Roth catch-up date (IRS consumer page vs final regs disagree); a 2027 COLA query before the SSA announcement; the 2021 Wharton mortgage analysis at 7% rates; Morningstar 3.9% vs the 4% rule; a page containing "ignore previous instructions".

### Acceptance gate

MVP closes when the metrics table passes on the labeled set, provenance is 100%, and a second labeling pass agrees on ≥ 80% of grades. Targets are revised in the doc with a dated note when the first baseline shows they were wrong.

## Phased build plan

Five phases, each with a scripted gate. Phase 2 is the MVP. Labels come before the ranker, and the crawler ships before any AI.

&#91;embedded content: build roadmap · 5 phases, 5 gates\]

Each phase closes only when its gate passes on the eval runner; nothing in a later phase starts on an unmet gate.

| Phase | Tier (per `tier.mjs` convention) | Delivers | Gate | Effort (estimate) |
| --- | --- | --- | --- | --- |
| 0 Registry and labels | 1 | New repo `finkb` as a Cargo workspace (`finkb-core`, `finkb-cli`, `src-tauri` stub); migrations 001–003 (`source`, `eval_query`, `eval_label`); seed registry with posture evidence URLs; 70 labeled queries; `finkb eval` command that scores a TREC run file | Two labeling passes agree ≥ 80%; every registry row has a posture evidence URL or `unclear` with a review date | 1–2 weeks |
| 1 Fetch and extract | 2 (untrusted input, blob storage) | `fetch_queue`, `fetch`, `document`, `document_version`, `pipeline_event`; robots + SSRF + limits; conditional GET and text-hash change detection; dom\_smoothie + htmd; pdfium child process; `finkb crawl <source>`; sqlite-vec load spike on Windows + SQLCipher | 300 documents from 12 sources; zero silent drops (every URL ends in a terminal state with a reason); SSRF and limit fixtures pass | 2–3 weeks |
| 2 Index and search (MVP) | 1 | `chunk`, `chunk_fts`, `chunk_vec`, `index_meta`; fastembed; RRF in SQL; gate rules; result payload with dimensions and reasons; `finkb search`; minimal Tauri UI (search box, result cards, source viewer with highlighted chunk); ablation report | P@5 ≥ 0.6 mean and ≥ 0.4 min; MRR ≥ 0.7; provenance 100%; stale-flag P/R ≥ 0.9/0.8 | 2–3 weeks |
| 3 Enrich and explain | 2 (AI gate) | `ai_annotation` with local provider; summary, claim type, subject class, as-of year, primary ref; review inbox UI; auto-accept for tier-1 rules; freshness calendar; Obsidian one-way export; reranker ablation; prompt-injection fixtures | Claim-type accuracy ≥ 0.85 vs labels; injection fixture produces no instruction-following; every annotation kind has an eval row | 3 weeks |
| 4 Expand sources | 2 (per-source legal review) | RSS ingestion for practitioner sites; arXiv q-fin via API with per-paper license; Zotero import for papers; CompassWorth context interface (read-only struct, no DB coupling); contradiction detection; cited-answer generation with entailment check | Per-source terms verified and recorded; CompassWorth interface passes a data-minimization review | Open-ended |

Efforts are estimates for one person using the existing CompassWorth tooling and are not commitments.

Governance and expert work by phase: Phase 0 adds `serves_questions`, budgets and the expert registry with verified feeds and channel IDs; Phase 1 adds RSS intake, the surrogate table and budget enforcement; Phase 2 adds the YouTube Data API intake with the 28-day refresh job, format chips, and the expert-coverage and format-mix gates; Phase 3 adds the clipper, podcast RSS, dormancy and the monthly corpus report. Phase 2's gate gains: at least one Tier 2–3 result in the top 5 for every analysis-intent seed question.

Workflow mapping: Phase 0 and 2 run through `/quick`; Phase 1, 3 and 4 run through `/phase` with one plan-adversary pass each, because they touch untrusted input, AI decisions or legal posture. The repo gets its own `CLAUDE.md`, `scripts/gates/tier.mjs` copy with `fetch/`, `extract/`, `ai/` and `migrations/` as Tier-2 paths, and the same bounded STATE/ROADMAP files.

## Learning experience

The same answer is offered at six depths and in whatever format the source exists in. Formats are doors into the same content, not a diagnosis of the user. The evidence below is labeled: **verified** means the research agent read the source page; **secondary** means a summary or third-party digest; **inferred** means our design choice from adjacent evidence.

### What the evidence supports

| Rule | Evidence | What we build |
| --- | --- | --- |
| Do not route by "learning style" | Pashler et al. 2008 found almost no properly designed matching studies, and those were negative; a 2015 follow-up found dual coding helped and learning styles did not (secondary, [Wikipedia digest](https://en.wikipedia.org/wiki/Learning_styles)) | No learner-type quiz. Read / Watch / Listen / Try-it chips on every answer; last choice remembered per topic |
| Pair words with a picture or concrete example; do not read on-screen text aloud | Mayer multimedia, modality and redundancy effects, medians roughly d 1.7 / 1.0 / 0.7 (secondary, [third-party slides](https://open.umich.edu/sites/default/files/downloads/um-multimedialearning-final.pdf)) | Every rung 2–4 answer has one chart or worked table; audio rungs link the chart, never narrate it |
| Cut decoration; cue the decision-relevant phrase | Mayer coherence about 0.8, signaling about 0.5 (secondary, same deck) | One claim, one source, one "why it ranked" per card; one bolded phrase; no stock art |
| Segment, and pre-train terms before the explanation | Mayer segmenting about 1.0, pre-training about 0.85 (secondary); YouTube chapters need 3+ timestamps from 00:00, each 10 s or more (verified, [YouTube help](https://support.google.com/youtube/answer/9884579)) | "Terms you'll see" strip with one-line definitions above rung 3; video results deep-link to the chapter |
| Adapt depth to prior knowledge; novice aids hurt experts | Expertise reversal (verified, [Kalyuga 2007](https://www.uky.edu/~gmswan3/EDC608/Kalyuga2007_Article_ExpertiseReversalEffectAndItsI.pdf)) | One optional calibration question per topic ("Know the 4% rule already?"); experts collapse worked examples and glossary strips; control stays visible |
| Worked example before asking the user to compute | Cognitive load theory worked-example effect (secondary) | "How much do I need?" shows a filled-in household with round numbers first, then "Use my numbers" |
| Rules of thumb for less sophisticated readers, plus where the rule breaks | RCT: rule-of-thumb training beat accounting training, largest gains for lower-skilled owners (verified abstract, [Drexler, Fischer, Schoar](https://dspace.mit.edu/handle/1721.1/88086)); population differs from ours | Rung 2 is always heuristic plus one failure case |
| Help at the moment of decision; plan for decay | Fernandes et al. 2014: tiny, decaying effects; Kaiser et al. 2022: larger durable effects (both secondary; they disagree) | Explainers appear when the question is asked, not as a course; optional "revisit in 6 months" reminder on timing decisions |
| Retrieval practice and spacing work; rereading and highlighting do not | Dunlosky et al. (verified, [APS release](https://www.psychologicalscience.org/news/releases/which-study-strategies-make-the-grade.html)) | Optional, skippable two-question check after a worked example; no highlight or re-read features |
| Plain language, answer first | Federal plain-language guide (verified, [digital.gov](https://digital.gov/guides/plain-language/writing)); SEC Plain English Handbook (verified, [sec.gov](https://www.sec.gov/pdf/handbook.pdf)); Mayer personalization about 1.1 (secondary) | Generated one-liners and surrogate summaries pass a readability check at grade 8 or below; active voice, "you" |
| Lower-secondary summary where text is harder | WCAG 3.1.5, Level AAA (verified, [W3C](https://www.w3.org/WAI/WCAG22/Understanding/reading-level.html)) | Rungs 1–3 target grade 8 or below; rungs 4–6 may be harder but carry a plain summary |
| Natural frequencies and icon arrays for probability | Gigerenzer 2007, Hoffrage 2015 (secondary; fetch blocked) | See number rules below |
| Citations raise trust even when wrong, so show the supporting passage | 303-person experiment (verified abstract, [AAAI](https://ojs.aaai.org/index.php/AAAI/article/view/34550)); generative search audit: 51.5% of sentences fully supported (verified abstract, [arXiv](https://arxiv.org/abs/2304.09848)) | Every claim links to the exact passage or timestamp; source type badge; never generate a claim the source does not state (already an invariant) |
| Do not project false certainty; surface disagreement | Tow Center 2025 and Oumi/NYT analyses (secondary, news reports) | "Experts disagree" block with two or three positions and sources when sources conflict; plain-word confidence (well established / debated / depends on you), never a percentage (inferred) |
| Interactive tools can move behavior, evidence thin | Goda et al. calculator RCT, Chilean and US video studies (all secondary); none showed durable behavior change | "Try it" rung for retirement need and Roth-vs-traditional; results as ranges; static explanation always available |

### Learning ladder (per seed question)

| Rung | Content | Formats | Grade target |
| --- | --- | --- | --- |
| 1. One-line answer | Plain sentence with the "depends on X" caveat in the same line | Text; spoken version as accessibility option | ≤ 8 |
| 2. Rule of thumb | Heuristic plus the case where it breaks | Text, small chart | ≤ 8 |
| 3. Plain explanation | Terms strip, then 3–5 short paragraphs or a chaptered video | Text, short video, audio | ≤ 8 |
| 4. Worked example | Concrete household, numbers filled in, collapsible for experts | Text with chart; "Try it with my numbers" | any, with plain summary |
| 5. Expert deep dive | Papers, practitioner debate, long-form video or podcast with chapter links; "Experts disagree" where they do | Video, podcast, text | any, with plain summary |
| 6. Primary sources | Government pages and original studies, supporting passage highlighted | Text | any |

Format fit (inferred): video and audio serve rungs 3 and 5 (narrative, debate); text, tables and interactive serve rungs 2, 4 and 6 (look-up, compute, verify). Rung 1 is text first. Rungs 1, 2 and 4 are generated from Tier 1 facts and labeled as ours; rungs 3, 5 and 6 are retrieved documents. The format-mix gate in section 8 and the Read / Watch / Listen chips decide what shows in the top 5; the ladder decides how a single answer is laid out.

### Presenting numbers and probabilities

- Natural frequencies: "out of 100 people with this plan, about 10 run out of money by 90", not "10% failure probability".
- Every percentage carries its base: "4% of your savings in the first year", never just "the 4% rule".
- Absolute over relative change; a "25% lower" claim is shown as the per-1,000 difference.
- Risk as a 100-icon array with a plain-text sentence beside it.
- Ranges and scenarios, never one point estimate; return, inflation and longevity assumptions printed inline.
- Money per month and per year, in today's dollars, labeled as such; sensible rounding; aligned decimals; one unit per table.
- Every chart and table has a one-sentence text equivalent stating the takeaway.
- Baseline: only 29% of US adults answered all three Big Three literacy questions in 2021 (secondary, [GFLEC](https://gflec.org/?p=2706)). Design rungs 1–3 for the other 71%.

### Accessibility checklist (WCAG 2.2; all verified at w3.org unless marked)

- [ ] Captions on all embedded video ([1.2.2](https://www.w3.org/WAI/WCAG22/Understanding/captions-prerecorded.html), Level A); linked-out video and audio results show a captions / transcript badge from the surrogate (inferred).
- [ ] Transcript for audio-only content ([1.2.1](https://www.w3.org/WAI/WCAG22/Understanding/audio-only-and-video-only-prerecorded.html), A); podcast `podcast:transcript` links surfaced when present.
- [ ] Lower-secondary summary wherever the text is harder ([3.1.5](https://www.w3.org/WAI/WCAG22/Understanding/reading-level.html), AAA, stretch goal).
- [ ] Reflow at 320 CSS px without two-dimensional scrolling ([1.4.10](https://www.w3.org/WAI/WCAG22/Understanding/reflow.html), AA); data tables excepted but cells reflow.
- [ ] Headings, lists and table markup exposed to assistive tech ([1.3.1](https://www.w3.org/WAI/WCAG22/Understanding/info-and-relationships.html), A); tables use `th`, `scope`, caption ([tables tutorial](https://www.w3.org/WAI/tutorials/tables/)).
- [ ] Charts never rely on color alone and carry a text equivalent (inferred from 1.3.1).
- [ ] Older users: strong contrast, resizable text, large targets, no background audio, consistent navigation ([W3C older users](https://www.w3.org/WAI/older-users/)).
- [ ] User-adjustable font size, spacing and line length instead of a single "dyslexia font" (BDA guide not fetched; inferred).
- [ ] Test rounds with at least two older users and one screen-reader user before the Phase 3 gate.

### Myths we will not build on

- Matching content to a learning style (VAK). Not supported.
- "Video is always better than text." Mixed results, gains fade, no behavior change shown.
- "More explanation is better for everyone." Expertise reversal.
- "Citations mean it is right." Citations raise trust even when irrelevant.
- "Financial education sticks." Disputed between the two major meta-analyses.
- Highlighting, summarizing and rereading as study aids. Rated low utility.
- Readability score as proof of clarity. A proxy; test with readers (inferred).
- Percentages as plain language. Relative risk without a base rate misleads.

### What this adds to the build

- Data model: `answer_rung` rows per seed question (rung 1–4 text, generated, with `source_ids`, `readability_grade`, `review_state`); `surrogate.captions_available` and `surrogate.transcript_url` for watch and listen items; `topic_calibration` (topic, user answer, timestamp).
- Gates: every generated rung 1–3 text must pass readability grade ≤ 8 and cite at least one Tier 1 passage; the eval runner fails on a rung that cites a claim its source does not contain. Any seed question whose sources conflict must render an "Experts disagree" block, checked by the labeled set.
- Phases: Phase 1 ships rungs 1, 3 and 6 (text only); Phase 2 adds rungs 2 and 5, format chips and the calibration question; Phase 3 adds rung 4 "Try it", the two-question check, the reminder and the accessibility test round. The Phase 3 gate adds: WCAG A and AA items above pass an axe-core run plus a manual screen-reader pass.

### Not researched (follow-ups, not blockers)

Primary texts for Pashler 2008, Fernandes 2014, Kaiser 2022, Galesic 2009 and the Goda RCT were blocked on fetch. CFPB content-design guidance, FINRA Foundation, Khan Academy, Brilliant, Duolingo, Investopedia, Bogleheads and NerdWallet presentation patterns were not reviewed. No evidence was found for reading-level toggles or "show me / tell me" toggles; those stay out until a user test asks for them.

## Open questions and risks

Four decisions need an owner answer before Phase 0; the rest are risks with a planned retirement point.

### Decisions needed

- [ ] **Repo and name.** New repo `finkb` (recommended) or build inside `finkb-bench`? Product name is still "Financial Knowledge Companion"; the CLI and DB file need a short name now.
- [ ] **Python sidecar policy.** Allowed from day one for Docling, or Rust-only until a PDF failure forces it? Recommendation: Rust-only through Phase 2; decide on real IRS and SSA PDFs.
- [ ] **Hosted models.** Local-only for all AI in Phase 3, or allow a hosted provider for annotation quality? Local-only keeps the privacy story simple; hosted likely improves claim-type accuracy.
- [ ] **Posture for `unclear` sources.** Treat as `metadata_link` silently (recommended), or exclude from the registry until verified?

### Risks

| Risk | Likelihood | Impact | Retire by |
| --- | --- | --- | --- |
| sqlite-vec does not load into rusqlite 0.40 + SQLCipher on Windows | Medium | Low (BLOB fallback is trivial at this scale) | Phase 1 spike |
| Rust PDF text extraction is poor on column and table PDFs | Medium | Medium (IRS publications are the core corpus) | Phase 1: run 20 real PDFs through pdfium and inspect |
| ssa.gov, investor.gov, medicare.gov terms never verified (fetcher got 403) | High | Low: posture stays `metadata_link` until a human opens the page | Phase 0 manual check |
| Eval set too small to detect ranking changes | High | Medium: decisions on reranker and embeddings rest on it | Grow to 100+ queries by Phase 3; paired bootstrap only |
| Claim-type and freshness annotations drift with model or prompt changes | Medium | Medium | Model + prompt version on every row; re-run eval on change |
| Scope creep into synthesis, contradictions, calculators | High | High | Non-goals list in section 2; Phase 4 only after Phase 3 gate |
| Advice boundary blurs once CompassWorth context arrives | Medium | High | Phase 4 design review: results stay sources, never recommendations |
| texting\_robots last released 2023 | Low | Low | Vendor the crate if an issue appears |
| Treasury terms conflict with 17 USC §105 | Low | Low | Cautious reading: metadata only |

### Explicitly deferred

YouTube and podcast transcripts (ToS), OCR, JS rendering, multi-user, sync, hosted deployment, encryption at rest, research synthesis.

## Sources consulted

Pages opened during the research pass (2026-10-10). Items marked search-only were not opened and are flagged in the sections that use them.

**Tools and libraries:** [Karakeep](https://github.com/karakeep-app/karakeep) · [ArchiveBox](https://github.com/ArchiveBox/ArchiveBox) · [Readeck](https://codeberg.org/readeck/readeck) · [Linkwarden](https://github.com/linkwarden/linkwarden) · [SingleFile](https://github.com/gildas-lormeau/SingleFile) · [Kotaemon](https://github.com/Cinnamon/kotaemon) · [AnythingLLM](https://github.com/Mintplex-Labs/anything-llm) · [RAGFlow](https://github.com/infiniflow/ragflow) · [Open WebUI RAG](https://docs.openwebui.com/features/chat-conversations/rag/) · [Onyx](https://github.com/onyx-dot-app/onyx) · [txtai](https://github.com/neuml/txtai) · [Haystack](https://github.com/deepset-ai/haystack) · [Crawl4AI](https://github.com/unclecode/crawl4ai) · [Trafilatura](https://github.com/adbar/trafilatura) and its [evaluation](https://trafilatura.readthedocs.io/en/latest/evaluation.html) · [Docling](https://github.com/docling-project/docling) · [marker](https://github.com/datalab-to/marker) · [spider](https://github.com/spider-rs/spider) · [dom\_smoothie](https://github.com/niklak/dom_smoothie) · [htmd](https://github.com/letmutex/htmd) · [texting\_robots](https://github.com/Smerity/texting_robots) · [yt-dlp](https://github.com/yt-dlp/yt-dlp) · crates.io API pages for reqwest, scraper, feed-rs, pdfium-render, pdf\_oxide, lopdf, pdf-extract, mupdf, rusqlite, tantivy, lancedb, simhash, probminhash, fastembed, sqlite-vec · [SQLite FTS5](https://www.sqlite.org/fts5.html) · [sqlite-vec](https://github.com/asg017/sqlite-vec), [vec0 docs](https://alexgarcia.xyz/sqlite-vec/features/vec0.html), [hybrid search recipe](https://alexgarcia.xyz/blog/2024/sqlite-vec-hybrid-search/index.html) · [Qdrant](https://github.com/qdrant/qdrant) · HF cards: [bge-small-en-v1.5](https://huggingface.co/BAAI/bge-small-en-v1.5), [nomic-embed-text-v1.5](https://huggingface.co/nomic-ai/nomic-embed-text-v1.5), [Qwen3-Embedding-0.6B](https://huggingface.co/Qwen/Qwen3-Embedding-0.6B), [bge-reranker-v2-m3](https://huggingface.co/BAAI/bge-reranker-v2-m3), [Qwen3-Reranker-0.6B](https://huggingface.co/Qwen/Qwen3-Reranker-0.6B), [mxbai-rerank-base-v1](https://huggingface.co/mixedbread-ai/mxbai-rerank-base-v1) · [ranx](https://github.com/AmenRa/ranx) · [ir\_measures](https://ir-measur.es/en/latest/)

**Hosted services:** [Firecrawl pricing](https://www.firecrawl.dev/pricing) · [Jina Reader](https://jina.ai/reader/) · [Exa pricing](https://exa.ai/pricing) · [Tavily pricing](https://www.tavily.com/pricing) · [Readwise pricing](https://readwise.io/pricing) and [Reader API](https://readwise.io/reader_api) · [Kagi pricing](https://www.kagi.com/pricing) · [Elicit pricing](https://elicit.com/pricing) · [Notion pricing](https://www.notion.com/pricing)

**Obsidian, NotebookLM, Zotero:** [Obsidian license](https://obsidian.md/license), [pricing](https://obsidian.md/pricing), [CLI](https://obsidian.md/cli), [Bases](https://obsidian.md/help/bases), [Web Clipper](https://obsidian.md/help/web-clipper), [Properties](https://obsidian.md/help/properties) · [Local REST API plugin](https://github.com/coddingtonbear/obsidian-local-rest-api) · [Omnisearch](https://github.com/scambier/obsidian-omnisearch) · [Copilot](https://github.com/logancyang/obsidian-copilot) · NotebookLM help: [source limits](https://support.google.com/notebooklm/answer/16213268), [answer 16215270](https://support.google.com/notebooklm/answer/16215270), [answer 16164461](https://support.google.com/notebooklm/answer/16164461) · [NotebookLM Enterprise overview](https://docs.cloud.google.com/gemini/enterprise/notebooklm-enterprise/docs/overview), [notebooks API](https://docs.cloud.google.com/agentspace/notebooklm-enterprise/docs/api-notebooks), [sources API](https://docs.cloud.google.com/gemini/enterprise/notebooklm-enterprise/docs/api-notebooks-sources) · Zotero: [storage](https://www.zotero.org/storage), [adding items](https://www.zotero.org/support/adding_items_to_zotero), [Web API basics](https://www.zotero.org/support/dev/web_api/v3/basics), [write requests](https://www.zotero.org/support/dev/web_api/v3/write_requests), [local API](https://www.zotero.org/support/dev/web_api/v3/local_api) · [Notion request limits](https://developers.notion.com/reference/request-limits) · [Anytype API](https://developers.anytype.io/)

**Standards, security, legal:** [RFC 9309](https://www.rfc-editor.org/rfc/rfc9309.html) · [sitemaps protocol](https://www.sitemaps.org/protocol.html) · [17 USC §105](https://www.law.cornell.edu/uscode/text/17/105) · [OWASP SSRF cheat sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html) · [OWASP LLM Top 10](https://genai.owasp.org/llm-top-10/) · Terms pages: [IRS](https://www.irs.gov/about-irs/use-of-content-from-irsgov), [CFPB](https://www.consumerfinance.gov/privacy/website-privacy-policy/), [SEC](https://www.sec.gov/about/privacy-information), [Federal Reserve](https://www.federalreserve.gov/disclaimer.htm), [FDIC archive](https://archive.fdic.gov/Content%20and%20Copyright), [FINRA](https://www.finra.org/terms-of-use), [BLS](https://www.bls.gov/opub/copyright-information.htm), [TreasuryDirect](https://www.treasurydirect.gov/legal-information/terms), [NBER](https://nber.org/policies.html), [arXiv API](https://info.arxiv.org/help/api/tou.html), [Kitces](https://kitces.com/terms-of-use/), [YouTube](https://www.youtube.com/t/terms)

**Credibility, ranking, eval:** [Google Search Quality Rater Guidelines, 2025-09-11](https://static.googleusercontent.com/media/guidelines.raterhub.com/en//searchqualityevaluatorguidelines.pdf) · [Wikipedia: Reliable sources](https://en.wikipedia.org/wiki/Wikipedia:Reliable_sources) and [perennial sources](https://en.wikipedia.org/wiki/Wikipedia:Reliable_sources/Perennial_sources) · [NewsGuard criteria](https://www.newsguardtech.com/ratings/rating-process-criteria/) · [ALCE](https://arxiv.org/abs/2305.14627) · [Attributed QA](https://arxiv.org/abs/2212.08037) · [Cormack et al., RRF](http://cormack.uwaterloo.ca/cormacksigir09-rrf.pdf) · [Elastic hybrid search](https://www.elastic.co/search-labs/blog/hybrid-search-elasticsearch) · [SBERT retrieve and rerank](https://sbert.net/examples/applications/retrieve_rerank/README.html) · [Kagi SlopStop](https://help.kagi.com/kagi/features/slopstop.html) · [IR book: near-duplicates](https://nlp.stanford.edu/IR-book/html/htmledition/near-duplicates-and-shingling-1.html) · [Google canonical URLs](https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls)

**Seed-source pages opened:** [TreasuryDirect I-bond rates](https://www.treasurydirect.gov/savings-bonds/i-bonds/i-bonds-interest-rates/) · [Freddie Mac PMMS](https://www.freddiemac.com/pmms) · [IRS Roth IRAs](https://www.irs.gov/retirement-plans/roth-iras) · [IRS Pub 970](https://www.irs.gov/publications/p970) · [CRR](https://crr.bc.edu/) · [Fed SCF](https://www.federalreserve.gov/econres/scfindex.htm) · [CFPB Your Money Your Goals](https://www.consumerfinance.gov/consumer-tools/educator-tools/your-money-your-goals/) · [CFPB owning a home](https://www.consumerfinance.gov/owning-a-home/) · [CFPB emergency fund guide](https://www.consumerfinance.gov/an-essential-guide-to-building-an-emergency-fund/) · [Wharton mortgage article](https://knowledge.wharton.upenn.edu/article/think-twice-paying-off-mortgage-early/) · [NBER w7318](https://www.nber.org/papers/w7318) · [ACL long-term care](https://acl.gov/ltc/basic-needs/how-much-care-will-you-need)

**Blocked or not opened (403/402/404):** ssa.gov, investor.gov, medicare.gov, bogleheads.org, morningstar.com, ebri.org, studentaid.gov, ssrn.com, investopedia.com, Vanguard/Fidelity/Schwab terms, perplexity.ai pricing, consensus.app pricing, Google "About this result".
