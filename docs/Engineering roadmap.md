# Engineering Roadmap: Integrating RFC-006 & RFC-007 Deterministic Gates into `Borealiscodes/synapse`

## Objective
Adapt `Borealiscodes/synapse` from a standard RAG/synthesis workspace into a forensically auditable legal and administrative case-analysis pipeline implementing **RFC-006** (*Institutional Enclosure Geometry*) and **RFC-007** (*Decoupling Attribution from Inference*).

### Core Invariants
1. **Pre-LLM Provenance:** Every source file and text chunk must be hashed (`SHA-256`) and locked into an immutable manifest *before* embedding or LLM inference.
2. **Quarantined Inference (`Record ≠ Admit ≠ Commit`):** LLM outputs are strictly untrusted proposals until validated by deterministic CPU checks (`deterministic-ai-gate`) and admitted across the commit seam (`active-inference-commit-seam`).
3. **Zero External Telemetry:** All ingestion, embedding, and inference must execute locally (Ollama / local embeddings / SurrealDB).

---

## Phase 1: Pre-Ingestion & Cryptographic Manifests (RFC-007 Layer 1)
**Target Files:** `open_notebook/utils/chunking.py`, `open_notebook/graphs/source.py`, `commands/source_commands.py`

* [ ] **1.1 Raw Artifact Hashing**
  * Intercept file upload/ingestion prior to `content-core` extraction.
  * Compute `SHA-256` of the raw binary file (e.g., PDF exhibit prepared via `pdf-organiser`) and record byte size, timestamp, and source filename.
* [ ] **1.2 Deterministic Chunk Manifesting**
  * Modify `open_notebook/utils/chunking.py` to enforce deterministic byte/character offsets alongside `OPEN_NOTEBOOK_CHUNK_SIZE`.
  * For every generated chunk, compute `chunk_sha256`, `start_offset`, `end_offset`, and `page_number` (where applicable).
  * Emit an immutable `manifest.json` per source artifact and store its root hash in SurrealDB (`source` table).
* [ ] **1.3 Gate Vectorization on Manifest Lock**
  * Update `open_notebook/graphs/source.py` so vector embeddings (`source_embedding`) cannot be written unless a valid `manifest_sha256` exists on the parent `Source` record.

---

## Phase 2: Schema-Bounded Extraction & Deterministic CPU Gate (RFC-007 Layer 2)
**Target Files:** `open_notebook/graphs/transformation.py`, `open_notebook/graphs/ask.py`, `open_notebook/domain/notebook.py`

* [ ] **2.1 Integrate `deterministic-ai-gate` Dependency**
  * Add `deterministic-ai-gate` to `pyproject.toml` and expose its verification primitives to the LangGraph runtime.
* [ ] **2.2 Replace Free-Form Transformations with Schema-Bounded Proposals**
  * Create a forensic extraction transformation (`forensic_claim_extract`) in `open_notebook/graphs/transformation.py`.
  * Constrain LLM output to strict JSON schemas requiring:
    * `claim_text`
    * `exact_verbatim_quote`
    * `claimed_chunk_id`
    * `claimed_timestamp` / `actor` / `institutional_action`
* [ ] **2.3 Add Deterministic Verification Node (`verify_proposal_node`)**
  * Insert a non-LLM LangGraph node immediately following LLM generation.
  * Run `deterministic-ai-gate` CPU checks:
    * Verify `exact_verbatim_quote` exists byte-for-byte inside the hashed chunk referenced by `claimed_chunk_id`.
    * Verify extracted dates/timelines conform to ISO-8601 and fall within valid bounds.
    * Tag each proposal with `verification_status`: `VERIFIED`, `QUOTE_MISMATCH`, `SCHEMA_FAIL`, or `ORPHAN_CITATION`.

---

## Phase 3: Active Inference Commit Seam & SurrealDB Quarantine (RFC-007 Layer 3)
**Target Files:** `migrations/`, `open_notebook/database/`, `open_notebook/domain/`

* [ ] **3.1 Enforce `Record ≠ Admit ≠ Commit` in SurrealDB**
  * Add a new SurrealDB migration separating raw records from admitted facts:
    * `raw_record`: Immutable ingested artifacts + `manifest_sha256`.
    * `proposed_claim`: Quarantined LLM extractions + deterministic gate audit logs.
    * `committed_fact`: Verified claims admitted across the commit seam.
  * Restrict `RELATE` edges (`references`, `supports`, `contradicts`) in the knowledge graph so only `committed_fact` nodes can form causal edges.
* [ ] **3.2 Wire `active-inference-commit-seam`**
  * Implement a commit controller between `verify_proposal_node` and SurrealDB persistence.
  * Block automatic database writes for any proposal failing deterministic checks; route failures to a human-in-the-loop review queue in the Next.js frontend.

---

## Phase 4: Institutional Enclosure & CAUSA Graph Modeling (RFC-006 Layer)
**Target Files:** `open_notebook/graphs/chat.py`, `frontend/` (Next.js UI)

* [ ] **4.1 Enclosure Geometry Schema**
  * Extend SurrealDB graph relations to model RFC-006 structural primitives on `committed_fact` nodes:
    * `Actor` -> `encloses` / `deflects` / `delays` -> `ProceduralGate`
    * Timeline delta calculations (`latency_days`) computed deterministically in Python from verified timestamps.
* [ ] **4.2 Audit-First Chat & Citation UI**
  * Update `open_notebook/graphs/chat.py` so RAG retrieval queries **only** `committed_fact` and verified `raw_record` chunks—never unverified `proposed_claim` rows.
  * Display `chunk_sha256` and exact byte/page provenance badges alongside every citation in the Next.js UI.

---

## Phase 5: Verification & Reproducibility Test Suite
**Target Files:** `tests/`

* [ ] **5.1 Determinism & Tamper Tests**
  * Test that modifying 1 byte in a source PDF invalidates `manifest_sha256` and blocks downstream commit seam admission.
* [ ] **5.2 Hallucination Rejection Tests**
  * Feed synthetic LLM responses with subtly altered quotes or fabricated dates into `verify_proposal_node` and assert `100%` rejection prior to SurrealDB commit.
* [ ] **5.3 Offline Air-Gap Audit**
  * Run full Docker Compose stack (`SurrealDB` + `Ollama` + `Synapse`) with outbound network disabled to verify zero external telemetry.
