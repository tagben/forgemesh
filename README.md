# ForgeMesh

A Python pipeline that automatically maps relational "bronze" tables to an OWL ontology and materialises the result through silver, gold, and RDF layers — producing a semantically-enriched knowledge graph, a structured audit trail, and a natural-language query interface over any relational schema.

Designed to work against any MySQL database and any OWL file, with no dataset-specific configuration in code.

**Launch write-up:** [I've open-sourced ForgeMesh](https://humous.substack.com/p/ive-open-sourced-forgemesh-plain): what it does, why it was built, and what isn't finished yet.

---

## Project status

**Early release (v0.1). Expect bugs.** ForgeMesh is a working research project, not a finished product. It has mostly been run by one person, on the datasets listed below.

What is solid:
- The pipeline (inspect → map → silver → gold → RDF) and its seven validation checks, on the four bundled project configurations.
- Row-level provenance in the graph: every triple carries `fm:bronzeSourceRef` and `fm:goldRunId`, linking it back through gold and silver to its source row.

Known gaps:
- **Answer-to-source drill-down.** Every answer gets a receipt (interpretation, query, sources, evidence, caveats), and for most answers it traces a sample of the rows behind the result back through gold and silver to the source rows. It doesn't yet open the rows behind a single figure in a grouped result ([#3](https://github.com/tagben/forgemesh/issues/3)), and queries using HAVING or subqueries stop at the graph ([#2](https://github.com/tagben/forgemesh/issues/2)).
- **LLM-written queries can be wrong.** The pipeline is validated; the query an LLM writes for a question it hasn't seen before is not. Check the query and the receipt. Known faulty query shapes are fixed automatically, and any rewrite is recorded in the receipt.
- **Gold SQL mode is weaker than graph mode.** Some joins between gold tables aren't possible yet, because a source table's primary key isn't always carried into gold.
- **MySQL is the only source database.**

**Maintenance.** I'm not actively developing ForgeMesh, so please don't expect reported issues to be fixed. Pull requests are welcome and I'll review them. The [open issues](https://github.com/tagben/forgemesh/issues) are a good place to start.

---

## What it does

Given a MySQL database and an OWL ontology, the pipeline:

1. **Inspects** every table — profiling column types, null rates, cardinality, value patterns, and data quality flags.
2. **Maps** every column to the best matching ontology property using a 3-tier confidence pipeline (corrections → local heuristics → LLM).
3. **Transforms** bronze rows into a silver layer — normalising dates, resolving value discriminators, quarantining unparseable rows.
4. **Promotes** to a gold layer — one denormalised table per ontology class, merging all silver tables that contribute rows for that class, with FK columns resolved to the correct OWL object property direction.
5. **Publishes** to Apache Jena Fuseki — converting gold rows to OWL-compliant triples with correct XSD datatypes, object property direction, and provenance annotations.
6. **Validates** the published graph against seven automated checks that reconcile entity counts, cardinality, join integrity, provenance, incompleteness, project identity, and NL query correctness.
7. **Answers** natural-language questions — translating to SPARQL via an LLM using schema context derived automatically from the ontology.

The output is a structured JSON mapping report with full evidence trails, a Fuseki-backed knowledge graph, and a validated, auditable pipeline.

---

## Validated datasets

Three datasets are currently supported and validated out of the box via `projects/*.yaml`:

| Dataset | Tables | Triples | C1 | C2 | C3 | C4 | C5 | C6 | C7 |
|---|---|---|---|---|---|---|---|---|---|
| **MIMIC-III** (clinical ICU) | 20 classes | ~11.8M | ★ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Synthea** (synthetic EHR) | 5 classes | ~3M | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **OpenFlights** (routes/airports) | 6 classes | ~900K | ★ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

★ = one accepted source-data signal (duplicate natural keys in raw data; see [Validation suite](#validation-suite)).

MIMIC-III is the flagship benchmark. It isn't bundled: download the free MIMIC-III
Clinical Database Demo from PhysioNet (a free account, no credentialing) and generate
its ontology on the first run (see `datasets/mimic/README.md`). Synthea ships as a
bundled synthetic sample and is the default project, so it runs with no downloads
(see [Quick start](#quick-start)). Set `DEFAULT_PROJECT=mimic` in `.env` once MIMIC
is loaded to make it the project the web app and MCP server start on.

---

## Pipeline phases

```
inspect → map → silver → gold → rdf → validate
```

Run all phases or any subset:

```bash
python main.py --project synthea --phases silver,gold,rdf,validate
python main.py --project mimic   --phases validate
python main.py --project mimic   --phases purge      # drop silver/gold + clear Fuseki graph
```

### Phase summary

| Phase | What it does |
|---|---|
| `inspect` | Profiles every table and column in the bronze DB |
| `map` | Runs the 3-tier mapping engine; writes `output/mapping_report_{project}.json` |
| `seed` | LLM-generates a lightweight starter ontology (one class per table) |
| `full` | LLM-generates a full ontology (two-pass or iterative RIGOR mode) |
| `silver` | Transforms bronze rows; writes `silver_{table}` per source table |
| `gold` | Promotes to `gold_{class}` per ontology class; resolves FK → object property direction |
| `rdf` | Publishes gold tables to Fuseki as OWL triples with provenance |
| `validate` | Runs the 7-check validation suite against the live graph |
| `purge` | Drops silver/gold MySQL tables and clears the Fuseki named graph |

---

## Project configuration

Each dataset is configured as a YAML file in `projects/`:

```yaml
# projects/synthea.yaml
name: synthea
description: Synthea synthetic patient records dataset

bronze_database_url: mysql+pymysql://root:password@localhost:3307/bronze_synthea
silver_database_url: mysql+pymysql://root:password@localhost:3307/silver_synthea
gold_database_url:   mysql+pymysql://root:password@localhost:3307/gold_synthea

ontology_path: owl/synthea_rigor_gpt-4o-mini_20JUN26.owl

rdf_source_name: synthea
rdf_dataset_name: synthea-mesh
named_graph_uri:  http://mesh.local/graph/synthea
ontology_named_graph_uri: http://mesh.local/graph/synthea/ontology
```

Invoke with `--project <name>`. The project YAML overrides the corresponding `.env` values for all database connections, ontology path, and graph URIs — no manual env switching needed.

---

## Validation suite

Seven checks run after each RDF publish. All checks query Fuseki live; C1 and C5 also query the gold MySQL tables.

| ID | Check | What it verifies |
|---|---|---|
| **C1** | Entity-count reconciliation | Graph subject count == gold row count per class (subject merge on duplicate natural keys is flagged, not failed) |
| **C2** | Relationship cardinality | Triples-per-subject ratio ≤ 1.5 for every object property (detects duplicate-predicate emission) |
| **C3** | Join integrity | Every FK object property with a known range class resolves to a real individual (not a dangling reference) |
| **C4** | Provenance completeness | Every class has `fm:bronzeSourceRef` and `fm:goldRunId` on all subjects |
| **C5** | Incompleteness reconciliation | Graph `fm:incompleteReference` count == gold null FK count (null DatatypeProperty literals are excluded by design — use `FILTER NOT EXISTS` to query them) |
| **C6** | Load-stamp freshness | Graph `fm:loadRunId` matches the latest RDF run in `output/pipeline_stats_{project}.json` |
| **C7** | NL query join correctness | Provided test questions return non-empty results with correct JOIN patterns |

Run standalone:

```bash
python main.py --project synthea --phases validate
```

---

## Mapping engine

### Tier 1 — cheap, local (no API calls)

| Strategy | What it does |
|---|---|
| `HistoryMatcher` | Checks human-confirmed corrections first. Any match short-circuits at score 1.0. |
| `AnnotationMatcher` | Scans `skos:altLabel`, `rdfs:label`, `rdfs:comment`. Verbatim and case-folded hits short-circuit immediately. |
| `NameMatcher` | Lexical and fuzzy matching between column name and property local name (semantic-verb prefixes `has`/`is`/`of` stripped). |
| `TypeMatcher` | Aligns the column's inferred XSD type against `rdfs:range`. FK columns score ObjectProperties 0.80 and DatatypeProperties 0.0; scalar columns invert this. |

### Tier 2 — moderate, local

| Strategy | What it does |
|---|---|
| `SemanticMatcher` | `all-MiniLM-L6-v2` cosine similarity. Only runs on columns still PENDING after Tier 1. |

### Tier 3 — remote (optional)

| Strategy | What it does |
|---|---|
| `LLMStrategy` | Single `gpt-4o-mini` batch call per table for columns still PENDING after Tier 2. Disable with `LLM_ENABLED=false`. |

### Aggregation

```
floor       = Σ(weight_s  for non-LLM strategies that ran)
denominator = max(Σ(weight_s for strategies that scored THIS uri), floor)
score(uri)  = Σ(score_s × weight_s) / denominator
```

Two structural gates apply after short-circuits:
- **Type gate** — eliminates candidates structurally incompatible with the column type.
- **Domain-preference filter** — when at least one candidate has `rdfs:domain` matching the table's class, mismatching-domain candidates are dropped.

| Strategy | Default weight |
|---|---|
| AnnotationMatcher | 0.20 |
| NameMatcher | 0.35 |
| TypeMatcher | 0.25 |
| SemanticMatcher | 0.45 |
| LLMStrategy | 2.00 |

---

## Ontology generation

### Generate Seed
Lightweight starter: one OWL class per table, datatype properties for scalars, object properties for FKs.

```bash
python main.py --project synthea --phases seed
```

Output: `{db}_gen_{yymmdd}_{seq}.owl`

### Generate Full (single-shot)
Two-pass LLM process over the entire schema:
- **Pass 1** — proposes all OWL classes and object properties.
- **Pass 2** — per-class data properties with `owl:DatatypeProperty`, `rdfs:domain`, and `xsd:range`.

```bash
python main.py --project synthea --phases full
```

Output: `{db}_full_{yymmdd}_{seq}.owl`

### Generate Iterative (RIGOR-inspired)
Table-by-table generation that eliminates cross-table property leakage and naming inconsistencies:

- Tables processed in **FK topological order** (dimension tables first, fact tables last).
- **Deterministic class URIs** derived from table names before any LLM call.
- **Gen-LLM** (gpt-4o) produces Manchester Syntax for one table at a time, with already-declared classes injected as context via an embedding index.
- **Judge-LLM** (gpt-4o-mini) validates coverage, coherence, and duplicates. Rejected fragments trigger a revision loop (max 2 retries).
- Every `DatatypeProperty` includes `skos:altLabel "<column_name>"@en` — the signal that drives `AnnotationMatcher` 1.0 scores in the mapping engine.
- **Post-accumulation split pass** — any `DatatypeProperty` that accumulated multiple `rdfs:domain` values is automatically split into class-scoped subproperties (e.g. `hasCharteventItemid`, `hasLabeventItemid`). This ensures the domain-preference filter and annotation-exact short-circuit route correctly per table.
- **`fm:sourceColumn` and `fm:sourceForeignKey` annotations** on properties provide deterministic routing in the gold builder, independently of what the mapping engine assigned.

```bash
ONTOLOGY_GENERATOR=iterative python main.py --project synthea --phases full
```

Output: `{db}_rigor_{yymmdd}_{seq}.owl`

---

## Silver layer

One `silver_{table}` per source table. One row per bronze row. Processed in parallel using `PIPELINE_WORKERS` threads.

### Transformer chain

| Priority | Transformer | Activates when |
|---|---|---|
| 1 | `DateNormaliser` | `transformation_spec` is set |
| 2 | `ValueMapper` | `value_mappings` list is non-empty |
| 3 | `NullHandler` | Always (catch-all) |

Unparseable rows go to `quarantine_{table}` — they never silently disappear.

---

## Gold layer

One `gold_{class}` per ontology class. `GoldDefinitionBuilder` groups tables by highest-confidence class URI, resolves FK columns to the correct OWL object properties (three strategies: entity-name match, `fm:sourceForeignKey` annotation, `fm:sourceColumn` annotation), and handles three edge cases:

- **Column deduplication** — same source column from merged tables: first occurrence wins.
- **C2 collision resolver** — when two CONFIRMED columns share a `mapped_property_uri`, the higher-confidence column retains the predicate; the lower-confidence column has its `property_uri` cleared to `""` and its status set to `PENDING_REVIEW` in the mapping report. Its data is still written to gold for auditability.
- **FK direction** — `rdfs:range` consultation ensures object property triples point in the semantically correct direction regardless of how the FK was defined in the relational schema.

---

## RDF layer

Reads gold tables; publishes OWL-compliant triples to Fuseki. Processed in parallel.

### Provenance triples

Every subject that carries audit columns emits two provenance triples:

| Predicate | Source column | Purpose |
|---|---|---|
| `fm:bronzeSourceRef` | `_bronze_source_ref` | Join key: triple subject → gold row → silver → bronze |
| `fm:goldRunId` | `_gold_run_id` | Staleness detection between gold tables and triplestore |

### Incompleteness quality triples

Null FK values (ObjectProperty columns whose range class has a gold table) emit `fm:incompleteReference` triples rather than being silently dropped. This enables queries to distinguish "FK missing" from "FK not applicable".

Null DatatypeProperty values produce no triple — the absence is the correct RDF representation. Use `FILTER NOT EXISTS` for "no reason code recorded" queries.

---

## Corrections store

`corrections.db` is a local SQLite learning loop. Confirmed column→property mappings are stored pattern-based and table-name-agnostic — confirmed names match across tables. On warm runs with a populated store, all columns resolve at Tier 1 with zero LLM calls.

---

## MCP server

Exposes the query engine as MCP tools for Claude Desktop or any MCP-compatible client.

Supports four query modes:

| Mode | What it queries |
|---|---|
| `sparql` (default) | NL → SPARQL → Fuseki (requires full pipeline) |
| `gold` | NL → SQL → gold MySQL tables (requires gold phase) |
| `silver` | NL → SQL → silver MySQL tables (requires silver phase) |
| `bronze` | NL → SQL → raw bronze tables (no pipeline needed) |

```bash
python mcp_server.py --project synthea --mode sparql
```

Register it with Claude Code (`claude mcp add forgemesh-synthea -- "$PWD/.venv/bin/python" "$PWD/mcp_server.py" --project synthea`)
or Claude Desktop. The JSON config is in [GETTING_STARTED.md](GETTING_STARTED.md#7-optional-query-from-claude-code-or-claude-desktop-via-mcp).

---

## Web app

```bash
python web_app.py --project synthea    # http://127.0.0.1:5173
```

| Tab | What it provides |
|---|---|
| **Query** | Chat-style SPARQL conversation, AG Grid results, token counter, ⌘↵ to submit |
| **Proof** | Verified answer badge, mapping confidence drawer, per-row provenance trace, data audit |
| **Ontology** | Multi-ontology catalog, class browser, gap suggestions, generate/upload OWL, version history |
| **Pipeline** | Run any phase from the browser with live streaming log output and a Stop button |
| **Mappings** | Browse the mapping report by table; confirm or reject individual column mappings |
| **Corrections** | Review, confirm, reject, or delete stored corrections |

---

## Architecture

```
semantic_mapper/
├── config/
│   ├── settings.py              — all thresholds and knobs (Pydantic, .env-backed)
│   └── logging_config.py        — structured logging setup
├── core/
│   ├── exceptions.py            — domain exception hierarchy
│   └── constants.py             — shared enums and string constants
├── db/
│   ├── inspector.py             — read-only SQLAlchemy introspection of MySQL tables
│   ├── models.py                — TableProfile, ColumnProfile (Pydantic)
│   ├── corrections_store.py     — SQLite learning loop (PatternLookup, CorrectionsStore)
│   └── run_store.py             — pipeline run history
├── ontology/
│   ├── loader.py                — rdflib OWL/RDF parser
│   ├── analyzer.py              — extracts classes, properties, labels, annotations
│   ├── seed_generator.py        — LLM-driven seed ontology
│   ├── full_generator.py        — two-pass LLM full ontology generation
│   └── iterative/               — RIGOR-inspired table-by-table generator
│       ├── topology.py          — FK topological sort (Kahn's algorithm + cycle detection)
│       ├── embedding_index.py   — incremental class retrieval (all-MiniLM-L6-v2)
│       ├── context_builder.py   — deterministic class URI derivation + TableContext
│       ├── gen_prompt.py        — Gen-LLM prompt
│       ├── judge_prompt.py      — Judge-LLM coverage + coherence evaluation prompt
│       ├── manchester_parser.py — owlready2 primary + regex fallback; never raises
│       ├── graph_merger.py      — duplicate-safe merge into accumulating rdflib.Graph
│       ├── property_namer.py    — canonical URI normalisation for DatatypeProperties
│       └── generator.py         — IterativeOntologyGenerator
├── engine/
│   ├── mapper.py                — MapperOrchestrator: Tier 1 → Tier 2 → Tier 3 → aggregation
│   ├── aggregator.py            — weighted-sum evidence combiner with floor denominator
│   ├── reasoner.py              — ontology constraint validation
│   └── strategies/
│       ├── history_matcher.py   — Tier 1: human-confirmed corrections (always wins)
│       ├── annotation_matcher.py — Tier 1: OWL skos:altLabel / rdfs:label scanning
│       ├── name_matcher.py      — Tier 1: lexical / fuzzy name matching
│       ├── type_matcher.py      — Tier 1: XSD datatype alignment + FK structural typing
│       ├── semantic_matcher.py  — Tier 2: sentence-transformer cosine similarity
│       └── llm_strategy.py      — Tier 3: GPT-4o-mini batch inference
├── silver/
│   ├── models.py                — frozen Pydantic contracts (versioned public API)
│   ├── pipeline.py              — SilverPipeline (parallel via ThreadPoolExecutor)
│   ├── transform/               — DateNormaliser, ValueMapper, NullHandler
│   └── writer/                  — SilverWriter ABC + MySQLSilverWriter
├── gold/
│   ├── models.py                — GoldTableDefinition, GoldColumnSpec (frozen Pydantic)
│   ├── builder.py               — GoldDefinitionBuilder: report → List[GoldTableDefinition]
│   │                              (FK resolution, C2 collision resolver, id_column assignment)
│   ├── pipeline.py              — GoldPipeline (parallel via ThreadPoolExecutor)
│   └── writer/                  — GoldWriter ABC + MySQLGoldWriter
├── rdf/
│   ├── models.py                — RdfPipelineStats (frozen Pydantic)
│   ├── pipeline.py              — RdfPipeline: gold → triples → Fuseki (parallel)
│   ├── triple_builder.py        — TripleBuilder: gold row → RDF triples
│   │                              (XSD types, FK direction, provenance, incompleteReference)
│   ├── uri_factory.py           — UriFactory: entity + id → percent-encoded URIRef
│   ├── writer/                  — RdfWriter ABC + FusekiWriter + RdflibWriter (tests)
│   └── query/
│       ├── safety.py            — SPARQL SELECT-only guard
│       ├── context_builder.py   — build_sparql_schema_context() with PREFIX declarations
│       └── engine.py            — SparqlQueryEngine (NL → SPARQL → rows via Fuseki)
├── query/                       — NL → SQL interface (silver/gold)
│   ├── safety.py                — validate_select_only() guard
│   ├── context_builder.py       — build_schema_context() for silver tables
│   ├── gold_context_builder.py  — build_gold_schema_context() for gold tables
│   └── engine.py                — SemanticQueryEngine (NL → SQL → rows)
└── validation/
    ├── runner.py                — ValidationRunner: orchestrates all checks
    ├── context.py               — ValidationContext: sparql, gold_engine, ontology, …
    ├── models.py                — CheckResult (frozen Pydantic)
    └── checks/
        ├── entity_count.py      — C1: graph subject count == gold row count
        ├── cardinality.py       — C2: triples/subjects ratio per object property
        ├── join_integrity.py    — C3: FK object properties resolve to real individuals
        ├── provenance.py        — C4: every subject has bronzeSourceRef + goldRunId
        ├── incompleteness.py    — C5: graph incompleteReference count == gold null FK count
        ├── load_stamp.py        — C6: graph loadRunId matches pipeline_stats file
        └── nl_query.py          — C7: NL test questions return non-empty results

main.py              — CLI pipeline runner (all phases, --project flag)
query.py             — CLI REPL (--sparql for SPARQL mode)
mcp_server.py        — MCP server (Claude Desktop / MCP clients)
web_app.py           — Local web app (FastAPI + AG Grid; no build step)
projects/            — Per-dataset YAML project files
owl/                 — OWL ontology files (seed, full, rigor iterations)
output/              — Mapping reports, pipeline stats (gitignored)
```

---

## Quick start

Needs Docker, Python ≥ 3.11 and an OpenAI API key (or a local model via Ollama).
Runs on the bundled synthetic Synthea sample, so no data download is needed.
**[GETTING_STARTED.md](GETTING_STARTED.md)** walks through each step with checks
and troubleshooting.

```bash
git clone https://github.com/tagben/forgemesh.git && cd forgemesh
python3 -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
cp .env.example .env                        # then set LLM_API_KEY

docker compose up -d db fuseki              # MySQL :3307 + Fuseki :3030
python scripts/load_synthea.py              # bronze: bundled Synthea sample
python main.py --project synthea            # inspect → map → silver → gold → rdf
python web_app.py --project synthea         # web app → http://127.0.0.1:5173
```

---

## Key environment variables

All variables can be set in a `.env` file or overridden by a project YAML. See `.env.example` for a complete list.

| Variable | Default | Description |
|---|---|---|
| `DATABASE_URL` | — | Bronze MySQL connection string |
| `ONTOLOGY_PATH` | — | Path to OWL file |
| `LLM_API_KEY` | — | API key (required for Tier 3 and NL query; any non-empty string for a local backend). Legacy alias: `OPENAI_API_KEY`. |
| `LLM_BASE_URL` | — | OpenAI-compatible endpoint override. Unset = cloud OpenAI; set to `http://localhost:11434/v1` to run fully local via Ollama. See [Running fully local](#running-fully-local-air-gapped). Legacy alias: `OPENAI_BASE_URL`. |
| `LLM_ENABLED` | `true` | `false` = Tier 1+2 only, no API calls |
| `LLM_MODEL` | `gpt-4o-mini` | Default chat model — ontology generation, Tier 3 mapping, narrator. Legacy alias: `OPENAI_MODEL`. |
| `AUTO_CONFIRM_THRESHOLD` | `0.70` | Min score to auto-confirm a mapping |
| `SAMPLE_SIZE` | `1000` | Rows sampled per column during profiling |
| `PIPELINE_WORKERS` | CPU count (max 12) | Parallel threads for silver/gold/RDF phases |
| `SILVER_DATABASE_URL` | — | Silver target DB |
| `GOLD_DATABASE_URL` | — | Gold target DB |
| `QUERY_MODEL` | `gpt-4o` | NL → SPARQL model |
| `QUERY_MAX_ROWS` | `100` | Max rows returned per query |
| `FUSEKI_URL` | `http://localhost:3030` | Fuseki base URL |
| `RDF_DATASET_NAME` | `forgemesh` | Fuseki dataset name |
| `RDF_NAMED_GRAPH` | `http://mesh.local/graph/current` | Named graph URI |
| `RDF_BASE_URI` | `http://mesh.local/data/` | Base URI for individual subjects |
| `ONTOLOGY_GENERATOR` | `full` | `iterative` to use the RIGOR-inspired generator |
| `ONTOLOGY_GEN_MODEL` | (= `LLM_MODEL`) | Override gen-LLM model for iterative generator; unset uses `LLM_MODEL` |
| `ONTOLOGY_JUDGE_MODEL` | (= `LLM_MODEL`) | Override judge-LLM model for iterative generator; unset uses `LLM_MODEL` |
| `CORRECTIONS_DB_PATH` | `corrections.db` | SQLite corrections store path |

---

## Running fully local (air-gapped)

ForgeMesh can run with **no external API calls** by pointing its OpenAI-compatible
client at a local model server. Embeddings (`all-MiniLM-L6-v2`) already run locally,
so the only thing to redirect is the chat/LLM backend.

**1. Install a local model server.** [Ollama](https://ollama.com) is the simplest:

**Windows (PowerShell):**

```powershell
winget install Ollama.Ollama
ollama pull qwen2.5:32b-instruct      # ~19 GB — 4o-class reasoning for ontology gen
ollama pull qwen2.5:14b-instruct      # ~9 GB — fast, for interactive queries
```

**macOS (Homebrew):**

```bash
brew install ollama
ollama pull qwen2.5:32b-instruct
ollama pull qwen2.5:14b-instruct
```

**Linux:**

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull qwen2.5:32b-instruct
ollama pull qwen2.5:14b-instruct
```

Ollama serves an OpenAI-compatible API at `http://localhost:11434/v1`. Sizing
guide for a single 16 GB GPU: a 14B (q4) fits entirely in VRAM; a 32B (q4) runs
by offloading a few layers to system RAM (slower but viable with ≥32 GB RAM); a
70B+ won't fit.

**2. Point ForgeMesh at it.** Settings are vendor-agnostic (`LLM_*`); selecting
the backend is config-only — set `LLM_BASE_URL` and name the local models, in
`.env` or in a separate profile such as `.env.local` (the commented Ollama block in
`.env.example` is a starting point):

```ini
LLM_BASE_URL=http://localhost:11434/v1
LLM_API_KEY=ollama                    # any non-empty string; local servers ignore it
LLM_MODEL=qwen2.5:32b-instruct        # ontology gen / mapping / narrator (heavy)
QUERY_MODEL=qwen2.5:14b-instruct      # NL→SQL / NL→SPARQL / MCP (interactive)
QUERY_PROVIDER=openai                 # NOT claude_code (that uses the cloud CLI)
ANTHROPIC_API_KEY=
```

The legacy `OPENAI_MODEL` / `OPENAI_API_KEY` / `OPENAI_BASE_URL` names still work
as aliases. If you used a separate profile, load it with `--env`:

```bash
python main.py --env .env.local --project synthea
./start-forgemesh.sh --env .env.local --project synthea   # also starts Ollama
```

Leaving `LLM_BASE_URL` unset restores cloud OpenAI — the same switch as choosing
`gpt-4o` vs `gpt-4o-mini`.

**Optional — split the ontology generator's two passes** to keep the slow big
model only where it counts. The generation pass benefits from the 32B; the more
frequent judge pass can use the faster 14B:

```ini
ONTOLOGY_GEN_MODEL=qwen2.5:32b-instruct
ONTOLOGY_JUDGE_MODEL=qwen2.5:14b-instruct
```

If unset, both passes use `LLM_MODEL`.

**3. Pre-cache embeddings before going offline.** The first load of
`all-MiniLM-L6-v2` downloads from HuggingFace. Cache it once while online, then
forbid future fetches:

**Windows (PowerShell):**

```powershell
python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('all-MiniLM-L6-v2')"
$env:HF_HUB_OFFLINE = "1"
```

**macOS and Linux:**

```bash
python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('all-MiniLM-L6-v2')"
export HF_HUB_OFFLINE=1
```

**4. Verify the air gap.** A preflight script checks config, the local chat
endpoint, and the embedding cache without touching the cloud:

**Windows (PowerShell):**

```powershell
python scripts/check_local_llm.py .env.local
```

**macOS and Linux:**

```bash
python scripts/check_local_llm.py .env.local
```

Run it once online, then again with your network adapter disabled — if it still
passes, nothing left the machine.

---

## Test suite

```bash
pytest tests/ -v
```

2,005 passing, 5 skipped, 2 known failures (one live-DB integration test, one telemetry count test).

| Module | Tests | What it covers |
|---|---|---|
| `tests/ontology/` | 496 | Full generator, iterative generator, coherence checks, property normalisation |
| `tests/web_app/` | 209 | Proof endpoint, row-trace endpoint, queried_uris extraction, caveat serialisation |
| `tests/prove/` | 152 | Proof composer, narrator (LLM mocked), grounding guard, subgraph, query rewriter |
| `tests/engine/` | 124 | All mapping strategies, aggregator, FK typing, domain-mismatch resolution |
| `tests/rdf/` | 137 | TripleBuilder (XSD types, FK direction, provenance, incompleteReference), RdfPipeline, SparqlQueryEngine, C2 collision resolver |
| `tests/harness/` | 33 | SQL and SPARQL queries against known test data |
| `tests/gold/` | 69 | GoldDefinitionBuilder (FK resolution, collision resolver, id_column), GoldPipeline, lineage columns |
| `tests/query/` | 64 | SemanticQueryEngine, safety, context builder |
| `tests/silver/` | 95 | Transformer chain, writer, pipeline, quarantine |
| `tests/eval/` | 17 | OMOP ground-truth precision/recall evaluation |
| `tests/db/` | 11 | Run store round-trip, corrections store |

All unit tests use SQLite in-memory — no Docker required for the test suite.

---

## Dependencies

| Package | Purpose |
|---|---|
| `rdflib` | OWL/RDF parsing and in-memory triplestore |
| `owlready2` | Manchester Syntax parsing in iterative generator |
| `requests` | HTTP POST to Fuseki |
| `sqlalchemy` + `pymysql` | MySQL introspection and writes |
| `pydantic` + `pydantic-settings` | Data models and settings |
| `sentence-transformers` | Semantic similarity (MiniLM) — Tier 2 + iterative embedding index |
| `rapidfuzz` | Fuzzy string matching |
| `openai` | Tier 3 inference, ontology generation, query interface |
| `python-dotenv` | `.env` loading |
| `fastapi` + `uvicorn` | Web app server |
| `mcp` | MCP server SDK (Claude Desktop integration) |
| AG Grid Community (CDN) | Result tables in web app (no npm required) |

---

## Project conventions

- **No dataset-specific knowledge in code** — every inference mechanism generalises across arbitrary schemas. Dataset-specific workarounds in configuration are bugs, not features.
- **No credentials in code** — everything via `.env` or project YAML.
- **Read-only DB access** — the inspector never requests write permissions on the bronze DB.
- **Full audit trail** — every mapping records all strategy evidence, individual scores, and composite rationale.
- **Corrections store is a runtime artefact** — excluded from version control.
- **All unit tests use SQLite in-memory** — no live DB connections required.

---

## Contributing and license

Contributions are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for the development
rules, and [SECURITY.md](SECURITY.md) for reporting vulnerabilities.

ForgeMesh is licensed under the [Apache License 2.0](LICENSE). Bundled third-party
material (fonts, sample data, Wikipedia snapshots, code lists) keeps its own license;
see [NOTICE](NOTICE).
