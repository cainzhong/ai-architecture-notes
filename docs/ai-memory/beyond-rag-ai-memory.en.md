# Beyond RAG: AI Memory Is More Than Chat History

## A Practical Architecture for Safe, Governed Continuity

## Bottom Line

Most RAG systems are designed to answer a present-tense question: *what information should the model retrieve for this request?* That is valuable, but it is not the same as continuity. As AI assistants move from single-turn Q&A into drafting, research, review, and operational workflows, they need to answer a different question: *where did this work leave off, and which parts of that history are still safe to use?*

This difference matters in real product architecture. A useful assistant may need to carry forward a stable writing preference, the current state of a document, previously explored research directions, confirmed matter context, or an explicit list of unfinished actions. None of those objects behave like ordinary retrieved passages.

The obvious implementations are also the most fragile. Continuously appending conversation history creates an ever-growing prompt. Embedding every message and running similarity search turns temporary instructions, obsolete findings, and unrelated workspace details into one undifferentiated corpus.

A better abstraction is a governed memory layer: a service that converts selected parts of prior work into typed, scoped, traceable, and removable records. Its purpose is not to remember everything. Its purpose is to preserve the right state, within the right boundary, for the right future task.

## Where the Problem Comes From

Many enterprise AI assistants begin with a familiar pattern: connect the model to a knowledge base, retrieve relevant materials for the user's question, and let the model generate an answer. That is the most common value of RAG. It gives the model current external context for the request in front of it.

The problem changes when an assistant moves from one-shot Q&A into continuous work. A user may ask the assistant to analyze a document today and continue revising the same document tomorrow. They expect the assistant to know which version is current, which options were already ruled out, which sources have been checked, which conclusions were only tentative, what still needs follow-up, and how the user prefers the output to be structured.

At that point, “retrieve relevant documents” is no longer enough. The system must answer a different question: where did this work leave off, and which historical state is still safe to use now?

The most direct solutions are to append all chat history to the prompt or embed every message into a vector database and search it later. Both can make an assistant appear to remember the past, but both create engineering risk: the context keeps growing, temporary instructions can become durable preferences, stale findings can return as if they were current, and workspace boundaries become blurry. The system also creates derived copies for search, summaries, and caches. If the original source is later deleted or corrected but those derived copies remain, old content can re-enter future prompts through the side door.

So the AI Memory discussed here is not a longer context window and not vector search over chat logs. It is a governed system capability that turns a small amount of useful prior work into structured objects, while continuously tracking where each object came from, where it may be used, whether it is still valid, and how it can be withdrawn.

## Why “Store Chat History and Search It” Is Not Enough

When a stateful assistant is introduced into a large-scale enterprise environment, several failure modes tend to appear together:

- The assistant cannot reliably continue a document or research task across sessions.
- Stable preferences, matter facts, drafting context, research findings, and open actions are stored as if they were the same kind of text.
- A one-time instruction is accidentally promoted into a durable preference.
- Similar content from another workspace competes in the same retrieval set.
- A vector index becomes an unofficial source of truth even after the underlying record has changed or been deleted.
- Deletion, correction, expiry, and index rebuilding are handled as operational afterthoughts.

These are not primarily embedding problems. They are lifecycle and trust-boundary problems.

## What This Article Answers

The design rests on five principles.

First, classify memory by meaning before classifying it by retention period. A writing preference and an old research conclusion may both be durable, but they require entirely different retrieval and validation rules.

Second, separate extraction from persistence. A model may propose a candidate memory; it should not decide whether the record becomes durable, where it is scoped, or whether it is safe to use.

Third, apply scope and authorization before ranking. Access boundaries are hard predicates, not relevance features.

Fourth, keep one authoritative record store. Exact, keyword, and semantic indexes are rebuildable projections, not primary data stores.

Finally, assemble context again for every request. The system should select a small set of relevant, permitted, and current memory items instead of replaying an unlimited transcript.

Together, these principles form one engineering loop: decide what is eligible to become Memory, let the model produce only Candidate Memory, use backend governance to promote or reject it, filter by scope before ranking, resolve every index hit back to the source of truth, and treat deletion, correction, expiry, and rebuild as part of the core design rather than operational cleanup.

## How to Design a Practical Memory Layer

### 1. Classify by meaning before retention

The short-term/long-term split is useful for storage planning, but it is too weak to serve as the primary memory model. Retention answers *how long a record lives*. It does not answer:

- which extraction schema applies;
- whether the content is eligible for durable storage at all;
- where it may be reused;
- whether it should be loaded deterministically or discovered through search;
- what role it may play inside a prompt;
- what should happen when its source changes, conflicts, expires, or disappears.

A stable writing preference and a prior research finding may both remain useful for months. The preference can directly shape presentation. The finding may only restore an investigation path and may require fresh evidence before it supports a new conclusion. Equal retention does not imply equal behavior.

Typed memory therefore needs to be more than a label. Each type acts as an executable contract:

```text
MEMORY TYPE CONTRACT
+ extraction schema
+ promotion requirements
+ server-side scope resolver
+ deduplication and conflict semantics
+ retrieval and prompt-usage policy
+ expiry, correction, deletion, and rebuild behavior
```

A practical starting set of `memory_type` values can be deliberately small:

| Memory type                 | What it represents                                           | Primary boundary                                             |
| --------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `user_workstyle_preference` | Stable work style such as structure, tone, level of detail, citation style, and formatting preference | Must not absorb matter facts, client facts, authorization claims, or one-off instructions |
| `matter_context`            | Confirmed background for a matter or validated work boundary, such as goals, facts, roles, constraints, and key issues | Reusable only inside its validated work boundary             |
| `drafting_context`          | The state of a specific document, including version, revision direction, section status, and unfinished edits | Should be loaded by document, version, and work boundary, not by broad similarity |
| `research_finding`          | A research path rather than a current answer: explored questions, source pointers, working findings, limits, and items to verify | Can guide later work, but time-sensitive claims require fresh validation before use |
| `workflow_open_task`        | Explicit follow-up actions, status, dependencies, blockers, and completion evidence | Must be requested or confirmed, not inferred by the model as something that “should” be done |

These are the core categories that become formal memory records. They determine the extraction schema, promotion rules, scope resolution, deduplication semantics, retrieval policy, and lifecycle behavior.

Two neighboring objects matter just as much, but they should not be mixed into `memory_type`. Trace is the evidence layer: raw events, tool calls, completed turns, and source references used for reconstruction, audit, and extraction input. Runtime Policy is the control plane: write rules, retrieval rules, retention rules, revalidation rules, and prompt assembly rules loaded from a governed policy registry. Neither should be extracted from ordinary conversation or rendered as a business memory card.

#### How type changes runtime behavior

The type influences the whole pipeline:

1. **Extraction window.** An explicit preference may be clear in one completed turn. Matter context or a research trail often needs a small, topic-bounded window.
2. **Scope resolution.** Preferences typically bind to a user, drafting context to a document, and matter context to a validated work boundary. The model may suggest a scope, but the backend resolves it from trusted context.
3. **Deduplication.** Two differently worded preferences may describe the same logical object. Conflicting research observations should usually retain their sources rather than overwrite one another silently.
4. **Retrieval mode.** Current document state is a deterministic lookup. Historical research is more naturally discovered within an already-authorized boundary.
5. **Prompt usage.** A preference may shape formatting; context may supply background; a research record must not become a system instruction.
6. **Lifecycle governance.** Preferences may remain durable, drafting context is usually replaced by later document versions, and research records may become stale or require revalidation when sources change or time passes.

#### Type and lifecycle are orthogonal

Retention remains important, but it becomes a second dimension:

| Lifecycle      | Meaning                                             | Typical objects                                    |
| -------------- | --------------------------------------------------- | -------------------------------------------------- |
| Ephemeral      | Exists only for the current request                 | Tool intermediates and temporary calculations      |
| Session-scoped | Supports continuity within a session                | Completed turns and rolling summaries              |
| Durable scoped | Reusable across sessions under an explicit boundary | Preferences, matter context, and document state    |
| Audit-only     | Retained for traceability, not ordinary answers     | Promotion decisions, deletion jobs, and run traces |

This two-dimensional model avoids a common mistake: treating “durable” as permission for broad reuse, or treating “short-lived” as unimportant to provenance and deletion.

At the logical level, a durable record should carry more than text:

```text
MEMORY RECORD
+ stable ID and record version
+ type and subtype
+ canonical content and structured payload
+ server-resolved scope
+ source references and provenance
+ confidence, freshness, and lifecycle status
+ content hash and logical dedupe key
+ extraction, policy, and schema versions
+ index synchronization state
```

The content describes *what the system remembers*. Scope, provenance, version, and status explain *why it may remember it, where it may use it, and whether it is still valid*.

### 2. Separate the Write Path from Retrieval Context Assembly

At the system level, writing memory and reading memory should be separated. The Write Path decides what the system has actually chosen to remember. Retrieval Context Assembly decides what subset of those records is allowed, useful, current, and small enough to enter the next model call.

```text
Write Path

Raw Events -> Completed Turns -> Candidate Memory -> Promotion Gate -> Authoritative Memory Records
                      |              |
                      |              -> Provenance and Audit
                      |
                      -> Async Index Tasks
                           |- Exact Index
                           |- Keyword Index
                           `- Semantic Index

Retrieval Context Assembly

Current Request + Trusted Scope
    |- Known-Scope Lookup: preferences, session summary, current document, open tasks
    `- Hybrid Discovery: exact match / keyword / semantic retrieval
             |
             v
         Source-of-Truth Fetch
             |
         Second-Pass Policy and Freshness Check
             |
         Bounded Memory Cards
             |
               Main LLM
```

This split keeps “what the system remembers” separate from “how the system finds it again.” A Memory Record is the source of truth. Exact, keyword, and semantic indexes only improve discoverability.

### 3. Write Path: From Chat History to Governed Memory

The write path is not a summarization job. Its purpose is to turn a completed unit of work into a small number of reusable records without allowing transient model output to become durable state by accident.

> Every JSON object below is synthetic and educational. It does not represent a production API, table, or environment. Field names, enums, thresholds, and token budgets illustrate component contracts and should be replaced by implementation-specific schema and policy registries.

```text
WRITE PATH

[Trace Events]
    |
    v
[Completed Turn]
    |
    v
[Strategy Routing]
    |
    v
[Extraction Input Package]
    |
    v
[LLM / Rule Extractor]
    |
    v
[Candidate Memory]
    |
    v
[Schema Validation]
    |
    v
[Policy Validation]
    |
    v
[Scope Resolution]
    |
    v
[Dedupe / Consolidation]
    |
    v
[Typed Memory Record]
    |
    v
[Retrieval Index Async Queue]
```

The key boundary is simple: LLMs extract candidates; the backend decides what becomes durable memory. A candidate may suggest content, type, scope, confidence, and revalidation hints, but the final typed memory record must be produced by deterministic validation, write policy, trusted identity, source references, and deduplication logic.

#### 3.1 Gate events before interpreting them

An enterprise assistant emits many records that should never be treated as stable context: streaming fragments, abandoned drafts, incomplete tool calls, generated suggestions the user did not select, deleted messages, and results already marked stale or requiring recomputation.

An admission gate filters on deterministic state rather than semantic interpretation. A source is eligible only when the interaction is complete, required tool calls have resolved, the source remains active, and its work boundary matches the current trusted context. Rejected events may remain in a controlled trace store for reconstruction, but they do not feed session summaries, extraction, or prompt assembly.

This distinction matters: retaining an event for operational replay is not the same as approving it for future influence.

#### 3.2 Reconstruct a completed turn

A single user-visible turn may span a user message, several tool calls, tool responses, model streaming output, a final response, and an updated artifact. Extracting message by message produces duplicate observations and makes intermediate output look final.

A turn builder reconstructs a stable unit containing:

- a stable turn identifier and session order;
- the user request and final assistant response;
- selected tool results and artifact references;
- normalized task, workspace, and document metadata;
- source-event references and a content fingerprint;
- a compact retrieval representation and a rolling-summary representation;
- completion and lifecycle state.

The completed turn answers *what happened in this interaction*. It remains a continuity and extraction source, not a durable typed memory record.

The builder is usually asynchronous and idempotent. Late tool output, artifact updates, or source deletion may require the same turn to be updated. Reprocessing identical input should be a no-op; changed source state should create a new record version rather than a duplicate logical turn.

The following synthetic example shows the shape of a completed turn without exposing any real account, environment, or business data:

```json
{
    "turn_ref": "turn-example-001",
    "session_ref": "current-session",
    "turn_index": 7,
    "scope": {
        "workspace_ref": "current-workspace",
        "document_ref": "current-document"
    },
    "source_event_refs": [
        "event-user-request",
        "event-tool-result",
        "event-assistant-final",
        "artifact-current-draft"
    ],
    "user_request": "Continue revising the current document and preserve the approved structure.",
    "assistant_final_summary": "Updated the document and listed the remaining sections.",
    "tool_result_summary": "Loaded the latest approved document revision.",
    "artifact_state": {
        "artifact_ref": "current-document",
        "revision_ref": "revision-example",
        "state": "updated"
    },
    "normalized_metadata": {
        "task_type": "document_revision",
        "intent": "continue_existing_work"
    },
    "retrieval_text": "Continue the current document, preserve its approved structure, and complete the remaining sections.",
    "status": "active",
    "is_completed": true,
    "content_hash": "sha256:<turn-content-hash>",
    "completed_at": "<timestamp>"
}
```

This example has two layers. The base completed turn is assembled from deterministic events first: the turn ID, session order, scope, source references, final request and response, artifact state, metadata, status, and content hash. Derived summaries sit on top of that stable record. They can be generated later, regenerated when policy changes, or omitted in a first version without invalidating the completed turn itself.

`turn_ref`, `session_ref`, and `turn_index` locate the interaction and restore the order of work. Scope fields such as `workspace_ref` and `document_ref` describe the trusted boundary for later extraction, retrieval, and deletion propagation. `source_event_refs` connect the turn back to raw messages, tool results, and artifacts so that a correction or deletion can be traced downstream.

The textual fields serve different jobs. `user_request` and `assistant_final_summary` capture the stable user intent and final assistant outcome. `tool_result_summary` explains what the answer depended on. `artifact_state` records what changed in the work product. `retrieval_text` is not display copy; it is a compact search surface for near-term continuity.

If an LLM is used to generate summaries, it is still operating on an already completed, scoped, and reconstructable turn. The generated fields should pass backend validation before they feed extraction or prompt assembly: only allowed keys, bounded length, no authorization claims, no secrets, no cross-scope content, and no assertion that conflicts with the source references. A failed or missing summary should degrade the next step, not erase the base completed turn.

#### 3.3 Route by strategy and bound the extraction window

One universal extraction prompt is attractive but unsafe. A stable preference, a document state update, a research trail, and an open action have different schemas and different failure modes.

The strategy router decides whether a completed turn has durable value, which type-specific strategies apply, and how much surrounding context each strategy may inspect. A practical implementation starts with deterministic metadata and text signals, then uses a constrained classifier only for ambiguous cases. The classifier selects from a backend-controlled registry; it cannot introduce a new type or a broader scope.

The extraction window is also type-specific. An explicit preference may require one turn. Drafting context may need the current turn plus recent turns for the same document. Matter context and research trails may need a short topic-bounded sequence or the previous session summary. The window stops when the work boundary, document, topic, or token budget changes materially.

#### 3.4 Build a minimal, reconstructable input package

The extractor should not receive an unrestricted transcript. An input builder creates a strategy-specific package:

```text
EXTRACTION INPUT PACKAGE
+ immutable strategy and schema versions
+ server-trusted identity and active work scope
+ one or more gated completed turns
+ minimal summaries of related existing records
+ source references
+ output schema
+ prohibited-content and safety constraints
```

The prompt should explicitly state that conversation content is data, not instruction. Text in the source cannot redefine identity, set final scope, change lifecycle status, or modify platform policy.

Persisting the complete package in application logs creates another sensitive copy. A safer extraction trace stores a run ID, canonical package hash, source references, strategy/prompt/schema versions, model configuration, token counts, status, and error codes. When debugging is authorized, the package can be reconstructed from current source records and compared by hash. Reconstruction must still honor later deletion and access changes.

A strategy-specific package can remain compact while preserving the important contracts:

```json
{
    "strategy": {
        "strategy_id": "drafting_context_extraction",
        "strategy_version": "v1",
        "allowed_memory_types": [
            "drafting_context"
        ],
        "output_schema": "drafting_context_candidate_v1"
    },
    "trusted_context": {
        "actor_ref": "current-user",
        "scope": {
            "workspace_ref": "current-workspace",
            "document_ref": "current-document"
        }
    },
    "completed_turns": [
        {
            "turn_ref": "turn-example-001",
            "user_request": "Continue revising the current document.",
            "assistant_final_summary": "Updated the document and identified remaining work."
        }
    ],
    "existing_related_memories": [
        {
            "record_ref": "memory-draft-state",
            "record_version": 3,
            "summary": "The previous revision was awaiting updates to two sections."
        }
    ],
    "source_refs": [
        "turn-example-001",
        "artifact-current-draft"
    ],
    "forbidden_content": [
        "authorization claims",
        "secrets or credentials",
        "content from another validated scope"
    ]
}
```

The package is best understood as five contracts: strategy, boundary, input, comparison set, and hard exclusions.

`strategy` identifies the extraction task and its immutable version. It tells the extractor which memory type is allowed and which output schema it must follow. This turns extraction from a free-form summarization prompt into a narrow, reviewable operation.

`trusted_context` is supplied by the backend. It describes the current actor and work boundary, but it does not delegate authorization to the model. The model may use the boundary to interpret the input; the promotion gate still resolves the final scope.

`completed_turns` is the bounded evidence window. It includes only gated, completed interactions selected for this strategy, not the entire transcript. `existing_related_memories` gives the model just enough context to notice a likely duplicate or update, while leaving actual merge and conflict handling to deterministic backend logic.

`source_refs` keeps the candidate tied to evidence. A memory proposal without usable source references should have a hard time becoming durable, because later correction, deletion, review, and re-extraction all depend on provenance. `forbidden_content` is the guardrail list for this run: authorization claims, secrets, credentials, and cross-scope content are data to reject or ignore, not instructions to obey.

#### 3.5 Isolate candidates from retrieval

The extractor—rules, an LLM, or a combination—writes candidate memory only. A candidate can support offline evaluation, error analysis, and review, but it remains invisible to retrieval and prompt assembly.

Model-generated scope, confidence, status, or revalidation flags are proposals. They are not governance decisions. This isolation prevents one extraction error or injected instruction from affecting every future request before the backend has validated it.

The candidate retains extraction evidence while making its non-active state explicit:

```json
{
    "candidate_ref": "candidate-example-001",
    "extraction_run_ref": "extraction-run-example",
    "memory_type": "drafting_context",
    "proposed_scope": "document",
    "candidate_text": "The current document uses the approved structure and still has two open revisions.",
    "structured_payload": {
        "revision_ref": "revision-example",
        "state": "in_progress",
        "completed_changes": [
            "preserved the approved section order"
        ],
        "open_changes": [
            "complete the implementation section",
            "verify the summary against the current document"
        ]
    },
    "source_refs": [
        "turn-example-001",
        "artifact-current-draft"
    ],
    "confidence_score": 0.91,
    "promotion_status": "pending"
}
```

The important word here is `pending`. `candidate_ref` identifies this model output inside the candidate store, but it is not the stable ID of a formal memory record. `extraction_run_ref` ties the proposal back to the strategy version, input package, model settings, and execution trace.

`memory_type` and `proposed_scope` are suggestions, not authority. They help the backend choose validation logic, but the final type, scope, status, and permissions are resolved later. `candidate_text` is useful for review and display, while `structured_payload` is what lets the system validate fields, deduplicate records, and run deterministic retrieval later.

`source_refs` is the evidence trail. `confidence_score` can influence the promotion decision, but it cannot override missing sources, scope conflicts, forbidden content, or deduplication rules. Until the promotion gate makes a decision, the candidate is not searchable, not durable, and not eligible for the main model prompt.

#### 3.6 Make promotion deterministic

The promotion gate evaluates a candidate in a fixed order:

1. **Schema validation:** required fields, types, controlled values, and numeric ranges.
2. **Source validation:** source existence, active state, and boundary consistency.
3. **Scope resolution:** final scope derived from server-side identity and object relationships.
4. **Policy validation:** type-specific confidence, provenance, retention, and revalidation requirements.
5. **Dedupe analysis:** exact-content fingerprint plus logical-record key.
6. **Consolidation:** comparison of payload, provenance, confidence, recency, and conflict semantics.

Some checks should remain non-configurable hard blocks: authorization claims embedded in content, secrets or credentials, and references that cross a validated work boundary. Ordinary threshold tuning must not bypass them.

Versioned policy can express tunable requirements while code-level controls preserve non-negotiable boundaries:

```json
{
    "policy_id": "drafting_context_write_policy",
    "version": "v1",
    "applies_to": {
        "strategy_id": "drafting_context_extraction",
        "memory_type": "drafting_context"
    },
    "promotion_rules": {
        "minimum_confidence": 0.8,
        "required_payload_fields": [
            "revision_ref",
            "state",
            "open_changes"
        ],
        "required_source_types": [
            "completed_turn",
            "document_artifact"
        ],
        "allowed_scope_levels": [
            "document"
        ],
        "missing_source_decision": "PENDING_REVIEW"
    },
    "hard_blocks": [
        "authorization_claim",
        "secret_or_credential",
        "cross_scope_source"
    ]
}
```

The policy separates tunable requirements from non-negotiable boundaries. `policy_id` and `version` make every promotion decision auditable. `applies_to` prevents a rule written for drafting context from being reused accidentally for preferences, research trails, or workflow tasks.

`promotion_rules` holds the adjustable requirements: minimum confidence, required payload fields, required source types, allowed scope levels, and the fallback when source evidence is incomplete. `hard_blocks` should not be relaxed by ordinary configuration. A user statement that claims authority, a secret, or a source from another validated scope must remain blocked even if someone lowers a confidence threshold.

Promotion needs a richer decision model than a Boolean:

| Decision         | Meaning                                                      | Typical action                                               |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `ADD`            | No logical record exists                                     | Create a new record                                          |
| `UPDATE`         | The same logical record gained detail or stronger provenance | Update and increment the version                             |
| `SKIP`           | Exact duplicate or no meaningful new evidence                | Avoid a new record; optionally update observation metadata   |
| `SUPERSEDE`      | A new state explicitly replaces an older one                 | Activate the new version and retire the old one from normal retrieval |
| `PENDING_REVIEW` | Evidence is incomplete or records conflict                   | Keep it non-active until reviewed or revalidated             |
| `BLOCK`          | A hard boundary was violated                                 | Reject and retain only a minimal reason code                 |

The result remains explainable and versioned rather than collapsing to `true` or `false`:

```json
{
    "candidate_ref": "candidate-example-001",
    "decision": "UPDATE",
    "target_record_ref": "memory-draft-state",
    "resolved_scope": {
        "workspace_ref": "current-workspace",
        "document_ref": "current-document"
    },
    "write_policy_version": "drafting_context_write_policy@v1",
    "reason_codes": [
        "same_logical_record",
        "newer_document_state",
        "required_sources_present"
    ],
    "resulting_record_version": 4
}
```

`content_hash` and `dedupe_key` solve different problems. The content hash detects identical normalized output and supports idempotency. The dedupe key identifies the logical memory object. Hash-only dedupe creates multiple records for paraphrases; key-only dedupe cannot distinguish a retry from a meaningful update.

#### 3.7 Commit authoritative state before derived search views

Only promoted candidates become authoritative records. The record carries its type, structured payload, resolved scope, provenance, version, lifecycle state, freshness, dedupe metadata, and index synchronization state.

This generalized record keeps content and governance metadata together without tying the design to a specific database:

```json
{
    "record_ref": "memory-draft-state",
    "record_version": 4,
    "memory_type": "drafting_context",
    "memory_subtype": "current_document_revision",
    "scope": {
        "workspace_ref": "current-workspace",
        "document_ref": "current-document"
    },
    "canonical_text": "The current document uses the approved structure and has two open revisions.",
    "structured_payload": {
        "revision_ref": "revision-example",
        "state": "in_progress",
        "open_changes": [
            "complete the implementation section",
            "verify the summary against the current document"
        ]
    },
    "source_refs": [
        "turn-example-001",
        "artifact-current-draft"
    ],
    "provenance": {
        "strategy": "drafting_context_extraction@v1",
        "schema": "drafting_context_record_v1",
        "promotion_decision": "UPDATE"
    },
    "quality": {
        "confidence": 0.91,
        "freshness": "current",
        "salience": 0.82
    },
    "content_hash": "sha256:<record-content-hash>",
    "dedupe_key": "sha256:<logical-record-key>",
    "status": "active",
    "index_sync_status": "pending"
}
```

The record body says what the system remembers. The surrounding fields explain why it can remember it, where it can be used, and whether it is still usable.

`record_ref` and `record_version` give the memory a stable identity and a current version. They are the handles used for deletion, audit, index synchronization, and conflict handling. `memory_type` and `memory_subtype` select the governing schema and policy. `canonical_text` is the reviewable human expression; `structured_payload` is the machine-readable state used for filtering, deduplication, and deterministic lookup.

`scope` is resolved by the server, not by the model. `source_refs` and `provenance` explain how the record was formed and allow later review, correction, deletion propagation, or re-extraction. `quality`, `status`, and `index_sync_status` prevent a retrieval hit from becoming automatic permission to use the record. `content_hash` supports idempotent processing, while `dedupe_key` captures the logical memory object across paraphrases and updates.

Exact, keyword, and semantic indexes are created asynchronously through an outbox. Where the primary store supports transactions, the record update and outbox event should commit in the same local transaction. That avoids the two classic dual-write gaps: a committed record with no indexing event, or an indexing event for a record that rolled back.

The outbox is a separate state machine from the record it indexes:

```json
{
    "outbox_ref": "index-job-example-001",
    "record_ref": "memory-draft-state",
    "source_record_version": 4,
    "action": "UPSERT",
    "target_indexes": [
        "keyword",
        "semantic"
    ],
    "semantic_roles": [
        "document_goal",
        "open_changes"
    ],
    "embedding_model_version": "current-approved-version",
    "status": "queued",
    "attempt_count": 0
}
```

Record state and outbox state are separate. A record may be active while its indexing task is queued, processing, retryable, or terminally failed. This creates a short period in which deterministic lookup can see the record but discovery search cannot. That is an explicit consistency model, not data loss.

An index failure does not revoke an approved memory. Retry, dead-letter handling, redrive, and full rebuild repair the derived views. Write-path observability should therefore include gate rejection reasons, candidate rate, false-promotion rate, consolidation decisions, outbox lag, indexing latency, retry count, and orphaned-index detection.

### 4. Retrieval Context Assembly: Not Similar Text, But Safe Context Assembly

Retrieval Context Assembly is a request-time control plane. Its job is not merely to find similar records, but to decide which current records are permitted, useful, fresh enough, and compact enough to influence this model call.

```text
READ-SIDE REQUEST PATH

Current Request
    -> Trusted Identity
    -> Retrieval Planner
    -> Path A: Known-Scope Lookup
    -> Path B: Semantic / Keyword / Exact Discovery
    -> Source-of-Truth Fetch
    -> Second-Pass Policy Filter
    -> Legal Revalidation Trigger
    -> Context Budget / Rerank / Compression
    -> Prompt Memory Cards
    -> Main LLM

Side output: Minimal Retrieval Trace
```

#### 4.1 Plan retrieval only when continuity is needed

Not every request benefits from memory. A self-contained question or a genuinely new task can skip memory retrieval. The planner activates when the user asks to continue earlier work, the request references an active session or document, or the task needs preferences, open actions, scoped context, or a prior investigation path.

Task classification should be deterministic where possible. A model can assist when signals are ambiguous, but it selects from a controlled list and cannot broaden scope or enable a new source.

It helps to separate a **retrieval policy** from a **retrieval plan**. The policy is a versioned rule set describing which memory types a task may use, which controls are mandatory, and what prompt roles are allowed. The plan is ephemeral: it combines that policy with the current validated scope, query text, available sources, revalidation triggers, and token budget.

The plan describes how this request will retrieve memory; it does not persist the text of actual hits:

```json
{
    "task_class": "continue_current_document",
    "validated_scope": {
        "workspace_ref": "current-workspace",
        "document_ref": "current-document",
        "session_ref": "current-session"
    },
    "retrieval_policy_version": "document_continuity_policy@v1",
    "paths": {
        "known_scope_lookup": [
            "active_session_summary",
            "recent_completed_turns",
            "current_drafting_context",
            "open_work"
        ],
        "hybrid_discovery": [
            "related_project_context",
            "research_trail"
        ]
    },
    "revalidation": {
        "required_if_selected": [
            "research_trail"
        ],
        "trigger": "used_for_current_factual_judgment"
    },
    "memory_budget": {
        "total_tokens": 1600,
        "session_continuity": 450,
        "preferences": 200,
        "workspace_and_draft": 550,
        "research_trail": 400
    }
}
```

The plan is a runtime decision, not durable business data. `task_class` identifies the kind of continuity the request needs. `validated_scope` is the server-approved boundary; if a required boundary is missing, the planner should narrow the source set or skip that memory type rather than search globally.

`retrieval_policy_version` records which policy made the call. `paths` separates deterministic known-object lookup from discovery search. `revalidation` keeps two ideas apart: a record may carry a persistent freshness requirement, but runtime refresh is triggered only if the selected memory will support a current factual judgment. `memory_budget` reserves prompt space by purpose so historical context does not crowd out the current request, fresh tool results, or the model's answer.

#### 4.2 Use Known-Scope Lookup for known objects

Deterministic lookup is appropriate when the logical object is already known: the active user's relevant preferences, the current session summary, recent completed turns, current document state, or open actions in the active workspace.

These records are not “similar content.” They are current objects selected by stable IDs, server-validated scope, lifecycle state, and ordering. Recent turns, for example, should be selected by session plus sequence, not by interpreting a random identifier as chronological.

Direct lookup still flows through the final policy check. Knowing exactly which record to load does not prove that it remains active or permitted.

#### 4.3 Use Exact, Keyword, and Semantic Discovery for discovery

Discovery is appropriate when the boundary is known but the relevant record is not.

- **Exact retrieval** handles validated identifiers or structured keys. Exact hits remain explicit candidates even when their semantic score is low.
- **Keyword retrieval** handles names, terms, statuses, and phrases that should match at the surface level.
- **Semantic retrieval** handles conceptual similarity, intent, and differently worded problem statements.

The three modes may run in parallel. Results are merged by stable record ID while retaining the evidence behind each hit. Write-time deduplication reduces duplicate records; retrieval-time deduplication combines multiple paths to the same record.

#### Keep semantic roles separate

A structured memory can contain several semantic roles. A research trail may include a topic, a working finding, source context, and unresolved caveats. Combining them into one embedding makes very different queries compete in the same representation.

| Role              | Representative input                                       | Intended query                                |
| ----------------- | ---------------------------------------------------------- | --------------------------------------------- |
| `topic`           | Problem statement, category, and key entities              | Have we worked on a related topic?            |
| `working_finding` | Provisional findings and factors                           | What did earlier work conclude provisionally? |
| `source_context`  | Source names, categories, uses, and short summaries        | What material informed the earlier work?      |
| `caveat`          | Limitations, open questions, and revalidation requirements | What remains uncertain or outdated?           |

Named vectors give each role a distinct search space. If the engine does not support them, separate collections or physical indexes are a reasonable alternative. Putting heterogeneous roles in one HNSW graph and relying only on metadata filtering can reduce role-specific recall: in many ANN implementations, filtered querying is not equivalent to traversing a graph built exclusively for that role, and unrelated vectors may still consume traversal and candidate budgets.

#### 4.4 Filter before ranking

The allowed-scope predicate is built from server-trusted context and applied before exact, keyword, or semantic ranking. Scope is a hard predicate; similarity, freshness, confidence, and salience are ranking signals.

Global top-K followed by authorization filtering fails in two ways. It exposes out-of-scope candidates to intermediate processing, and it causes candidate starvation: highly similar records outside the boundary may consume the search budget before valid records are considered.

If a required scope is missing, the planner narrows the source set or skips that memory type. It does not fall back to a broader search. Index-level filtering remains only the first pass because an asynchronous index may carry an older scope or lifecycle state.

#### 4.5 Resolve every hit against the source of truth

An index hit should contain a stable record pointer, index version, matched semantic role, and the minimal retrieval scores. The system then batch-fetches current authoritative records.

The fetch must obtain both governance data—scope, current authorization, record version, active/deleted/stale/superseded state, and expiry—and the fields required for the current prompt role. If the record is missing, deleted, or unreadable in the current boundary, the hit is dropped and an idempotent cleanup event is emitted.

Falling back to stale index text would invert the architecture by turning a derived view into the source of truth. A known-scope lookup that already reads the primary store can skip the extra fetch, but not the second-pass policy evaluation.

#### 4.6 Run a second policy pass on current state

After fetching the record, the backend verifies in order:

1. current scope and authorization;
2. lifecycle state and source validity;
3. whether the current task may use this memory type;
4. the allowed role: writing style, session continuity, matter background, drafting context, open work, or research trail;
5. whether revalidation is required before use;
6. the minimal field projection for downstream ranking and prompt assembly.

The output can be deliberately small:

| Decision                | Meaning                                                  |
| ----------------------- | -------------------------------------------------------- |
| `ALLOW`                 | Current state and intended use satisfy policy            |
| `HOLD_FOR_REVALIDATION` | Revalidation against current sources must complete first |
| `DROP`                  | Scope, state, provenance, or intended use is invalid     |

The model may follow a usage boundary in a memory card. It must not be responsible for authorization or lifecycle enforcement.

#### 4.7 Distinguish persistent revalidation requirements from runtime triggers

A time-sensitive record can carry a persistent `requires_revalidation`-style marker. The marker means that the record cannot support a new factual conclusion without current evidence. It does not mean every retrieval must call an external system.

Runtime revalidation is required only when the record is selected and the current task intends to rely on its working finding. Formatting changes, draft restoration, or display of an open action do not need to load the old finding at all. Skipping revalidation for such a request does not clear the persistent requirement.

Revalidation has three useful outcomes:

- current evidence supports the earlier finding, so it may be used as continuity context with provenance and observation time;
- current evidence conflicts, so the fresh source wins and the old record becomes stale, superseded, or reviewable;
- evidence is unavailable, so the system retains only the topic, source pointer, or unresolved action—not the old conclusion.

This pattern applies beyond any single domain: product documentation, operational runbooks, pricing, platform capabilities, and external knowledge all change over time.

#### 4.8 Rerank within explicit context budgets

Only allowed and, where required, revalidated candidates reach reranking. Useful signals include exact and keyword evidence, semantic similarity for the selected role, task-intent match, scope specificity, freshness, confidence, salience, and prior successful use. Staleness, conflict, and redundancy contribute penalties.

A document-bound state may outrank broad matter context even with a lower cosine similarity. Exact evidence can preserve a candidate, but it does not bypass lifecycle checks or prompt budgets.

Memory budgets are easier to reason about when allocated by purpose: preferences, session continuity, matter and drafting context, and research trails. Category limits may be soft, but the total memory budget is hard. The request, current tool output, and answer generation need separately reserved capacity.

Compression should also remain type-aware. Preserve a preference as a rule, drafting context as version plus outstanding edits, and a research trail as topic, source pointers, and caveats. A universal summarizer would undo the benefits of typed memory by mixing those roles back into unstructured prose.

#### 4.9 Render bounded memory cards

The prompt receives a minimal projection, not the raw record:

```text
[Memory type | source=<id> | record_version=<version> | scope=<scope> | usage=<usage>]
Content: <only the fields required for this request>
Boundary: <allowed use and prohibited inference>
```

A clean prompt separates trust zones:

```text
[TRUSTED RUNTIME INSTRUCTIONS]
Small usage constraints derived by the backend from current policy

[CURRENT REQUEST]
The user's task for this model call

[CURRENT TOOL OR KNOWLEDGE CONTEXT]
Fresh evidence retrieved or produced for this request

[MEMORY CONTINUITY CONTEXT]
Governed preference, session, draft, workflow, and research cards
```

Memory cards are untrusted continuity data, not system instructions. Imperative text inside a card cannot grant access, change platform policy, or override fresh tool output.

#### 4.10 Record a minimal retrieval trace

When memory is actually used, a minimal trace should record the request ID, validated scope, task class, policy version, executed paths, aggregate candidate counts, selected record IDs and versions, prompt roles, revalidation state, and failure stage.

```json
{
    "request_ref": "request-example-001",
    "status": "completed",
    "task_class": "continue_current_document",
    "retrieval_policy_version": "document_continuity_policy@v1",
    "executed_paths": [
        "known_scope_lookup",
        "hybrid_discovery"
    ],
    "candidate_summary": {
        "found": 8,
        "dropped_by_policy": 3,
        "excluded_by_budget": 2,
        "selected": 3
    },
    "selected_contexts": [
        {
            "record_ref": "memory-draft-state",
            "record_version": 4,
            "usage": "drafting_context"
        },
        {
            "record_ref": "memory-user-preference",
            "record_version": 2,
            "usage": "writing_style"
        },
        {
            "record_ref": "session-summary-current",
            "record_version": 5,
            "usage": "session_continuity"
        }
    ],
    "revalidation": {
        "required": false,
        "status": "not_needed"
    }
}
```

This trace is intentionally about decisions, not copied content. `request_ref`, `status`, `task_class`, and `retrieval_policy_version` identify the run and the rule set. `executed_paths` shows whether continuity came from deterministic lookup, discovery, or both. `candidate_summary` records how the candidate set narrowed without storing rejected text. `selected_contexts` keeps only the record ID, version, and prompt usage of memory that actually reached the model. `revalidation` explains whether freshness was required and how that requirement resolved.

The standard trace should not duplicate rejected candidate text, complete cards, or the full prompt. Per-candidate diagnostics and prompt snapshots belong in isolated, tightly authorized troubleshooting storage with a shorter retention period.

Retrieval evaluation should extend beyond relevance: known-scope accuracy, contribution of each discovery mode, missing-source rate, second-pass drop reasons, stale-memory usage, revalidation success, category-budget utilization, useful memory tokens, and cleanup latency all expose different failure modes.

### 5. Storage and Indexes: Keep Authoritative State Separate from Discovery

This section answers how memory can remain authoritative while still being discoverable. The authoritative store needs transactions, record versions, lifecycle state, and reverse provenance lookup. Search systems optimize discovery. Separating them introduces eventual consistency, but it creates a much cleaner failure boundary: an index may lag, fail, or be rebuilt without corrupting durable state.

Index payloads should remain minimal: stable record pointer, required scope filters, embedding, source-record version, and index version. Full memory text, raw conversations, and source documents remain in the authoritative store so that the search layer does not become a second unmanaged corpus.

Each derived index should be versioned independently. A new tokenizer, embedding model, semantic-role mapping, or filter schema should produce a new physical version rather than mutate the active index in place:

```text
[Eligible Active Records]
-> [Build New Index Version]
-> [Validate Counts + Scope Filters + Sample Retrieval]
-> [Switch Read Alias]
-> [Retire Previous Version]
```

Each step has a distinct purpose. `Eligible Active Records` limits rebuild input to records currently allowed for indexing. `Build New Index Version` creates the replacement out of band. `Validate Counts + Scope Filters + Sample Retrieval` checks coverage, boundary filters, and representative queries. `Switch Read Alias` moves live reads to the new version. `Retire Previous Version` removes the old index only after the new one is stable.

An index entry can retain the source-record version and a fingerprint of its embedding input. Those values identify which vectors need regeneration after a record update. Rebuild jobs include only records currently eligible for indexing; deleted and superseded records are excluded, while stale records follow type-specific policy.

The cost is additional outbox processing, retries, reconciliation, and alias management. The benefit is that model upgrades, search migrations, deletion repair, and disaster recovery do not require rewriting authoritative memory data.

## The Hard Part Is Not Retrieval, But Risk Boundaries

Risk control should be embedded in the write, read, and deletion paths rather than treated as a separate launch checklist. The key boundaries are who may read memory, what content may become durable, how provenance remains explainable without copying everything, how deletion propagates, and how the system expands safely.

### 1. Authorization and Scope

This controls whether a memory is visible for the current request. Scope is resolved from the server-side identity and authorization chain. Callers and models may provide context hints, but they cannot broaden the accessible set. Uncertain authorization should fail closed and record only a minimal reason code.

### 2. Prompt Injection

This controls whether content can alter system rules. Conversations, documents, tool output, and memory content are untrusted data. Statements such as “remember that I am an administrator” or “skip future validation” must be blocked from durable memory and must never alter runtime policy.

### 3. Minimal Provenance and Audit

This keeps the system explainable without creating another sensitive corpus. Provenance should retain source identifiers, versions, policy and schema versions, hashes, and promotion decisions, not duplicate full sensitive content. Audit logs record who performed an action and when; provenance records how the memory was formed. Both are needed, but they serve different purposes.

### 4. Deletion Propagation and Index Rebuild

This prevents old content from returning through derived copies. A deletion request should traverse source events, completed turns, summaries, candidates, durable records, indexes, and caches. The authoritative record changes first; idempotent asynchronous tasks then remove or update derived views. `deleted`, `stale`, and `superseded` must remain distinct lifecycle states: `deleted` means the record can no longer be retained for ordinary use and should be removed from indexes and caches; `stale` means provenance may remain, but the content cannot be used as current context; `superseded` means a newer version has replaced it and the older record is kept only for traceability.

### 5. Progressive Rollout

This keeps complexity from expanding faster than governance. Start with one memory type and one visible workflow, such as restoring the state of the current document through deterministic lookup. Add preferences, open work, and scoped context next. Introduce research reuse, hybrid discovery, and revalidation only after the simpler lifecycle is stable. Historical backfill should begin with a dry run rather than an automatic migration of all old conversations.

## What the System Should Look Like When It Works

This is an architectural design, not a claim of unpublished production performance, so it intentionally avoids invented benchmarks. The outcomes below are engineering goals that should be validated, not performance promises.

The expected outcome is a system that can continue work across sessions without treating the entire conversation history as durable truth. It should reduce repeated setup, isolate temporary instructions from stable preferences, prevent cross-workspace contamination, provide a clear explanation for every memory used, and make deletion or index migration operationally recoverable. The prompt should contain less history but more useful history, leaving explicit room for the current request, fresh tool or knowledge results, and the model's answer.

The design should still be validated with offline evaluation, shadow traffic, and staged rollout. Acceptance should cover more than relevance: low false-promotion rates, zero unauthorized cross-scope retrieval, controlled stale-memory reuse, successful source-of-truth resolution, and bounded deletion propagation time.

## Engineering Lessons

The lessons are design tradeoffs: make boundaries, provenance, and lifecycle correct first, then improve recall and automation.

1. Memory is a governed state model, not a synonym for chat history.
2. Type determines behavior; retention is only one attribute.
3. Let models extract candidates, but keep persistence and authorization deterministic.
4. Apply scope before relevance ranking.
5. Treat every index as a rebuildable projection.
6. Use historical research as a map, not as current evidence.
7. Design deletion, correction, and reindexing from the first release.
8. A narrow end-to-end memory loop is more valuable than a broad but ungoverned platform.

## Back to One Sentence

A useful enterprise memory layer sits between transient conversation context and durable business state. It extracts only selected information, assigns an explicit type and scope, preserves provenance, and re-evaluates every record before it reaches the model again.

Typed memory, candidate promotion, source-of-truth retrieval, bounded context cards, asynchronous indexing, revalidation, and deletion propagation are not independent features. Together, they make memory explainable, bounded, and reversible. That is the minimum architecture required for an AI assistant to remain stateful without becoming uncontrolled.

## References and Acknowledgements

This article was inspired in part by the Oracle Developers Blog article [*From RAG to Memory Systems: Building Stateful AI Architecture*](https://blogs.oracle.com/developers/from-rag-to-memory-systems-building-stateful-ai-architecture), published on June 4, 2026. It builds on ideas such as memory manager, typed memory, promotion gate, per-turn prompt reassembly, trace/replay, and filter-before-ranking, and applies them to enterprise AI assistant continuity, governance, revalidation, and deletion propagation.

This article was written with AI assistance under the author's direction. GPT 5.5 and GPT 5.6 both supported drafting, structuring, and later refinement, while the author reviewed, edited, and finalized the arguments, architecture choices, examples, and public-release wording.