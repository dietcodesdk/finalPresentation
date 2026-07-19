# Recommendation Engine Blueprint

## Purpose

This repository is a notebook-driven recommendation and evidence engine for sustainability-oriented signal detection, retrieval, scoring, and summarization.

At a high level, the system does two things:

1. It classifies raw company evidence into progressively richer business and sustainability labels.
2. It runs a retrieval layer across web search and a local chunk knowledge base, then uses those retrieval outputs to score signals, rank evidence, and generate solution-level summaries.

The project is designed for analyst-style execution rather than as a packaged service. The notebooks are operator workflows. The helper modules under `src/helper_functions` hold most of the reusable logic.

## Canonical Runtime Entry Points

There are three notebook entry points in the repo:

- `src/notebooks/classification_flow_v1.ipynb`
  - Main end-to-end workflow.
  - Starts from raw input data.
  - Runs multi-stage classification.
  - Runs the RAG/websearch orchestration.
  - Continues into chunk-relevance scoring, weightage, and summary generation.

- `src/notebooks/kb_setup_creation.ipynb`
  - Standalone notebook for knowledge-base setup, CSV ingestion, collection inspection, and ad hoc chunk search.
  - Useful for validating vector-store setup in isolation.

- `src/notebooks/unified_orchestrator.ipynb`
  - Reference notebook for the web + chunk orchestration layer.
  - Important for understanding the original phase order and schema expectations.

After the recent refactor, the main operator path is:

`classification_flow_v1.ipynb` -> helper layer -> web + chunk retrieval -> downstream signal scoring and summaries

## Source of Truth for Configuration

The canonical top-level config is:

- `config/user_instructions.yaml`

This file drives both:

- the classification pipeline
- the RAG/websearch orchestration

It points to nested config files:

- `config/classification_yamls/*`
- `config/rag_websearch/web_search_config.yaml`
- `config/rag_websearch/chunk_retrieval_config.yaml`
- `config/rag_websearch/signal_registry.yaml`
- `config/rag_websearch/solution_registry.yaml`

Important detail:

- `config/rag_websearch/orchestrator_config.yaml` exists, but the active helper layer defaults to `config/user_instructions.yaml`.
- If behavior differs from older notebook docs, trust the helper code and `config/user_instructions.yaml`.
- The signal registry is treated as the master signal catalogue; the solution registry defines the active run scope by listing the signal IDs that should participate in a run.
- The runtime writes a compact registry-scope manifest to `outputs.registry_scope_manifest`, defaulting to `web_retrieval/registry_scope_manifest.json`.

## Path Resolution Model

The repo now uses one explicit path model.

- `src/runtime.py` is the only shared source of truth for:
  - `REPO_ROOT`
  - `CONFIG_ROOT`
  - `DATA_ROOT`
  - `resolve_repo_path(...)`
  - `resolve_data_path(...)`
- notebooks discover repo root only in their first bootstrap cell so `from src...` imports work from either repo root or `src/notebooks`
- helper modules do not use `os.chdir(...)` as a control mechanism
- boundary loaders resolve relative user/config/output paths up front
- leaf modules are expected to consume already-resolved absolute paths

In practice this means:

- `classification_helper_functions.py` normalizes top-level classification YAML paths
- `unified_orchestrator_lib.py` normalizes orchestrator config, registry, workbook, chunk-source, Chroma, and chunk-output paths
- `rag/config_loader.py` reads an already-resolved config file path instead of rediscovering repo structure

## Current Checked-In Operating Profile

Current defaults in the checked-in branch:

- `company_in_focus`: `CVC Capital`
- client/competitor list: `CVC Capital`, `Blackstone`, `KKR & Co. Inc.`, `EQT Group`
- signal registry count: `15`
- active signal count from solution registry: `15`
- signal type split:
  - `3` pre-check
  - `2` trigger
  - `10` scalar
- web competitor-focus signals: `2`
- solution count: `3`
- sub-solution count: `4`

Current default outputs:

- precheck workbook: `web_retrieval/signal_answer_results.xlsx`
- trigger workbook: `web_retrieval/signal_trigger_results_cvc.xlsx`
- web retrieval workbook: `web_retrieval/web_retrieval_results_cvc.xlsx`
- chunk retrieval workbook directory: `data/evidence_reports/hex/`
- registry-scope manifest: `web_retrieval/registry_scope_manifest.json`

## Architecture Overview

The system is split into five layers.

### 1. Notebook orchestration layer

Primary notebook:

- `src/notebooks/classification_flow_v1.ipynb`

Responsibilities:

- load raw input
- run stage-wise classification
- prepare metadata for retrieval
- invoke the RAG/websearch helper bridge
- continue into chunk-relevance scoring
- generate signal-level and sub-solution-level summaries

### 2. Classification helper layer

Primary module:

- `src/helper_functions/classification_helper_functions.py`

Responsibilities:

- YAML loading
- classification taxonomy lookup
- prompt building
- placeholder extraction/filling
- OpenAI classification calls
- confidence scoring
- multi-company dataframe expansion
- Dask-based batch execution

This layer powers the main classification stages:

- sustainability mention
- client mention
- internal/external
- sustainability topic
- author archetype
- sentiment
- chunk relevance
- optional trigger-check logic in the notebook

### 3. Theme and weightage layers

Modules:

- `src/helper_functions/theme_finder_helper_functions.py`
- `src/helper_functions/weightage_helper_functions.py`

Responsibilities:

- extract and consolidate themes from classified text
- apply YAML-driven rule scoring
- compute weighted signal strength and downstream ranking features

### 4. Unified orchestration layer

Primary modules:

- `src/helper_functions/unified_orchestrator_lib.py`
- `src/helper_functions/unified_orchestrator_models.py`
- `src/helper_functions/classification_rag_websearch_helper.py`
- `src/helper_functions/registry_scope.py`

Responsibilities:

- resolve config and path surfaces
- assemble repo-root-relative runtime config into absolute paths before execution
- translate `user_instructions.yaml` into runtime configs for web and chunk systems
- build a `RegistryContext` from the configured signal and solution registries
- derive per-stage `StageScope` objects for precheck, trigger, web retrieval, and chunk retrieval
- run precheck, trigger, web retrieval, and chunk retrieval in a controlled order
- enforce the manual `human_check` review gate
- provide merge utilities for trigger outputs and retrieval outputs
- maintain notebook-compatible dataframe handoff code, though the current bridge has that block disabled
- pass scoped signal lists and solution mappings into lower-level web execution explicitly; web-agent helpers do not self-load registry YAML fallbacks for scoped runs

### 5. Retrieval engines

Web retrieval:

- `src/helper_functions/web_agent_runner_lib.py`

Chunk retrieval:

- `src/helper_functions/rag/`

Responsibilities:

- web search via `openai-agents`
- structured result validation
- Excel export
- Chroma-backed vector storage
- semantic search
- lexical BM25 search
- weighted or RRF hybrid fusion
- cross-encoder reranking

## Detailed Module Map

### `classification_helper_functions.py`

This is the core classification runtime.

Key responsibilities:

- load user YAML and taxonomy YAMLs
- derive class labels by level and superclass
- construct prompts from reusable prompt fragments
- generate `Additional_input` columns
- run classification against raw text or chunk text
- run confidence scoring
- fan out across companies where needed
- orchestrate via Dask partitions

Most important execution function:

- `final_pipeline_to_run_along_dask(...)`

This is the main batch classification entrypoint used repeatedly by the notebook.

### `theme_finder_helper_functions.py`

This layer exists after the early classification stages.

It:

- merges theme-finder config with user config
- chunks text for theme extraction
- calls OpenAI JSON responses
- aggregates chunk-level themes back to source-row outputs

### `weightage_helper_functions.py`

This is a config-driven scoring layer.

It:

- resolves `{client_company}` placeholders in YAML
- normalizes source columns
- applies numeric and string matching rules
- writes score columns back into the dataframe

### `web_agent_runner_lib.py`

This is the web execution engine.

It:

- reads web config
- configures the OpenAI API key
- builds `openai-agents` agents
- expands the signal registry into batch jobs
- runs jobs concurrently within a stage
- normalizes sources, URLs, dates, and structured outputs
- writes `signal_answers`, `trigger_answers`, and `results` Excel sheets
- renders stage instructions from `config/rag_websearch/web_search_config.yaml`
- applies the shared `min_source_year` cutoff in trigger and retrieval prompts
- injects per-signal competitor guidance when `signal_retrieval.sources.web_search.focus_competitor` is true
- exports web dates in `YYYY/MM/DD` format
- populates trigger sources from URL citations when available, falling back to broader consulted sources
- retries parse-like and transient API/runtime failures up to three total attempts using exception classes, status codes, and structured SDK metadata rather than message substring markers
- logs final failed jobs to `web_retrieval/web_agent_failures.jsonl` without changing workbook schemas
- logs successful web retrieval jobs with zero final evidence rows to `web_retrieval/web_agent_no_results.jsonl` without adding workbook rows or columns

### `unified_orchestrator_lib.py`

This is the integration backbone.

It:

- resolves root-relative config paths
- builds stage-specific web configs
- builds chunk runtime config
- injects company and competitor context
- validates review workbooks
- computes exclusion scope from `human_check`
- handles precheck resume semantics
- runs trigger phase
- runs web retrieval phase
- runs chunk retrieval phase
- merges trigger outputs into web/chunk outputs
- exposes KB setup, ingestion, collection-inspection, and chunk-search utilities

### `classification_rag_websearch_helper.py`

This is the notebook bridge added for `classification_flow_v1.ipynb`.

It:

- builds notebook context from `config/user_instructions.yaml`
- runs precheck with the correct schema
- raises the manual review pause
- runs post-review orchestration in order
- emits ordered stage logs
- contains normalization logic for `df` and `websearch_df`, but the current post-review function does not populate those variables because the merge/handoff block is disabled

### `rag/` package

Key components:

- `pipeline.py`
  - top-level `RAGPipeline`
- `chunking.py`
  - CSV loading and optional text chunking
- `vector_store.py`
  - Chroma integration
- `search.py`
  - semantic, lexical, hybrid retrieval
- `helper.py`
  - metadata filter construction and signal/solution expansion
- `embeddings.py`
  - OpenAI embedding generation
- `reranking.py`
  - cross-encoder reranking
- `filters.py`, `dates.py`, `config_loader.py`
  - normalization and config helpers

## End-to-End Logical Flow

### Phase A. Core classification flow

In `classification_flow_v1.ipynb`, the first half of the notebook processes raw input rows.

The stages are:

1. load classification config
2. load input Excel
3. run sustainability mention classification
4. split and refine client mention logic
5. run internal/external classification
6. run sustainability topic classification
7. run author archetype
8. run sentiment
9. run theme finder
10. run weightage
11. run ESG mapping

The output of this phase is a classification-rich dataframe.

### Phase B. Prepare retrieval metadata

The notebook then adds metadata columns needed by the retrieval layer, especially company-filter-related columns that later support chunk retrieval and post-retrieval scoring.

### Phase C. RAG/websearch orchestration

The RAG/websearch section now contains exactly four code cells:

1. import helper + build context
2. run precheck
3. pause for manual review
4. run post-review orchestration

The import/context cell exposes `POST_REVIEW_STAGE`, which may be one of:

- `all`
- `trigger`
- `retrieval`
- `chunk`

This selector is intentionally simple and does not support arbitrary stage combinations.

The post-review orchestration order is intentionally sequential:

1. validate reviewed precheck workbook
2. compute exclusions from `human_check`
3. set up KB collection
4. ingest KB CSV, skipping duplicate `chunk_id` values
5. run trigger check
6. run web retrieval
7. run chunk retrieval

No blind parallelization is used across those top-level stages.

Current implementation note:

- trigger, retrieval, and chunk stages can be run independently through `stage_to_run`
- `classification_rag_websearch_helper.py` still contains the trigger merge and dataframe handoff logic, but that block is currently disabled
- the helper currently returns stage outputs without repopulating `df` or `websearch_df`
- downstream chunk-relevance cells still expect `df` and `websearch_df` if they are run

### Phase D. Chunk relevance scoring

Once the retrieval handoff dataframes are available, the rest of the notebook remains unchanged.

It:

- deduplicates retrieved chunk rows per signal by `doc_id`
- adds signal-specific instructions
- recency-filters retrieval rows
- classifies chunk relevance for KB and web rows
- extracts logic/evidence from rationale JSON
- concatenates RAG + websearch relevance outputs
- runs weightage
- computes signal strength
- produces summary sheets

### Phase E. Summary generation

The final part of the notebook:

- aggregates top evidence per signal
- produces signal-level summaries
- produces sub-solution summaries
- merges summaries back into final outputs

## Human-in-the-Loop Control

The precheck workbook is not just an export. It is a control artifact.

Mechanism:

- precheck writes rows to `web_retrieval/signal_answer_results.xlsx`
- operator fills `human_check`
- rows with `human_check == 0` seed downstream exclusions
- exclusions are expanded by sub-solution mapping, not by naive global signal removal

This review gate is essential to the architecture.

## KB Setup and Ingestion Logic

KB setup and ingestion are designed to be idempotent enough for repeated notebook use.

Current behavior:

- setup can recreate the collection only if explicitly requested
- ingestion loads documents from the configured CSV
- existing document IDs are read from the vector store
- only unseen `chunk_id` values are ingested
- existing IDs are skipped

Important limitation:

- ingestion is insert-only by ID
- existing IDs are not refreshed if text or metadata changed
- if the source CSV changes for an existing `chunk_id`, a recreate or separate refresh mechanism is required

## Retrieval Design

### Registry context and stage scope

The RAG/websearch orchestration interprets registries through `src/helper_functions/registry_scope.py`.

- `RegistryContext` loads the configured signal and solution registries once per orchestration call.
- The signal registry is the catalogue of available signal definitions.
- The solution registry is the active run scope; only referenced signal IDs are eligible to run.
- Unknown solution-registry signal IDs fail at runtime.
- Signal-registry entries that are not referenced by the solution registry are inactive diagnostics.
- Runtime helpers outside `registry_scope.py` must not parse signal or solution registry YAMLs for execution. They consume `RegistryContext`, `StageScope`, or explicit mappings derived from a `StageScope`.
- `StageScope` applies stage policy on top of the active scope:
  - precheck uses active `pre-check` signals
  - trigger uses active `trigger` signals
  - web retrieval uses active scalar signals with enabled web search and query templates
  - chunk retrieval uses active scalar signals with enabled RAG and hybrid queries

The manifest at `outputs.registry_scope_manifest` records active IDs, stage runnable IDs, exclusions, and skip reasons for stages that actually executed. Precheck resets this debug manifest for a fresh run, then later stages upsert their own execution scopes into the same file.

### Web retrieval

The web pipeline is registry-scope driven.

The signal registry provides:

- signal IDs
- signal names and descriptions
- signal type
- web query templates
- optional `signal_retrieval.sources.web_search.focus_competitor`

The stage scope provides the actual signal list and solution/sub-solution mappings used for each web stage.
Lower-level web-agent execution requires those scoped signals and mappings to be passed explicitly. Missing scoped inputs are runtime errors rather than triggers to reload registry YAML paths.

The web config provides shared prompt/runtime controls:

- `shared.run.min_source_year`
- `shared.run.competitor_focus_template`

Trigger and retrieval instructions include placeholders for those controls:

- `{min_source_year}`
- `{competitor_focus_instruction}`

Competitor guidance is prompt-only. It does not alter query templates and is enabled per signal by `web_search.focus_competitor`, not by RAG metadata filters.

The web stage writes structured outputs to Excel and keeps source metadata. Trigger dates and retrieval published dates are normalized to `YYYY/MM/DD`.

### Chunk retrieval

Chunk retrieval uses:

- the chunk `StageScope`
- scoped solution lookup/entries derived from `chunk_scope.signal_to_solution_mappings`
- the chunk source config
- global chunk search config
- per-signal metadata filters

Search modes:

- semantic
- lexical
- hybrid

Current branch behavior uses hybrid retrieval plus reranking in the unified phase.

## Data Artifacts

### Input artifacts

- raw input Excel
- chunk source CSV
- classification YAMLs
- web config
- chunk config
- signal registry
- solution registry

### Intermediate artifacts

- classification dataframes in notebook memory
- precheck workbook
- trigger workbook
- web retrieval workbook
- chunk retrieval workbook
- registry-scope manifest JSON

### Persistent stores

- Chroma data under `data/processed/chroma_db_dynamic_companies_check`

## Important Invariants

- `config/user_instructions.yaml` is the active top-level config
- `src/runtime.py` is the shared repo-root authority for Python helpers
- notebooks should bootstrap imports with `sys.path` insertion, not `os.chdir(...)`
- RAG/websearch notebook orchestration must preserve manual `human_check`
- RAG/websearch stages must use `RegistryContext`/`StageScope` rather than treating the full signal registry as the run list
- RAG/chunk dataframe helpers require scoped solution mappings explicitly; missing mappings are errors, not fallback prompts to reload registry paths
- solution-registry signal references are the active run scope and must resolve to signal-registry definitions
- `classification_flow_v1.ipynb` downstream of the RAG/websearch section is intentionally left stable, but currently still depends on `df` and `websearch_df` being available before continuing
- `df` and `websearch_df` are contract variables for the later chunk-relevance section
- checked-in relative chunk output and vector-store paths resolve under `data/`
- trigger merge must happen before downstream logic that expects `trigger_check` in retrieval-derived tables
- trigger and retrieval web prompts share a single source cutoff from `web_search_config.yaml`
- competitor-focused web behavior is controlled only by `signal_retrieval.sources.web_search.focus_competitor`
- trigger source exports prefer citations from run metadata, with consulted sources as fallback
- web-exported dates should use `YYYY/MM/DD`

## Known Design Tradeoffs

- notebooks are convenient for operators but make hidden state easier to accumulate
- the project mixes research workflow logic and production-like helper modules
- config is powerful but spread across many YAMLs
- insert-only ingestion is safe for duplicates but not for source refresh
- manual review improves control but breaks full automation

## Best Short Context for an LLM

If a model needs a one-shot summary of the repo, the shortest correct framing is:

This is a notebook-driven recommendation engine that first classifies raw sustainability/company evidence, then runs a staged retrieval pipeline across web search and a local Chroma-backed chunk store. The top-level config is `config/user_instructions.yaml`. The main notebook is `src/notebooks/classification_flow_v1.ipynb`. Classification logic lives in `classification_helper_functions.py`, retrieval orchestration in `unified_orchestrator_lib.py` and `classification_rag_websearch_helper.py`, web execution in `web_agent_runner_lib.py`, and vector retrieval in `src/helper_functions/rag/`. The RAG/websearch flow is sequential: precheck -> manual review -> KB setup -> KB ingestion -> trigger -> web retrieval -> chunk retrieval -> trigger merge -> chunk relevance scoring -> weighting -> summaries.
