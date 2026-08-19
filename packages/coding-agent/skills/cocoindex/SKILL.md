---
name: cocoindex
description: Build and maintain incremental data pipelines for ALPHA with CocoIndex v1. Use for live agent context, RAG/vector indexes, document and code ingestion, ETL, embeddings, knowledge graphs, file/database synchronization, and pipelines that should reprocess only changed data. Prefer the user's darnleyweekes-spec/cocoindex fork for examples and current v1 guidance.
compatibility: Requires Python 3.10-3.13. Install cocoindex>=1.0.0 in the target project's environment when execution is needed.
---

# CocoIndex for ALPHA

CocoIndex is ALPHA's preferred incremental indexing and data-pipeline capability when an agent needs continuously fresh context rather than a one-time batch. It lets a project declare the desired target state from source data and recomputes only the changed delta.

Core model:

```text
TargetState = Transform(SourceState)
```

Use the user's fork as the preferred implementation/reference source:

```text
https://github.com/darnleyweekes-spec/cocoindex
```

CocoIndex is a capability inside ALPHA. It does not replace ALPHA's controller, specialist routing, safety/reliability layer, approval gates, or memory policy.

## ALPHA routing

Load this skill for any ALPHA specialist working on data freshness, retrieval, ingestion, or indexing. Typical ownership:

- **HAL** — data/ML pipelines, embeddings, evaluation datasets, RAG infrastructure.
- **JAN** — text processing, chunking, retrieval, language-data workflows.
- **BIANCA** — connector architecture, databases, external-system integration.
- **FRED** — forward-deployed customer implementations and production integration.
- **TED** — operational reliability, update cadence, monitoring, recovery, and pipeline health.
- **ROSALIND** — source-backed research corpora that must remain fresh as inputs change.
- **VISION / ALPHA** — decide whether incremental indexing is justified and keep the implementation scoped to the mission.

Do not create a new CocoIndex controller agent. Route CocoIndex through the specialist already responsible for the workflow.

## When to use CocoIndex

Use CocoIndex when the task involves one or more of:

- continuously fresh context for AI agents;
- indexing codebases, docs, meeting notes, inbox-style data, Slack-like corpora, PDFs, or videos;
- vector embeddings or semantic retrieval;
- RAG pipelines where only changed inputs should re-embed;
- knowledge-graph extraction and synchronization;
- ETL or database transformations with automatic change detection;
- file-to-file or file-to-database transformations;
- streaming/Kafka-style ingestion;
- local or production indexes that should stay synchronized with source state.

Do not use CocoIndex merely because data exists. For a small one-off transform, direct Python/Pandas may be simpler. Prefer CocoIndex when incremental recomputation, lineage, synchronization, freshness, or persistent target state provides real value.

## Version rule

Use **CocoIndex v1 (`>=1.0.0`) only**. Do not emit removed v0 APIs from model memory or old tutorials.

Removed v0 patterns include:

- `@cocoindex.flow_def`, `FlowBuilder`, `Flow`, `open_flow`
- `DataScope`, `DataSlice`, `add_collector()`, `collect()`, `export()`
- `cocoindex.sources.*`
- `cocoindex.functions.*`
- `cocoindex.targets.*` / `storages.*`
- `transform_flow`, `cocoindex.op.function()`
- `cocoindex setup`

For v1, use `coco.App`, `@coco.fn`, connector APIs, target-state declaration, and `cocoindex update`.

## Project installation

Do not add CocoIndex to Prime Agent's core runtime just to make the skill visible. Install it in the project that needs it.

Preferred setup:

```bash
python3 --version
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -U "cocoindex>=1.0.0"
```

Or in an existing `uv` project:

```bash
uv add "cocoindex>=1.0.0"
```

For a new CocoIndex project:

```bash
cocoindex init my-project
cd my-project
```

Before adding optional connectors or model packages, inspect the target project's existing dependency manager and environment. Do not replace an existing environment or dependency strategy without need.

## Core v1 concepts

### Apps

An app binds a `@coco.fn` main function to parameters:

```python
import pathlib
import cocoindex as coco

@coco.fn
async def app_main(sourcedir: pathlib.Path) -> None:
    ...

app = coco.App(
    coco.AppConfig(name="MyApp"),
    app_main,
    sourcedir=pathlib.Path("./data"),
)
```

### Processing functions

Use `@coco.fn` for functions that participate in the CocoIndex processing graph. Add `memo=True` to expensive deterministic work so unchanged inputs/code are skipped:

```python
@coco.fn(memo=True)
async def expensive_operation(data: str) -> Result:
    return await expensive_transform(data)
```

Useful controls include:

- `memo=True` — cache by dependency/code state;
- `version=N` — explicit invalidation when behavior changes;
- `batching=True` — batch concurrent async calls where supported;
- `runner=coco.GPU` — serialize GPU-bound work where appropriate.

### Components

Prefer stable component identities:

```python
await coco.mount_each(process_file, files.items(), target_table)
await coco.mount(setup_fn, arg1)
result = await coco.use_mount(init_fn)
```

Use explicit `component_subpath(...)` only when a stable explicit path is needed. Never key persistent components from transient object identity or unstable list positions.

### Target state

Declare what should exist. Let CocoIndex handle create/update/delete synchronization:

```python
table.declare_row(row=record)
localfs.declare_file(out_path, content, create_parent_dirs=True)
```

### Shared resources

Use `ContextKey` plus `@coco.lifespan` for reusable resources such as DB pools and embedding models:

```python
EMBEDDER = coco.ContextKey[SentenceTransformerEmbedder]("embedder")

@coco.lifespan
async def coco_lifespan(builder: coco.EnvironmentBuilder):
    builder.provide(EMBEDDER, SentenceTransformerEmbedder("all-MiniLM-L6-v2"))
    yield
```

Keep `ContextKey` names stable across runs because identity affects incremental state.

## Incremental agent-context pattern

For fresh ALPHA context, structure the pipeline as:

```text
source corpus
  -> parse / normalize
  -> chunk with stable identity
  -> expensive embedding or extraction under memoization
  -> persistent target index
  -> ALPHA retrieval layer
```

The critical behavior is delta-only recomputation. If one source file changes, do not rebuild or re-embed the entire corpus unless the transformation itself changed in a way that invalidates all outputs.

## Example: local documents to embeddings

Use the official v1 pattern and adapt it to the project's storage choice:

```python
import pathlib
import cocoindex as coco
from cocoindex.connectors import localfs
from cocoindex.ops.text import RecursiveSplitter
from cocoindex.resources.file import PatternFilePathMatcher

splitter = RecursiveSplitter()

@coco.fn(memo=True)
async def process_file(file, target) -> None:
    text = await file.read_text()
    chunks = splitter.split(text, chunk_size=1200, chunk_overlap=200)
    # Embed/store chunks using the project's target connector.
    ...

@coco.fn
async def app_main(sourcedir: pathlib.Path) -> None:
    target = ...
    files = localfs.walk_dir(
        sourcedir,
        recursive=True,
        path_matcher=PatternFilePathMatcher(
            included_patterns=["**/*.md", "**/*.txt"]
        ),
    )
    await coco.mount_each(process_file, files.items(), target)

app = coco.App(
    coco.AppConfig(name="AlphaContext"),
    app_main,
    sourcedir=pathlib.Path("./knowledge"),
)
```

Use the project's approved embedding model and storage backend; do not hardcode credentials in source.

## Connectors

CocoIndex supports multiple source/target styles. Choose based on the existing system rather than introducing a new database without need. Common options include:

- PostgreSQL / pgvector
- SQLite / sqlite-vec
- LanceDB
- Qdrant
- SurrealDB
- Apache Doris
- Local filesystem
- Amazon S3
- Kafka
- Google Drive sources

When a connector requires credentials, keep them in the project's existing secret-management path. Never commit credentials or connection strings containing secrets.

## Live mode

Catch-up mode scans sources, processes changed state, synchronizes targets, and exits. Live mode stays running and applies supported source changes continuously.

```bash
cocoindex update main.py
cocoindex update main.py -L
```

Use live mode only where a long-running process is operationally supported. TED or the owning operations agent should define restart behavior, health checks, and deployment ownership before production use.

## CLI

Common v1 commands:

```bash
cocoindex init my-project
cocoindex update main.py
cocoindex update main.py:my_app
cocoindex update main.py -L
cocoindex update main.py --full-reprocess
cocoindex ls main.py
cocoindex show main.py --tree
```

Treat `drop` or full reprocessing as consequential operations when they can destroy state, create substantial cost, or affect production. Require the normal ALPHA approval boundary before running them against important environments.

## Reliability rules

1. Use stable source and component identities.
2. Put expensive LLM, parsing, and embedding work behind `memo=True` when semantically safe.
3. Track source-to-target lineage rather than writing ad hoc update scripts around CocoIndex.
4. Verify incremental behavior by changing one controlled input and confirming only its dependent output recomputes.
5. Separate local development state from production state.
6. Add health checks for live pipelines and observable failure reporting.
7. Do not silently fall back from an incremental design to full reprocessing.
8. Do not let retrieval/indexing capability broaden an agent's permissions to data it was not already authorized to access.

## ALPHA safety boundary

CocoIndex changes how approved data is processed; it does not create new authorization.

- Respect each agent's existing data-access scope.
- Do not ingest private customer, employee, health, credential, or regulated data into new stores without an approved data boundary.
- Do not index secrets, `.env` files, auth stores, private keys, raw tokens, or credential payloads.
- Do not connect to external databases, cloud stores, or messaging systems without the needed credentials and approval.
- Do not make production `drop`, destructive schema, or full-reprocess decisions automatically when cost or availability could be material.
- Preserve source citations/provenance where the downstream ALPHA workflow relies on factual grounding.

## Verification checklist

Before calling a CocoIndex integration complete:

1. `python -c "import cocoindex; print(cocoindex.__version__ if hasattr(cocoindex, '__version__') else 'cocoindex import ok')"` succeeds in the project environment.
2. `cocoindex show <app.py> --tree` resolves the expected app/components where supported.
3. Initial catch-up completes without unhandled errors.
4. Modify one controlled source item and rerun.
5. Confirm unrelated expensive components are not recomputed.
6. Confirm deleted source state is reflected correctly in targets when the workflow requires deletion propagation.
7. Verify retrieval returns current content after the delta update.
8. For live mode, verify restart/recovery behavior before production deployment.

## Preferred references

Use the user's fork first:

- Repository: `https://github.com/darnleyweekes-spec/cocoindex`
- Official agent skill in the fork: `skills/cocoindex/SKILL.md`
- API reference: `skills/cocoindex/references/api_reference.md`
- Connectors: `skills/cocoindex/references/connectors.md`
- Patterns: `skills/cocoindex/references/patterns.md`
- Project setup: `skills/cocoindex/references/setup_project.md`
- Database setup: `skills/cocoindex/references/setup_database.md`

The fork's bundled skill is the source of truth for CocoIndex v1 syntax. When model memory, old tutorials, or third-party examples conflict with that skill, follow the fork's v1 guidance.
