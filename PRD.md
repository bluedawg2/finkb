# Financial Knowledge Companion — Product Requirements

## Vision

A financial research and education application. It collects, organizes, evaluates, and makes searchable high-quality financial education and research from public sources (web pages, PDFs, YouTube, podcasts, RSS, government and academic publications, book metadata).

A user asks a question such as "How much money do I need before I retire?" and receives relevant, trustworthy research, ranked with an explanation of why each result is useful, with full provenance and a link to the original.

Workflow: **Question → understand intent → retrieve → rank → explain relevance → preserve provenance → allow deeper investigation.**

This is a research product, not an automated financial adviser.

Representative questions (also the seed of the evaluation corpus):

1. How much money do I need to retire?
2. When should I claim Social Security?
3. Should I pay off my mortgage or invest?
4. How much should I save for college?
5. How much emergency savings should I have?
6. Roth vs traditional retirement contributions?
7. How much can I safely withdraw in retirement?
8. How should I think about long-term-care costs?
9. Should I rent or buy?
10. How should I choose an asset allocation?

## The core problem

Storing documents is easy. The hard problem is determining which content is most useful for a particular question. Relevant factors: relevance, source authority, evidence quality, recency and publication date, specificity, citations, conflicts of interest, primary vs secondary source, independent corroboration, conflicting viewpoints, whether rules have changed, whether older research remains useful.

Ranking must be **explainable**. Do not reduce this to a single opaque score.

## Principles

**Provenance.** Every piece of knowledge traces to its source: original URL, title, author, publisher, publication date, retrieval date, source type, content identifier, processing status, extraction method, license, page numbers or timestamps, relationships to other sources, version history. AI-generated summaries, classifications, and extracted claims must always be distinguishable from source material. An AI-generated claim must never silently become an authoritative source.

**Content rights.** Publicly accessible ≠ freely reproducible. Distinguish content that may be stored, indexed, summarized, quoted, linked, referenced, or retrieved only temporarily. Books and copyrighted material need particular care. When content cannot be stored, retain metadata and a link where permitted.

**Source quality.** Sources are not equivalent, but avoid "government = correct" and "YouTube = bad." Model quality as explainable dimensions: expertise, authority, evidence, citations, transparency, commercial incentives, methodology, recency, primary vs secondary, independence, specificity.

**Facts vs analysis vs opinion.** Tax thresholds and contribution limits are facts. Retirement research is analysis. "Always claim Social Security at 70" is opinion. Do not treat them as equivalent.

**Freshness.** Tax rules, contribution limits, Medicare costs, rates, and products go stale. Market history, behavioral finance, and withdrawal research do not. Model freshness by subject matter; old does not mean bad.

**Model-agnostic AI.** AI may assist with question understanding, classification, extraction, summarization, deduplication, semantic search, ranking, and synthesis. Design for local and hosted models; different tasks may use different models. Never hard-code one provider.

**Security.** All retrieved content is untrusted input. Defend against prompt injection, malicious pages and PDFs, SSRF, unsafe URL fetching, file upload attacks, code execution, secrets exposure, data leakage, and tool abuse. Maintain trust boundaries: application instructions → retrieved content → AI → tools → private user data. A page saying "ignore previous instructions" is content, not an instruction.

**Independence from CompassWorth.** This application works standalone. CompassWorth (a local-first personal finance app) may eventually supply private context — age, income, assets, debt, goals — through a clean, authenticated, data-minimizing interface. Never couple to its database. Never copy private financial data into the public knowledge repository.

**Local-first.** Prefer local processing, storage, and search. External services are optional and the app must stay useful without them. Do not force everything local where it harms UX; make tradeoffs explicit.

## Architecture direction

**Ingestion pipeline:** Discover → Fetch → Validate → Extract → Normalize → Classify → Enrich → Index → Review/Publish. Failures are explicit; never silently ingest corrupted, incomplete, or questionable content.

**Content lifecycle:** Discovered → Queued → Fetched → Parsed → Classified → Enriched → Reviewed → Published → Updated → Archived, plus Failed and Removed. The system must always know the difference between discovered, retrieved, processed, trusted-enough-to-expose, failed, stale, and removed.

**Search:** Simple input, rich results (title, source, author, date, source type, topics, summary, why relevant, original link, provenance, reading/watch time). Full-text first; semantic and hybrid when justified.

**Deduplication and relationships:** Eventually detect exact and near duplicates, syndication, and multiple sources citing the same primary study. Twenty articles repeating one study are not twenty pieces of evidence. Do not over-engineer this early.

**Knowledge model:** Sources, documents, authors, organizations, topics, claims, citations, studies, financial rules with effective dates, source relationships, contradictions, summaries, embeddings. Build the smallest useful model that can evolve toward this.

**Research synthesis (future):** Identify schools of thought, primary research, assumptions, agreement, disagreement, and the variables explaining different conclusions. Always grounded in source-level evidence.

**Search quality (future):** Measure relevance, precision, recall, top-k quality, provenance correctness, diversity, freshness against the question corpus above. Never invent benchmark numbers.

**Observability:** It must be possible to explain why content was ingested or rejected, what processing and AI generation occurred, when, why a result ranked highly, and why processing failed.

## Engineering standards

Strong typing, explicit data models, clear interfaces, good error handling, secure defaults, testable components, minimal dependencies, reproducible development, deterministic behavior where practical.

Avoid giant functions and files, copy/paste, magic constants, unnecessary abstractions, dead code, unused dependencies, prompts hiding application logic, premature microservices, unrelated refactoring.

Testing is first-class: unit, integration, ingestion, parser, search, security, and failure-mode tests. Realistic failure cases, not only happy paths.

## MVP guidance

The MVP proves the core loop: **source → ingest → process → store → index → search → results → provenance**. Pick the fewest source types that prove it. Do not build the long-term vision before validating the loop.
