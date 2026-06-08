# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dev dependencies
uv pip install -e ".[dev]"

# Lint / format
pre-commit run --all-files
black --check . --line-length 120
flake8 data_juicer/

# Run tests
python tests/run.py                                    # partial mode (changed files only)
python tests/run.py --mode regression                  # all tests
python tests/run.py --tag ray                          # Ray-mode tests
python tests/run.py --tag standalone --pattern test_my_op.py  # specific file

# Run a single test case
python -m pytest tests/ops/filter/test_my_op.py::TestMyOp::test_method -xvs

# CLI entry points
dj-process --config demos/process_simple/process.yaml
dj-analyze --config demos/analyze_simple/analyze.yaml
dj-install --env all
dj-mcp
```

## Architecture

### Data flow

```
YAML config → Config init → Executor → NestedDataset.process(ops) → export
```

1. A YAML config defines the `dataset_path`, a `process` list of operator specs, and an `export_path`.
2. `Executor.run()` resolves the config, loads the dataset via `DJDataset`, and feeds it through a chain of operators.
3. Operators transform the dataset in-place via `dataset.process(operators)`, which maps to Hugging Face `Dataset.map` calls.
4. Results are exported (JSONL, parquet, etc.) by the `Exporter`.

### Operator system (`data_juicer/ops/`)

The core abstraction. Every data transformation is an operator subclass:

| Base class | Purpose | Process signature |
|---|---|---|
| `Filter` | Keep/discard samples | `process(sample) → bool` |
| `Mapper` | Modify samples | `process(sample) → sample` |
| `Deduplicator` | Remove near-duplicates | `process(dataset) → dataset` |
| `Selector` | Subset/sample the dataset | `process(dataset) → dataset` |
| `Grouper` | Sort/reorder samples | `process(dataset) → dataset` |
| `Aggregator` | Aggregate batch-level metadata | `process_batch(samples) → stats` |
| `Pipeline` | Compose sub-ops into a logical unit | internal sub-ops run sequentially |

All operators are registered in `OPERATORS` (a `Registry` instance) via `@OPERATORS.register_module()`. The YAML config references operators by their registered name.

Operator categories live under `data_juicer/ops/{filter,mapper,deduplicator,selector,grouper,aggregator,pipeline}/`.

### Key infrastructure

- **`Registry`** (`data_juicer/utils/registry.py`): Named module registry — operators, models, and other pluggable components are registered and looked up by string name. Powers the config-to-class resolution.
- **`NestedDataset`** (`data_juicer/core/data/dj_dataset.py`): Wraps Hugging Face `Dataset`/`DatasetDict` with support for nested multimodal data (text + images + audio + video in a single sample). The `process()` method applies operators by calling `map`/`filter` on the underlying HF dataset.
- **`Executor`** (`data_juicer/core/executor/`): Factory pattern — `DefaultExecutor` for single-machine, `RayExecutor` for distributed, `PartitionedRayExecutor` for large datasets with partitioning.
- **`Tracer`** (`data_juicer/core/tracer/`): Sample-level change tracking — intercepts mapper/filter calls to record before/after for debugging and visualization. Wrapped around ops via `wrap_mapper_with_tracer` / `wrap_filter_with_tracer`.
- **Config system** (`data_juicer/config/config.py`): `jsonargparse`-based, merging CLI args, YAML files, and defaults. The `init_configs()` function returns a `Namespace` used throughout the pipeline.
- **`Fields`** (`data_juicer/utils/constant.py`): Reserved column name prefixes (`__dj__stats__`, `__dj__meta__`, etc.) used to attach operator statistics and metadata to samples without colliding with user columns.

### Directory map

```
data_juicer/
  core/          Data model, executor, exporter, monitor, tracer
  ops/           All operators (filter, mapper, deduplicator, selector, ...)
  config/        YAML config parsing and merging
  tools/         CLI entry points (process_data, analyze_data, dj_install, mcp_server)
  utils/         Registry, constants, compression, model utils, mm utils
  analysis/      Data analysis and visualization
  format/        Format conversion (jsonl, csv, parquet, ...)
  download/      Dataset download helpers
demos/           Example pipeline configs and data
tests/           Mirrors source tree; test_*.py files, run via tests/run.py
```

### Operator conventions

- Batched operators define `_is_batched_op = True` and receive `List[Dict]` / `Dict[List]` instead of single samples.
- Stats filters set `_is_stats_filter = True` — they compute stats via a mapper side-effect and then filter on the stored stats column.
- Forkability: operators that spawn subprocesses are registered in `UNFORKABLE` to avoid nested multiprocessing.
- GPU ops use `accelerate` (preferred) or raw CUDA; check `is_cuda_available()` from `resource_utils`.

### Test structure

Tests mirror the source tree under `tests/`. Two execution modes:
- **`standalone`**: Single-process, tagged with `@pytest.mark.standalone` (or default).
- **`ray`**: Distributed mode, tagged with `@pytest.mark.ray` (via `__test_tags__` on methods).

Run via `tests/run.py` (not bare `pytest`) to get partial-mode detection and coverage reporting.

## Agent skills

### Issue tracker

Issues and PRDs live as GitHub issues on `yepingzhao/data-juicer`. See `docs/agents/issue-tracker.md`.

### Triage labels

All five canonical triage roles use their default label names (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context layout — one `CONTEXT.md` + `docs/adr/` at the repo root. See `docs/agents/domain.md`.
