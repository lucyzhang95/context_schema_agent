# Knowledge Graph Schema Discovery Agent

## Project goal

Build an agentic pipeline that takes a set of biological and chemical entities and produces a rich, structured context
object for every node.

---

## Background and design decisions

### Unit of work

The unit of work is a **node**. Nodes represent biological or chemical entities including but not limited to diseases,
phenotypes, proteins, small molecules, pathways, biological processes, and anatomical features. Each node gets a context
object.

Relationships between nodes are captured as part of that context (ranked by
importance). Consider things such as entity location, entity prevalence, and entity interactors, at different biological
scale for context.

### Why agentic

The starting schema is loaded from the archive (`output/archive/schema_final_N.json`,
highest N). The agent's job is to **validate and refine the controlled vocabularies**
by attempting to populate the schema for real nodes, discovering gaps, and
adjusting vocabulary terms. The schema fields themselves (names, descriptions,
types, applies_to_types) are **fixed** and must not be added, removed, or
renamed. Only the controlled vocabulary term lists may be modified.

### Entity summarization (LLM-based)

Prior to extracting data to the schema, each entity is first queried via an LLM
(OpenAI) to produce a summary of important biological/chemical information about
that entity. The LLM draws on all of its training knowledge — not a single
external source. The summary response is capped at **1,000 tokens** (`max_tokens=1000`).
This summary then serves as the source material for populating schema fields.

### External data sources

- **LLM knowledge** (via OpenAI API) is the primary source for entity context.
  No external search APIs are used.

### OpenAI Batch API

All LLM calls in Phase 1 (summarization) and Phase 2 (schema-field mapping)
use the **OpenAI Batch API**. This applies to every run — both the 500-node
iteration set and production runs on the full 250k node set.

Batch API mechanics:

1. Build a JSONL file where each line is a chat completion request:
   ```json
   {"custom_id": "node-DOID:123", "method": "POST", "url": "/v1/chat/completions", "body": {"model": "gpt-4o-mini", "messages": [...], "max_tokens": 1000}}
   ```
2. Upload the JSONL file via `client.files.create()`
3. Create a batch via
   `client.batches.create(input_file_id=..., endpoint="/v1/chat/completions", completion_window="24h")`
4. Poll `client.batches.retrieve(batch_id)` until status is `completed`
5. Download results via `client.files.content(output_file_id)`

Batch size: **5,000 nodes per batch**. For 500 nodes this means 1 batch per
phase; for 250,000 nodes this means 50 batches per phase.

Batch API pricing is **50% off** standard token pricing:

- gpt-4o-mini batch: $0.075/M input, $0.30/M output

Each batch submission produces three artifacts that must be persisted:

| Artifact          | Location                    | Format                                                                                            |
|-------------------|-----------------------------|---------------------------------------------------------------------------------------------------|
| Batch identifiers | `output/batches/batch_ids/` | One JSON file per submission: `{batch_id, phase, batch_number, node_count, submitted_at, status}` |
| Batch inputs      | `output/batches/inputs/`    | JSONL files as submitted: `phase1_batch_001.jsonl`, `phase2_batch_001.jsonl`                      |
| Batch outputs     | `output/batches/outputs/`   | JSONL files returned: `phase1_batch_001_output.jsonl`, `phase2_batch_001_output.jsonl`            |

Naming convention: `{phase}_{batch_number:03d}` (e.g., `phase1_batch_001`,
`phase2_batch_042`).

### Nodes

The nodes are found in `./db/nodes.csv`. The starting schema is loaded from
`./output/archive/schema_final_N.json` (the file with the highest N).

### Cost

Calculate the cost of the experiment up front and ask the user to approve a
certain dollar value to spend. Do not exceed this amount in API queries. Use
Batch API pricing (50% off) for the estimate.

External benchmarking has a **separate, smaller budget cap**. The user is
prompted once for a benchmark budget (default 2 USD) when `--benchmark` is set.
This budget is not drawn from the per-iteration cap. Token consumption: one
Phase-1 summarization, one Phase-2 population, plus one categorization pass and
one re-categorization pass per row, plus a single column-mapping call. For a
10k-node external file this is well under 2 USD on Batch API pricing.

---

## Starting schema

The starting schema (`output/archive/schema_final_N.json`, highest N) contains:

- **2 identity fields**: `id`, `name` (carried from CSV)
- **21 novel biological context fields**, each with a controlled vocabulary:
  organism, tissue_location, cell_type, cellular_compartment, biological_system,
  biological_scale, biological_process, molecular_function, pathway_category,
  mechanism_of_action, disease_association, clinical_relevance,
  phenotype_category, chemical_classification, drug_class, regulatory_role,
  interaction_type, inheritance_pattern, developmental_stage, taxonomic_domain,
  expression_context
- **No CSV columns are repeated** except id and name
- All context fields use `field_type: "controlled"` with vocabularies of 8–30 terms (max 30)

---

## Schema versioning

All schema and summary artifacts live in `output/archive/` with run-number
suffixes. There is no separate `output/schema/` folder.

At the **start of each run**:

1. Scan `output/archive/` for `schema_final_N.json` files.
2. Load the file with the **highest N** as the starting schema.
3. The new run's outputs use run number **N+1**:
    - `output/archive/schema_final_(N+1).json`
    - `output/archive/refinement_summary_(N+1).md`

No files are moved or deleted — each run appends new numbered files.

---

## Current task

Run **multiple iterations** of schema refinement in a single invocation.
Each iteration processes **500 diverse nodes** (different random sample each
time) and refines the controlled vocabularies. The schema output of iteration
N feeds as input to iteration N+1, creating a chained refinement process.

### CLI interface

```bash
python schema_agent.py --mode async --iterations 10
python schema_agent.py --mode async --iterations 5 --resume
python schema_agent.py --mode async --iterations 5 --benchmark db/external_protein_nodes.csv
```

- `--iterations N` (default 10): number of refinement iterations to run
- `--mode batch|async` (default batch): API mode for Phase 1 & 2
- `--resume`: resume from the latest `schema_final_N.json` in `output/archive/`
- `--benchmark <path>`: after the final iteration, run external node
  benchmarking against the file at `<path>` (e.g.
  `db/external_protein_nodes.csv`). Uses the latest `schema_final_K.json`. Does
  not modify the schema.

### Per-iteration workflow

For each iteration (i = 1 … N):

1. **Load the latest schema** from `output/archive/schema_final_K.json`
   (highest K). On iteration 1 this is the pre-existing schema; on subsequent
   iterations it is the schema finalized by the previous iteration.
2. **Select 500 new diverse nodes** from `nodes.csv` (covering all 9 entity
   types, mixing high-degree and low-degree nodes). Each iteration draws a
   fresh random sample — nodes may repeat across iterations but each sample
   is independently randomized.
3. **Phase 1 — Summarize**: query the LLM to summarize each entity (capped at
   1,000 tokens).
4. **Phase 2 — Populate**: map each summary to schema fields. Each
   controlled-vocabulary field returns a **list of labels**.
5. **Phase 3 — Refine**: synchronous agent loop reviews population results and
   modifies controlled vocabularies only.
6. **Output three files** for this iteration:
    - `output/archive/schema_final_(K+1).json` — the refined schema
    - `output/archive/refinement_summary_(K+1).md` — structured per-field summary
    - `output/archive/nodes_(K+1).json` — the 500 populated nodes

### After the final iteration

All plots are generated automatically:

- `images/pca_context.png` — PCA of node context vectors (from latest nodes file)
- `images/node_types_by_iteration.png` — stacked barplot of entity types per iteration
- `images/term_changes_by_iteration.png` — grouped barplot of terms added/removed per iteration

### Cross-iteration state

The outer loop in `main()` tracks three pieces of state across iterations:

1. **Cumulative term frequencies** (`cumulative_freq`): a dict mapping each
   controlled-vocabulary field name to a dict of term → count, accumulated
   across all iterations. Updated in-place by `update_cumulative_freq()` inside
   `run_pipeline`. Shown in the Phase 2 analysis so the Phase 3 agent can see
   which terms are consistently used vs. rarely seen.

2. **Stability counts** (`stability_counts`): per-field count of consecutive
   iterations with no vocabulary changes. Reset to 0 whenever a field's
   vocabulary is modified.

3. **Locked fields** (`locked_fields`): fields whose stability count reaches
   `STABLE_LOCK_THRESHOLD` (default 3). Once locked, a field's vocabulary is
   frozen: Phase 2 analysis marks it `[LOCKED]`, the Phase 3 prompt instructs
   the agent to skip it, and suggestions for locked fields are suppressed.

### Term normalization

All vocabulary terms and populated node values are normalized on save/clean:
lowercase, strip whitespace, replace spaces with underscores. This is handled
by `normalize_term()` in `schema_tools.py`. Deduplication is applied after
normalization.

### External node benchmarking

A validation feature that runs an **external node file from a different
database** through the same outer loop and benchmarks the agent-generated
context against the database's own annotations. The goal is to measure how
accurately the pipeline (a) categorizes nodes into entity types, (b) populates
the 21 novel biological context fields, and (c) agrees with an independent
source on overlapping fields.

This is a **side-car validation feature**. It does not modify the schema, does
not feed back into vocabulary refinement, and does not consume the
per-iteration budget unless explicitly enabled.

#### Inputs

External node files live in `db/` and must contain at minimum an identifier
column and a name column. They may carry any number of additional pre-annotated
context columns — these are the "gold" values used for comparison.

The first benchmark file is `db/external_protein_nodes.csv`:

| Pipeline field | Source column  | Notes                                         |
|----------------|----------------|-----------------------------------------------|
| `id`           | `protein`      | UniProtKB accession; not yet normalized       |
| `name`         | `protein_name` | Primary signal for entity-type categorization |

> **Important:** During entity-type categorization, rely **primarily on
> `name`**. The `id` column may be inconsistent across external sources at this
> stage. The `id` is used only as a *reassurance / cross-check* signal in step
> 2 below. **Never** infer the entity type from the file name itself.

#### Workflow

Let `K+1` denote the run number of the latest finalized schema (same convention
as the rest of the pipeline). Let `{node_entity}` be the dominant entity type
detected for the file (e.g. `protein`).

1. **Entity-type categorization (name only).**
   For every row, classify the node into one of the 9 supported entity types
   using **only the `name` column** plus the LLM's own knowledge. Do not use
   the file name, the `id`, or any other column. This is a Phase-1-style Batch
   API call (`gpt-4o-mini`).

2. **Cross-check with `id`.**
   Re-run categorization using **both `id` and `name`**. For UniProtKB inputs
   this means the LLM may resolve the accession to a known protein. Compare
   the two passes:
    - Build a reclassification table:
      `{id, name, type_from_name, type_from_id_and_name, changed: bool, reason}`.
    - Write `output/archive/external_reclassification_report_(K+1).md`
      summarizing how many nodes changed type, which directions are most
      common, and a few representative examples.
    - The categorization used **downstream** is `type_from_id_and_name`. Report
      any rows where the two passes disagree as a known caveat in the final
      benchmark report.

3. **Run the existing outer-loop Phases 1–2 on the external nodes.**
   Using the latest `schema_final_(K+1).json` from `output/archive/`:
    - Phase 1 — Summarize each external node (Batch API, `gpt-4o-mini`, ≤1000
      tokens).
    - Phase 2 — Populate the 21 novel biological context fields against the
      loaded schema's controlled vocabularies.
    - **Skip Phase 3.** Benchmarking does not refine vocabularies.
    - Persist batch artifacts under `output/batches/` using the existing layout,
      but with `external_` prefixed to the batch number key (e.g.
      `phase1_external_batch_001.jsonl`).

4. **Write per-entity output files** to `output/archive/`:
    - `output/archive/external_nodes_summary_(K+1).md` — coverage stats per
      field, plus the schema file name used (mirrors `refinement_summary_N.md`
      format but read-only).
    - `output/archive/external_{node_entity}_nodes_(K+1).json` — populated nodes
      (same shape as `nodes_(K+1).json`).
    - `output/archive/external_{node_entity}_nodes_(K+1).csv` — flattened CSV
      (lists joined with `|`, nulls preserved as empty cells, same shape as
      `nodes_(K+1).csv`).

   If a file mixes entity types, write **one set of output files per detected
   `{node_entity}`**, partitioning the rows accordingly.

5. **Column-name harmonization.**
   The original `db/external_protein_nodes.csv` carries its own context columns
   under whatever names the source database uses. These will not match the
   pipeline's 21 field names.
    - For each non-id/non-name column in the original file, ask the LLM
      (`gpt-4o`, single call) to map it to the closest of the 21 schema field
      names by **reasoning over the column's values**, not just the column
      name.
    - Build a mapping table `{original_column → pipeline_field | null}`. Allow
      many-to-one (collapse multiple source columns into one pipeline field by
      union) and `null` (no good match).
    - Apply the mapping to produce a *renamed* copy of the original file:
      `output/archive/external_{node_entity}_nodes_(K+1)_renamed.csv`. Leave any
      value with no mapping as `null`. Write the mapping itself to
      `output/archive/external_{node_entity}_column_mapping_(K+1).json`.

6. **Content comparison.**
   With both files now sharing column names, compare row-by-row on `id`. See
   *Benchmark metrics* below for what to compute. Write everything to
   `output/archive/external_benchmark_report_(K+1).md`.

#### Benchmark metrics

For each of the 21 fields that exists in **both** the agent-generated and the
harmonized external file, compute and report:

| Metric                              | Why it matters                                                                                                                                                          |
|-------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Both-populated rate**             | % of rows where both sides have a non-null value. Establishes the comparable subset.                                                                                    |
| **Both-null rate**                  | % where both sides agree the field is inapplicable. A weak form of agreement.                                                                                           |
| **Agent-only / external-only rate** | Asymmetry of coverage. Tells you whether the agent over- or under-fills relative to the external source.                                                                |
| **Jaccard similarity**              | For list-valued fields. `                                                                                                                                               |A ∩ B| / |A ∪ B|` per row, averaged over the both-populated subset.                                                                             |
| **Set precision / recall / F1**     | Treat external as ground truth, agent values as predictions. Report micro and macro across rows.                                                                        |
| **Exact-match rate**                | Strict equality of the two sets per row. Sanity check; expected to be low if vocabularies differ.                                                                       |
| **Cohen's κ (per field)**           | Binarize as "value present" or per-term, depending on field cardinality. Captures agreement above chance.                                                               |
| **Embedding cosine similarity**     | For terms that don't match exactly, embed both label sets (`text-embedding-3-small`) and compute mean pairwise cosine. Catches `"endothelial_cell"` vs `"endothelium"`. |
| **Top-K confusion matrix**          | For categorical fields with ≤30 terms, render a confusion matrix of agent label vs external label.                                                                      |

Aggregate these into a single report with three sections:

1. **Coverage agreement** — table of all 21 fields with the four rate columns.
2. **Content agreement** — table of all 21 fields with Jaccard, F1, κ,
   embedding cosine.
3. **Per-field deep dive** — for each field, the top 5 most-disagreeing terms
   and 2–3 example rows.

The headline number is a single **consistency score**: the macro-average F1
across all comparable fields, weighted by the both-populated rate. Report it
in the first line of the report.

#### Plots

All benchmark plots are written to `images/` with the prefix `external_`:

- `images/external_pca_context.png` — PCA of agent-generated context vectors
  for the external nodes (analogous to `pca_context.png`).
- `images/external_field_agreement.png` — grouped barplot of the four
  coverage-rate columns per field.
- `images/external_f1_by_field.png` — sorted barplot of per-field F1.
- `images/external_jaccard_distribution.png` — histogram of per-row Jaccard
  similarities (one panel per field with enough overlap).
- `images/external_reclassification_sankey.png` — Sankey of
  `type_from_name → type_from_id_and_name` from step 2.

#### What benchmarking does *not* do

- It does **not** modify or refine the schema.
- It does **not** update cumulative term frequencies, stability counts, or
  locked fields.
- It does **not** affect the per-iteration budget cap unless `--benchmark` is
  set; the benchmark run is budgeted separately (see *Cost*).
- It does **not** treat the external file as ground truth in any absolute
  sense — divergences are a measurement, not a verdict on either side.

### Budget

The user is prompted **once** for a per-iteration budget cap (in USD). This
same cap applies independently to each iteration. Total spend =
per-iteration budget × number of iterations (worst case).
Total spend should not exceed 20 USD.

#### Refinement summary format

For each controlled-vocabulary field, ranked by coverage % (highest first):

- **field_name** — coverage: XX% | applicable coverage: YY%
    - Terms added: term1, term2, ...
    - Count of terms added: N
    - Terms removed: term1, term2, ...
    - Count of terms removed: N

Where "coverage" = nodes with values / total, "applicable coverage" = nodes
with values / nodes where the LLM responded (excludes genuinely inapplicable).
All 21 fields must be listed. If no changes, show "Terms added: none" /
"Terms removed: none". Nothing else in the summary.

### Rules for this task

- Each iteration selects 500 new diverse nodes: cover all 9 entity types, mix
  high-degree and low-degree nodes. Samples are independently randomized.
- Use the LLM for entity context via Batch API or async API
- For each node, record which fields you could fill and which you could not
- When you encounter a value that fits a field but is not in the controlled
  vocabulary, add, rename, or merge vocabulary terms as appropriate
- **Schema fields are fixed** — do NOT add, remove, or rename fields. Only
  modify the controlled vocabulary term lists.
- **Maximum 30 unique terms** per controlled vocabulary. This is a hard limit
  enforced programmatically — vocabularies over 30 terms are truncated on save.
  Remove or merge terms before adding new ones if at the cap.
- **No null-like placeholder values** in any controlled vocabulary. Terms like
  "not_applicable", "unknown", "none", "not_specified", "none_known",
  "unclassified", "other", "not_a_drug", "not_organism_specific" are
  automatically stripped from vocabularies on save and from populated node
  values after Phase 2. If no vocabulary term fits a node's field, the value
  should be `null` (JSON null), not a placeholder term.
- Controlled-vocabulary fields return **lists** of labels, not single values
- Save at least 2 versioned checkpoints before finalizing
- When calling save_schema or finalize_schema, pass only the `controlled_vocabularies` dict (merged into base schema
  automatically)
- External benchmarking is **read-only with respect to the schema**. It must
  not call `save_schema`, `finalize_schema`, or `write_summary`. It writes only
  `external_*` files.

---

## Architecture

### Pipeline overview

The pipeline runs an outer loop of I iterations (default 10). Each iteration
chains from the previous one's finalized schema.

**Outer loop** (for iteration i = 1 … I):

1. Load the latest schema from `output/archive/schema_final_K.json` (highest K)
   and set the output run number to K+1, put the file name of the latest schema you used in summary report
2. Select 500 new diverse node IDs (fresh random sample)
3. **Phase 1 — Summarize** (Batch API or async, gpt-4o-mini):
   a. Build requests — one summarization request per node
   b. Submit and collect results
   c. Parse summaries
4. **Phase 2 — Populate** (Batch API or async, gpt-4o-mini):
   a. Build requests — one schema-mapping request per node
   b. Submit and collect results
   c. Parse populated node objects
   d. Update cumulative term frequencies across iterations
   e. Write populated nodes to `output/archive/nodes_(K+1).json`
5. **Phase 3 — Refine vocabularies** (synchronous agent loop, gpt-4o):
   a. Analyze Phase 2 results (coverage stats, this-iteration + cumulative
   term frequencies, suggested vocabulary additions; locked fields marked)
   b. Agent loop receives aggregate analysis + current schema (NOT all 500 nodes)
   c. Agent modifies only unlocked controlled vocabularies (not the fields),
   saves checkpoints, writes refinement summary, and finalizes
6. Clean up schema checkpoint intermediates from `output/archive/`
7. Compare finalized schema to starting schema; update per-field stability
   counts. Lock fields stable for 3+ consecutive iterations.
8. Output: `schema_final_(K+1).json`, `refinement_summary_(K+1).md`,
   `nodes_(K+1).json`, and `nodes_(K+1).csv` in `output/archive/`
9. **(Optional) External node benchmarking** — only if `--benchmark <path>`
   was passed and only after the final iteration:
   a. Categorize each row's entity type from `name` only (Batch API,
   gpt-4o-mini).
   b. Re-categorize using `id + name`; write
   `external_reclassification_report_(K+1).md`.
   c. Run Phase 1 (summarize) and Phase 2 (populate) against
   `schema_final_(K+1).json`. Skip Phase 3.
   d. Write `external_nodes_summary_(K+1).md`,
   `external_{node_entity}_nodes_(K+1).json`, and
   `external_{node_entity}_nodes_(K+1).csv` to `output/archive/`.
   e. Harmonize column names against the original external file via
   LLM-driven mapping; write
   `external_{node_entity}_column_mapping_(K+1).json` and
   `external_{node_entity}_nodes_(K+1)_renamed.csv`.
   f. Compute benchmark metrics (coverage agreement, Jaccard, F1, Cohen's κ,
   embedding cosine) and write `external_benchmark_report_(K+1).md`.
10. **(Optional) Generate external benchmarking plots** with the `external_`
    prefix (PCA, per-field agreement, F1, Jaccard, reclassification Sankey)
    into `images/`.

**After final iteration**: generate all plots (PCA, node types, term changes).

### Tools the agent has access to (Phase 3 only)

- `save_schema(controlled_vocabularies, version)` — checkpoint to `output/archive/` (vocabs are merged into base schema
  automatically)
- `finalize_schema(controlled_vocabularies)` — saves `schema_final_N.json` in archive, and converts
  `schema_final_N.json` to `schema_final_N.csv`, ends session (vocabs merged automatically)
- `write_summary(content)` — writes `refinement_summary_N.md` in archive

### Utility functions (used programmatically, not as agent tools)

- `get_type_distribution()` — understand graph composition
- `get_predicate_distribution()` — understand relationship types
- `sample_nodes(node_type, count, strategy)` — strategy: random | high_degree | low_degree
- `get_node_by_id(node_id)` — fetch a single node
- `build_summarize_request(node, custom_id)` — build a Phase 1 JSONL line
- `build_populate_request(node, summary, schema, custom_id)` — build a Phase 2 JSONL line
- `submit_batch(jsonl_path)` — upload file and create batch
- `poll_batch(batch_id)` — poll until completed, return output file ID
- `download_batch_results(output_file_id, dest_path)` — download output JSONL

### Schema output format

The final schema_final.json should include:

- `fields`: array of field definitions, each with:
    - `name` (snake_case)
    - `description`
    - `field_type`: string (id/name only) | controlled (everything else)
    - `controlled_vocabulary`: vocabulary name (key in controlled_vocabularies)
    - `applies_to_types`: list of node types, or empty for universal
    - `required`: boolean
- `controlled_vocabularies`: dict of vocab name → value list (8–30 terms each, max 30)
- `type_specific_fields`: dict of node type → field names
- `notes`: agent's observations about the domain

### Populated nodes output format

The output/archive/nodes_N.json should be an array of objects, one per test node.
Each controlled-vocabulary field is a **list** of matching labels (an entity may
belong to multiple categories). Fields that could not be determined are `null`.

```json
[
  {
    "id": "DOID:0001816",
    "name": "angiosarcoma",
    "organism": ["Homo sapiens"],
    "tissue_location": ["blood", "soft_tissue"],
    "cell_type": ["endothelial"],
    "cellular_compartment": null,
    "biological_system": ["cardiovascular"],
    ...
  }
]
```

Each value in a list must come from the corresponding controlled vocabulary.

The output/archive/nodes_N.csv should be a csv file (dataframe object), each node as row and each with each node as a
row and each 21 novel biological context field as a column. In case the value is a list, separate each list object
using "|". Leave the `null` as in the `nodes_N.json`.

Each column name and row values must come from the corresponding controlled vocabulary.

---

## Stack and conventions

- **Language**: Python 3.11+
- **LLM**: OpenAI API via `openai` Python SDK (synchronous client)
- **Model**: `gpt-4o-mini` for Phase 1 & 2; `gpt-4o` for Phase 3 agent loop
- **Batch API**: OpenAI Batch API for Phase 1 and Phase 2 (JSONL upload, poll, download)
- **Embeddings**: `text-embedding-3-small` via the OpenAI API, used only by
  the external benchmarking comparator
- **Validation**: `pydantic` for all structured output
- **Checkpointing**: write to `output/archive/` after every meaningful step
- **Environment**: `OPENAI_API_KEY` set in `.env`

### Code conventions

- Use snake_case everywhere
- All tool functions return a plain dict (serialized to JSON for the API)
- Keep tool functions pure — no side effects except `save_schema`, `finalize_schema`, `write_summary`, `write_nodes`
- Hard ceiling of 40 agent turns in Phase 3 to prevent runaway loops
- Print progress and tool calls to stdout so the run is observable
- Phase 1 & 2 support two modes: synchronous Batch API or async direct API calls (`--mode batch` or `--mode async`)
- Multi-iteration support via `--iterations N` flag (default 10)
- Plots are generated automatically after the final iteration

---

## File structure

```
scripts/
  schema_agent.py     # main pipeline: Phase 1/2 (batch or async) + agent loop (Phase 3)
  plot_pca.py         # ad-hoc PCA plotting script
  plot_node_types.py  # stacked barplot of node type distribution per iteration
  color_scheme.py     # colorblind-friendly palette for entity types
  tools/
    graph_tools.py    # sample_nodes, get_type_distribution, get_predicate_distribution
    batch_tools.py    # build_summarize_request, build_populate_request, submit_batch, poll_batch, download_batch_results
    async_tools.py    # async direct-API alternatives to batch_tools (Phase 1 & 2)
    schema_tools.py   # load_latest_schema, save_schema, finalize_schema, write_summary, write_nodes, cleanup_checkpoints
output/
  archive/            # all versioned outputs (schema_final_N.json, refinement_summary_N.md, nodes_N.json, nodes_N.csv)
                      # external benchmarking outputs (all prefixed external_):
                      #   external_nodes_summary_N.md
                      #   external_{node_entity}_nodes_N.json
                      #   external_{node_entity}_nodes_N.csv
                      #   external_{node_entity}_nodes_N_renamed.csv
                      #   external_{node_entity}_column_mapping_N.json
                      #   external_reclassification_report_N.md
                      #   external_benchmark_report_N.md
  batches/
    batch_ids/        # one JSON file per batch submission (batch_id, phase, metadata)
    inputs/           # JSONL files submitted to the Batch API (phase1_batch_001.jsonl, etc.)
    outputs/          # JSONL files returned by the Batch API (phase1_batch_001_output.jsonl, etc.) 
                      # CSV files converted from the JSONL files (phase1_batch_001_output.csv, etc.)
images/               # ad-hoc plots (PCA, etc.)
                      # external benchmarking plots (all prefixed external_):
                      #   external_pca_context.png
                      #   external_field_agreement.png
                      #   external_f1_by_field.png
                      #   external_jaccard_distribution.png
                      #   external_reclassification_sankey.png
db/
  nodes.csv                   # source node data (250k nodes, 9 entity types)
  protein_nodes.csv           # test node data (10225 nodes, 1 entity type)
  external_protein_nodes.csv  # first external benchmark file (UniProtKB-keyed proteins)
```
