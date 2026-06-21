# PSMpy user guide

A practical **how-to** for running PSMpy. The `docs/spec/` files and `docs/adr/`
explain *why* the model is built the way it is; this guide explains *how* to
install it, author an input dataset, validate, compile, solve, and report.

> Scope today: the first vertical slice is **global2050 · Japan · `power`**
> (ADR-0007). The authoring/solving/reporting layers and a toy ladder are on
> `main`; the real Japan run is HPC-gated (`docs/STATUS.md`). There is **no
> unified solve CLI yet** — solving is driven through the Python API below.

---

## 1. Install & environment

PSMpy uses [`uv`](https://docs.astral.sh/uv/) with a per-repo `.venv` (user-local
on the HPC — home directory only; see `CLAUDE.md`).

```bash
uv sync                 # core deps (numpy / pandas / pyarrow / pyyaml)
uv sync --extra solve   # ALSO install gamspy — needed to SOLVE (and to read a GDX)
uv run pytest tests/    # test suite (solver tests auto-skip without a GAMS licence)
```

- **Reading a GDX and solving need `gamspy`** (the `solve` extra). The package
  itself is free; *solving* uses your existing GAMS/solver licence (e.g. on the
  HPC). Authoring (load / validate / **compile**) needs neither.
- Everything runs **user-local** — no sudo, no system Python.

---

## 2. Five-minute path — load, validate, compile a toy (no solver)

The smallest end-to-end fixture is `tests/data/toy/` (one region, one carrier,
two sources, a demand). This runs with **no GAMS licence**:

```python
from pathlib import Path
from psmpy.authoring import EnergyModel

model = EnergyModel.from_dataset(Path("tests/data/toy"))   # tables/ + YAML -> validated IR
print(len(model.sources), "sources,", len(model.sinks), "sink(s)")

report = model.validate()          # tier-1 schema + tier-2/3 element & cross rules
assert report.ok, [f"{e.id}: {e.message}" for e in report.errors]

sf = model.compile(time_resol="12DAY")   # IR -> SymbolFrames (year-independent symbols)
print("compiled:", type(sf).__name__)
```

`validate()` returns a `ValidationReport` (`.ok`, `.errors`, `.warnings`,
`.issues`, `.summary`, `.tier_errors`). `compile()` returns an opaque
`SymbolFrames` object — pass it to the solve loop (§4); it is **not** a dict.

---

## 3. Authoring your own dataset

Inputs are **data, not code** (ADR-0003). A dataset is a directory of canonical
tables (CSV/Parquet) plus YAML for buses/groups. Use `tests/data/toy/` as the
template; the closure for a `power` slice is:

| File | Holds |
| --- | --- |
| `regions.csv` | region ids |
| `carriers.csv` | `carrier, kind, unit` (every numeric column has a declared unit) |
| `buses.yaml` | structured buses `(carrier, segment)` (ADR-0004) |
| `acct_categories.csv` | accounting categories (`K`) |
| `years.csv` | `year, is_vintage_only` |
| `source_techs.csv` / `sources.csv` | tech catalog + instances (catalog ⊕ instance override) |
| `source_accounting.csv` | per-source accounting (`geta`) |
| `dispatch_profiles.csv` | per-timeslice availability (optional) |
| `demand_annual.csv` / `demand_profile.csv` | Sink demand + shape |
| `links*`, `storages*`, `groups`, `emission_*`, `supply_limits`, `share_constraints` | added for richer cases (all optional) |

- The canonical schema (keys/units/dtypes) lives in `src/psmpy/authoring/schema.py`
  (`CANONICAL_SCHEMAS`) and is specified in `docs/spec/data_schema.md`.
- The **single canonical writer** is `psmpy.authoring.write.write_dataset` (both
  the R producer and the GDX-extract path emit the identical layout).
- Flat root or a `tables/` subdir both load. Validation localizes errors
  (broken ref, `min>max`, unreachable sink) with `AUTH-*` ids.

---

## 4. Solving (needs `gamspy` + a GAMS licence)

Two routes. High-level facade:

```python
from psmpy.runtime.loop import RunOptions

result = model.run(RunOptions(
    years=[2030, 2035],      # solve grid (defaults to the non-vintage years)
    start_year=2030,
    t_int=5,                 # interval length (years)
    case_dir="runs/toy",     # per-year carry-over state is written here
))
```

Explicit (compile once, then the myopic loop):

```python
from psmpy.runtime.loop import run_case

result = run_case(
    model.compile(),
    years=[2030, 2035], t_int=5, start_year=2030, case_dir="runs/toy",
)
```

- `run_case` builds a **fresh GAMSpy container per solve-year**; inter-year
  carry-over (cohorts `gsc/fsc/esc`, the exhaustible ledger `pmax_ex`) is held as
  pandas and dumped to `runs/<case>/state/<year>/*.parquet` (ADR-0010). Restart
  from any year with `from_year=`.
- Returns a `RunResult`: `.results` (`{year: SolveResult}` with the solved
  `vgs`/`vgr`/`vg`/… and objective), `.statuses`, `.reports` (post-solve
  CHK-3xx), `.year_inf` (first infeasible year, if any), `.degraded`.
- **Without `gamspy`/a backend** the path stops cleanly at `compile()` (§2);
  `run`/`run_case` is where the licence is required. The loop records an
  infeasible year rather than aborting the whole run.

---

## 5. Reporting

Solved results convert to Track P **Report-schema** frames via the reporting
adapter (reuses the pure-pandas Track P core — no gamspy). `to_report` needs
three inputs — the `RunResult`, the **IR** (for the bus decode + region
hierarchy) and the compiled **`SymbolFrames`** — so use the explicit route
where `sf` is in scope:

```python
from psmpy.runtime.loop import run_case
from psmpy.reporting.adapter import to_report, write_report

sf = model.compile()
result = run_case(sf, years=[2030, 2035], t_int=5, start_year=2030, case_dir="runs/toy")

frames = to_report(result, model._ir(), sf)   # RunResult + IR + SymbolFrames -> frames
write_report(frames, "reports/toy")            # parquet, consumed by tools/pyreport
```

`model._ir()` is the current IR accessor (a public reporting facade is a natural
future addition). See `src/psmpy/reporting/adapter.py` for the exact signature
and `docs/spec/postprocess.md` for the quantity catalog and IAMC export; the
`tools/pyreport/` package (in `PowerSystemModel-master`) turns these frames into
tables / IAMC / plots.

---

## 6. Extraction (M0) — build a dataset from a GAMS input GDX

To produce the canonical tables from the reference GAMS input GDX (needs
`gamspy` + the GDX file):

```bash
# one step (anywhere with the GDX + gamspy):
uv run --extra solve python -m psmpy.extract.run_extract \
    --gdx path/to/input/300C.gdx --out datasets/japan_power --format parquet
```

On the HPC the read is split: `dump_symbols` on a compute node (reads the
~352 MB GDX, writes symbol parquet), then `extract_dataset` anywhere (no
licence). The extractor body is GDX-decoupled — `run_extraction` injects the
read via a `symbols_provider`, so the pure transforms are unit-tested with
synthetic symbol dicts (no real GDX). See `docs/STATUS.md` (M0) and
`docs/hpc_runbook.md`.

---

## 7. Command reference

| Command | Purpose |
| --- | --- |
| `uv run pytest tests/` | full suite (solver tests auto-skip without a licence) |
| `uv run python -m psmpy.units --dump [--out PATH]` | emit the canonical unit-token contract (JSON) |
| `uv run --extra solve python -m psmpy.extract.run_extract --gdx GDX --out DIR` | GDX → canonical dataset |

There is **no `psmpy solve` CLI yet** — drive solving from Python (§4). The
five-category scenario config + overlays (M3) exist as an API
(`docs/spec/scenario.md`); a CLI surface is future work.

---

## 8. Where to read next

| You want… | Read |
| --- | --- |
| the big picture / rules | `CLAUDE.md`, `docs/adr/` |
| current implementation state | `docs/STATUS.md` |
| input tables + units | `docs/spec/data_schema.md` |
| the IR / validation / compile | `docs/spec/authoring.md` |
| equations / the solve | `docs/spec/solving.md` |
| scenarios / overlays | `docs/spec/scenario.md` |
| reporting (Track P) | `docs/spec/postprocess.md` |
| names (GAMS ↔ authoring ↔ reporting) | `docs/naming.md` |
| deliberate GAMS deviations | `docs/spec/deviations.md` |
