# Federal File and Metadata Extraction and Migration
## An AI-Powered Multi-Agent Pipeline Case Study

**By Jennifer Serafini**  
*Principal Software Engineer, Architect & Technical Lead*

---

## Executive Summary

A federal research agency faced a critical mandate to modernize a system used for scientific publication peer review, a foundational repository for hierarchically organized scientific files built on end-of-life legacy technology. I was tasked with designing and building the modernization architecture from the ground up. The project involved migrating over 100,000 files whose metadata was inconsistently applied, entrenched in filenames and file content, and had never been intentionally classified.

I designed and built an AI-powered, multi-agent pipeline alongside a modernized iRODS and API-driven target architecture. The pipeline enforces safety by architecture rather than by policy. This was a deliberate design choice grounded in the understanding that LLMs violate prompt-based restrictions at rates between 20 and 62 percent. Every guardrail is structural. The system decouples file storage from metadata management, ensuring scalability, data standardization, and future-proof accessibility.

---

## My Role

I served as Project Architect, Technical Lead, and primary Developer. I was selected for this initiative based on my technical breadth, my approach to cross-team communication, and my demonstrated ability to identify and mitigate critical system risks before they reach production.

This migration was designed to integrate directly with a major enterprise project I had previously led: the development of the target API, built on an Oracle and iRODS backend. Leading both initiatives allowed me to architect a seamless bridge between the legacy data ecosystem and the modern platform.

---

## The Challenge

The legacy system relied on a complex Oracle database schema where file metadata was deeply distributed across a rich hierarchy of organizational containers. The core challenges were significant on multiple fronts.

**Scale and data inconsistency.** Over 100,000 active files carried metadata that was inconsistently applied and entrenched in file naming conventions and file content rather than intentionally classified. No controlled vocabulary had ever been enforced. Metadata lived in Entity-Attribute-Value models, free-text fields, and embedded naming patterns that varied across decades of submissions.

**Technological obsolescence.** The frontend was approaching end-of-life, creating immediate security and maintenance exposure.

**Accuracy at scale.** Processing a large, continuously growing dataset without resource exhaustion while strictly preserving complex parent-child relationships and historical access controls required careful architectural planning.

**Evolving requirements.** Scope changes during implementation required the architecture to adapt while preserving the integrity of what had already been built and validated.

---

## The Solution

### Target Architecture: iRODS + API

Physical file binaries remain on their reliable Network Attached Storage, mounted via an iRODS data virtualization layer. The API, which I previously designed and built on an Oracle and iRODS backend, serves as the central hub for all metadata. This completely decouples storage from presentation and transitions the workflow to a metadata-driven model aligned with the target Data Dictionary.

### The AI-Powered Legacy Data Pipeline

To safely extract and map legacy Oracle data across 100,000+ files, I designed a containerized, multi-agent pipeline orchestrated by a deterministic Python control plane. The control plane treats LLMs as tools rather than decision-makers. Every agent has a defined scope, explicit inputs, and validated outputs before anything proceeds.

---

## Pipeline Architecture

### Assessor Agent
Analyzes the live Oracle database and produces a rich JSON knowledge base of the schema, including table structures, column definitions, and relationships. This knowledge base becomes the ground truth that every downstream agent validates against.

### Architect Agent
Reads the migration specification documents and chunks them into vector embeddings stored in ChromaDB. At runtime, the pipeline retrieves the current business rules from ChromaDB to govern how legacy data maps to the target schema. The pipeline stays synchronized with specification changes without manual rule updates because the retrieval layer always reflects the current version of the spec.

### Inventory Builder
The deterministic core of the pipeline. Uses hardcoded SQL queries against the legacy schema, executed in batches to prevent memory exhaustion and ensure predictable, auditable behavior. The batched approach processes records in manageable pages rather than attempting full-dataset extraction, eliminating resource exhaustion risk at scale.

### LLM-as-Judge Validation Agent
A second agent validates every table and column name in the generated SQL against the schema assessment produced by the Assessor Agent before any query is executed against live data. If the generating agent references a schema element that does not exist in the actual database, the validation agent catches it before execution. The system cannot proceed on a query that references nonexistent schema elements regardless of how confidently it was generated. This is the architectural answer to the hallucination problem, not a prompt-level workaround.

### QC Validator
Provides automated quality control by comparing pipeline output against baseline analyst reports. Validates file counts, columnar data, and mapping accuracy before human review. A dashboard surfaces validation results in real time so the migration state is visible at any point during the run, queryable by file, by batch, or by phase.

### Human-in-the-Loop Checkpoints
The pipeline includes mandatory human review gates before consequential operations proceed. After inventory validation, a human reviews the results and the QC output before the migration manifest is generated. After manifest generation, a second human review occurs before migration begins. The system surfaces the information needed for those decisions. It does not make them autonomously.

---

## Credentialing and Safety Architecture

Safety is enforced by architecture, not by prompts or policy.

**Strict least-privilege credentialing.** The pipeline connects to the legacy Oracle database using a strictly read-only user account for all schema inspection and query generation. A hallucinating model with read-only credentials cannot corrupt data regardless of what it generates.

**Isolated write operations.** Write permissions are provided only for narrowly scoped write functions through a separate credential. The LLM agents never have access to write credentials.

**Hard-coded operation blocks.** Programmatic constraints prevent destructive operations from executing under any circumstances. DROP TABLE, TRUNCATE, and similar commands are blocked at the architecture layer. These are not prompt-level instructions. They are structural blocks that cannot be overridden by model output.

**Dynamic column validation.** Before any insert operation, the pipeline queries the actual database schema to confirm that target columns exist. A hallucinated column name fails validation and stops the pipeline rather than producing a corrupted row.

**Retry queue and dead letter pattern.** Failed rows are retried up to three times. Hard failures after retry are written to a dead letter table with full context, including what was attempted, what failed, and the complete row data. No partial inserts are permitted. The migration continues past a hard failure without silent data loss, and the dead letter table provides a complete, auditable record of everything that requires investigation.

*Note: The original architecture included transactional batch rollback at the study level. Scope changes during implementation shifted the failure handling to the retry queue and dead letter pattern described above. The design principle is consistent across both approaches: no partial inserts, no silent failures.*

---

## Results and Impact

**Standardization at scale.** Over 100,000 files with inconsistently applied, unclassified legacy metadata were systematically parsed, cleaned, and mapped to a strict Controlled Vocabulary.

**Preserved context.** Redundant legacy structures were flattened while preserving critical provenance, audit histories, and hierarchical parent-child relationships.

**Resilience and security.** Strict architectural guardrails and a containerized pipeline provide reproducible, deterministic results without risk to legacy data integrity. The pipeline has never produced a partial insert or a silent failure.

**Institutional visibility.** The real-time observability dashboard gave the client and the team continuous visibility into migration state across every phase, queryable by file, by batch, or historically by completed phase. This was a deliberate trust-building design decision, not an afterthought.

---

## Key Architectural Decisions

**Why a deterministic Python control plane rather than an orchestration framework.** LLM orchestration frameworks abstract control flow in ways that make failure modes harder to reason about. A deterministic control plane makes every transition explicit, every failure visible, and every rollback path defined. In a federal data environment where getting it wrong has real consequences, that transparency is not optional.

**Why readonly credentials rather than prompt-level restrictions.** Models violate prompt-based restrictions at rates between 20 and 62 percent. A readonly database user cannot execute a write operation regardless of what the model generates. The architecture enforces the constraint. The prompt does not need to.

**Why human-in-the-loop checkpoints rather than fully autonomous execution.** An agent that can approve its own actions is not an agent with an approval step. The human review gates are positioned where human judgment adds the most value, after inventory validation and after manifest generation, and are enforced structurally. The pipeline cannot proceed past those points without explicit approval.

---

*Jennifer Serafini | github.com/Jenjineer | linkedin.com/in/jserafini-principalsoftwareengineer*
