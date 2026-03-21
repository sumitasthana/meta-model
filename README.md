# Meta Model as the Central Intelligence Layer — Two-Phase Architecture

---

## 1. The Central Idea: Meta Model as the Anchor

**What the Meta Model is**

- The Meta Model is a graph database that holds a living, queryable representation of an organisation's code, data, and business rules in one unified structure
- It is not a documentation output and not a pipeline artifact — it is the operational source of truth
- Every other component in this solution exists in one of two roles: it either hydrates the graph or it consumes the graph
- The graph is the product; agents are its clients

**The two-phase boundary**

- Phase 1 is hydration — the graph is built once, on demand, by scanning and analysing the legacy codebase; this phase runs infrequently
- Phase 2 is runtime consumption — once hydrated, all scripts and pipelines pull source fields, transformation logic, and target column names dynamically from the graph at execution time
- The boundary between these two phases is the most important architectural line in the solution
- Nothing in Phase 2 reads source code, queries a config file, or hardcodes a column name — the graph is the only authoritative source

> **Core principle:** After hydration, the Meta Model is the schema registry, the transformation catalogue, and the lineage map — all in one.

---

## 2. What the Meta Model Must Represent

**Code structure layer**

- Repository, Module, File — the containment hierarchy, each node keyed on a stable path and file hash
- Class and Function — the behavioural units; functions are the primary nodes because they are what do things
- Argument — typed inputs to functions, preserving the callable contract for any agent that later generates or validates code
- Decorator — audit wrappers, retry logic, and compliance markers that affect function behaviour without appearing in its body

**Data interaction layer — the runtime-critical layer**

- SchemaObject — every source and target table and column the codebase reads or writes, regardless of access pattern
- READS and WRITES edges from Function to SchemaObject — this is what Phase 2 pipelines will query at runtime to know which fields to pull
- Transformation logic captured on DataFlow edges — source column, target column, and the transformation expression linking them
- This layer is what makes dynamic field resolution possible in Phase 2; without it, pipelines are still hardcoded

**Intent and governance layer**

- LLMSummary — generated description of a function's intent, timestamped and model-tagged, always marked as inferred
- BusinessRule — formalised rules governing how data must be handled, linked to the functions they constrain
- Risk tags on Function nodes — PII, audit_required, regulatory_report — queryable signals for routing and compliance
- GOVERNED_BY edges from Function to BusinessRule — the traceability chain an auditor or regulator can follow

**Lineage bridge**

- SchemaObject nodes are the join point between the code graph and the existing STM (source-to-target mapping) graph
- A single Cypher traversal goes from a Function node through WRITES to a SchemaObject, then through MAPS_TO into the STM lineage
- This bridge is what elevates the Meta Model from a code documentation tool to an end-to-end lineage surface

---

## 3. Phase 1 — Hydration (On-Demand, Runs Infrequently)

> **Phase 1 intent:** Scan the legacy codebase once, extract every structural and semantic fact that can be known, and write it into the Meta Model. The goal is completeness and accuracy, not speed.

**When Phase 1 runs**

- Initial setup — the first full scan of all legacy repositories to establish the baseline graph
- Post-migration — after a significant refactor or module rewrite, to update the graph with changed facts
- Compliance trigger — when a new regulatory requirement demands that the lineage graph be refreshed
- Not on every deploy, not on every commit — hydration is expensive and its output is meant to be stable

**Hydration agent 1 — AST extraction**

- Parses every Python file using the built-in ast module — no regex, no heuristics, no LLM at this stage
- Produces Repository, Module, File, Class, Function, and Argument nodes with full structural properties
- Computes a SHA-256 hash of each file — unchanged files are skipped on re-runs, making partial re-hydration safe
- Outputs a structured intermediate representation; does not write to Neo4j directly

**Hydration agent 2 — cross-reference resolution**

- Resolves import chains across module boundaries to build CALLS edges between Function nodes
- Uses a symbol resolver rather than name matching — name matching produces false positives in large codebases
- Produces ExternalDep nodes from dependency manifests and links them to importing functions via IMPORTS edges
- Unresolvable symbols are written as UNRESOLVED nodes — the graph records what it does not know

**Hydration agent 3 — schema and transformation extraction**

- This is the most critical hydration agent for Phase 2 — it populates the fields and logic that runtime pipelines will query
- Extracts SchemaObject nodes and READS/WRITES edges from raw SQL strings, SQLAlchemy models, and pandas data access patterns
- Captures transformation expressions from function bodies and writes them as DataFlow edges with source column, target column, and expression
- String-interpolated SQL is flagged as DYNAMIC_SQL — it is not a SchemaObject, and Phase 2 pipelines must handle it separately
- This agent is the bridge between the code graph and the STM lineage graph

**Hydration agent 4 — LLM summarisation**

- Runs last because it depends on structural facts already written by earlier agents
- Assembles a context packet per function by querying the graph — not by reading raw source in isolation
- The prompt separates verified graph facts from unverified source logic — this separation prevents hallucination from contaminating static facts
- Outputs LLMSummary nodes, supplementary DataFlow edges, and risk tag annotations — all written with `confidence: inferred`
- Runs only on functions that touch a SchemaObject or exceed 20 lines without a docstring — blanket summarisation is wasteful

**Graph writer — the single write surface**

- All hydration agents submit to a single graph writer rather than writing to Neo4j directly
- Every write is a MERGE keyed on a stable identifier — the pipeline can be re-run at any time without duplication or corruption
- Each write batch carries a `scan_run_id` — nodes not touched in a run are flagged DEPRECATED, never deleted
- Historical lineage is preserved — the graph answers questions about what used to be true, not just what is true now

---

## 4. Phase 2 — Runtime Consumption (Dynamic, Runs Continuously)

> **Phase 2 intent:** Once hydrated, the Meta Model becomes the runtime authority. All scripts resolve source fields, transformation logic, and target column names by querying the graph — nothing is hardcoded.

**The fundamental shift Phase 2 represents**

- Before the Meta Model: a pipeline script contains hardcoded source table names, column lists, and transformation expressions
- After the Meta Model: a pipeline script queries the graph at startup, receives the fields and logic it needs, and executes against them
- A schema change in the source system requires updating the graph — not editing pipeline scripts across dozens of files
- This is the operational value of the Meta Model; hydration is the cost, Phase 2 is the return

**How a Phase 2 pipeline resolves its inputs**

- At startup, the pipeline identifies itself by a pipeline ID or function name registered in the graph
- It queries the graph for all SchemaObject nodes connected to its registered Function node via READS edges — these are its source fields
- It queries DataFlow edges from those source fields to get transformation expressions and target column mappings
- It queries WRITES edges to confirm the target columns it is authorised to write to
- It executes using these dynamically resolved fields — no column name appears as a string literal in the script itself

**What dynamic resolution enables**

- A source column rename propagates automatically — update the SchemaObject node, all pipelines that READS it pick up the change on next run
- A new target column requirement is added by writing a new DataFlow edge to the graph — no pipeline code changes
- Audit trails are automatic — every pipeline execution queries the graph, so the graph knows which fields were active at what time
- New pipelines are registered in the graph first, then implemented — the graph is the specification, not a byproduct

**Consumption agent — compliance query**

- Issues Cypher traversals to answer regulatory questions that would otherwise require manual code review
- All functions writing to PII-classified columns without an audit decorator — a single graph traversal
- Full lineage from a regulatory report target column back through all source tables and the functions that populate them
- The answer quality is entirely a function of how completely and accurately Phase 1 hydrated the graph

**Consumption agent — impact analysis**

- Traverses READS, WRITES, CALLS, and IMPORTS edges to answer change-impact questions before a change is made
- Rename a source column — the agent flags every Function that READS it and every DataFlow that references it
- Upgrade an external package — the agent surfaces every function that imports it and every pipeline that calls those functions
- This agent makes the Meta Model useful during active development, not just during documentation or compliance cycles

**Consumption agent — code generation**

- Queries the graph to assemble a verified context packet per function: signature, reads, writes, calls, business rules
- Passes this packet to an LLM as the ground truth contract the generated code must satisfy
- After generation, re-runs the AST extractor on the output and diffs the resulting subgraph against the original to verify READS and WRITES are preserved
- READS and WRITES match is a hard assertion — if the generated function does not touch the same tables, it is wrong regardless of appearance
- Functions with PII, audit, or regulatory risk tags require human review regardless of generation quality — this is a compliance requirement, not a technical limitation

---

## 5. Graph Integrity — What Makes Both Phases Trustworthy

**Confidence labeling is a first-class schema concern**

- Every edge carries a `confidence` property — `static` for facts from AST extraction, `inferred` for facts from LLM agents
- Phase 2 pipelines resolving source fields must filter on `confidence: static` — inferred edges must never drive runtime field resolution
- This separation is what makes the graph safe to use as a runtime authority, not just an advisory documentation layer

**Idempotency is non-negotiable**

- Every write uses MERGE keyed on a stable content-derived identifier — never INSERT without a uniqueness check
- The file hash is the change signal — agents skip unchanged files on re-runs, making partial re-hydration safe and fast
- Running the hydration pipeline twice on the same codebase must produce the same graph — this is a testable invariant

**Deprecation over deletion**

- No node is ever deleted — nodes that no longer exist in source are marked DEPRECATED with a timestamp and a `replaced_by` pointer
- A DEPRECATED SchemaObject node tells Phase 2 pipelines that a field has been retired — they can handle it gracefully rather than failing
- A regulator asking about code or data pipelines from two years ago gets a complete answer because nothing has been erased

**Fail loud, never silent**

- An unresolvable import becomes an UNRESOLVED node — the graph knows the gap exists
- Dynamic SQL becomes a DYNAMIC_SQL flag — Phase 2 pipelines know they cannot rely on static field resolution for that function
- A generation failure gets a GENERATION_FAILED marker — the gap is queryable and actionable
- Silently dropping uncertain data creates false confidence in graph completeness, which is more dangerous than an acknowledged gap

---

## 6. The Proposition

**Phase 1 delivers**

- A complete, queryable graph of every source field, transformation expression, and target column across the legacy codebase
- Traceability from every pipeline function back to the business rules and regulatory requirements it must satisfy
- A bridge to the existing STM lineage graph, making code and data one unified queryable surface
- A one-time investment that does not need to be repeated unless the codebase undergoes significant structural change

**Phase 2 delivers**

- Pipelines that never hardcode a source field, transformation, or target column name — all resolved dynamically from the graph
- Schema changes that propagate through all pipelines by updating one graph node, not by editing dozens of scripts
- Compliance queries that run in seconds rather than requiring senior engineers to read code for days
- New pipelines that are specified in the graph first and implemented second — the Meta Model is the design authority

**The agents are replaceable; the graph is not**

- Every hydration agent — AST parser, schema extractor, LLM summariser — can be improved, replaced, or re-run without affecting the graph's consumers
- Every consumption agent — documentation generator, compliance checker, code generator — can be updated independently without re-hydrating
- Invest first in graph completeness and integrity; agent sophistication is secondary
- A richer, more accurate graph makes every existing and future agent more capable without any changes to the agents themselves

---

*Internal working document — Sumit Builds*
