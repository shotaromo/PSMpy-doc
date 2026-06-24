# PSMpy user guide

A practical **how-to** for running PSMpy. The `docs/spec/` files and `docs/adr/`
explain *why* the model is built the way it is; this guide explains *how* to
install it, author an input dataset, validate, compile, solve, and report.

> Scope today: the first vertical slice is **global2050 · Japan · `power`**
> (ADR-0007). The authoring/solving/reporting layers and a toy ladder are on
> `main`; the real Japan run is HPC-gated (`docs/STATUS.md`). The unified `psmpy`
> CLI (`validate` / `sweep` / `extract` / `run`) covers the shell workflow; the
> Python API below is the in-process equivalent (and what the CLI calls).

---

## 1. Install & environment

PSMpy uses [`uv`](https://docs.astral.sh/uv/) with a per-repo `.venv` (user-local
on the HPC — home directory only; see `CLAUDE.md`).

```bash
uv sync                 # core deps (numpy / pandas / pyarrow / pyyaml)
uv sync --extra solve   # ALSO install gamspy — needed to SOLVE (and to read a GDX)
uv sync --extra report  # ALSO install pyreport — needed for the IAMC report (§5)
uv run pytest tests/    # test suite (solver / report tests auto-skip without their extra)
```

- **Reading a GDX and solving need `gamspy`** (the `solve` extra). The package
  itself is free; *solving* uses your existing GAMS/solver licence (e.g. on the
  HPC). Authoring (load / validate / **compile**) needs neither.
- **The IAMC report needs `pyreport`** (the `report` extra) — the shared Track P
  reporting core, pure-pandas (no solver, no licence). See §5.
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

**From the shell — `psmpy run`** (the usual way for a real dataset; forwards to
`scripts/solve_case.py`, which owns the solve + writes the results):

```bash
uv run --extra solve psmpy run \
    --dataset datasets/japan_power \
    --case runs/japan_power_300C \
    --years 2020:2050:5            # start:stop:step
    # --report                     # also emit the IAMC layer (§5; needs --extra report)
```

It validates → compiles → solves the myopic loop, writing `runs/<case>/solve/
<year>/*.parquet` (the solved levels), `state/<year>/` (carry-over), and
`summary.json`. `psmpy run --help` lists the solver knobs (`--solver`,
`--crossover`, `--threads`, …). `--dry-run` stops after compile (no licence).

**From Python** — two routes. High-level facade:

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

## 5. Reporting — the IAMC layer (needs the `report` extra)

PSMpy **reuses the shared Track P reporting core** (the `pyreport` package) rather
than re-implementing report math — "one core, two adapters" (`docs/spec/
postprocess.md` §4): `io_gdx` is the GAMS-side adapter, `psmpy.reporting.report` is
the PSMpy-side one. It is pure-pandas (no gamspy, no licence); install it with
`uv sync --extra report`.

**Turnkey — `psmpy run --report`.** The simplest path: add `--report` to the solve
(§4). After the solve it feeds the in-memory result through the shared core and writes
the wide IAMC table to `runs/<case>/report/iamc/<case>.csv`:

```bash
uv run --extra solve --extra report psmpy run \
    --dataset datasets/japan_power --case runs/japan_power_300C --report
```

The emit is non-fatal — a missing `report` extra (or any reporting error) is logged,
never aborts a solve whose results are already on disk.

**From Python — in-process.** `from_solution` turns a live `RunResult` into the IAMC
long frame via the shared core; `write_iamc` writes the wide CSV:

```python
from psmpy.runtime.loop import run_case
from psmpy.reporting import from_solution, write_iamc

sf = model.compile()
result = run_case(sf, years=[2030, 2035], t_int=5, start_year=2030, case_dir="runs/toy")

iamc = from_solution(result, model._ir(), sf)   # -> IAMC long [var, region, year, value, label, unit]
write_iamc(iamc, "runs/toy", scenario="toy")     # -> runs/toy/report/iamc/toy.csv (wide pyam layout)
```

**Post-hoc — `from_run_dir`.** Reproduce the IAMC from a solved case already on disk
(reads `runs/<case>/solve/<year>/*.parquet` + recompiles from the dataset — **no
re-solve, no licence**):

```python
from psmpy.reporting import from_run_dir
iamc = from_run_dir("runs/japan_power_300C", "datasets/japan_power")
```

**What's reported today.** The families that carry data: **capacity**,
**capacity_addition**, **emission**, and **energy_use / carbon_capture / Primary
Energy** (the last for the mapped `P_*` primary-supply carriers). `capex` is emitted
but ~0 (cost params pending); `generation` is computed but is an intermediate, not an
IAMC variable. The per-slice **dispatch** families (supply / withdrawal / losses /
curtailment) are deferred behind the HPC parity gate — they are the parity-sensitive
Phase-1b quantities (`docs/STATUS.md`). Plots / dashboard reuse `pyreport`'s plot CLI
on the merged IAMC (a follow-up).

> A decoupled alternative — `psmpy.reporting.to_report` / `write_report` write the raw
> Report-schema frames to parquet for a SEPARATE `pyreport` process (no `pyreport`
> import); use it only when the two repos must stay process-decoupled.

`model._ir()` is the current IR accessor. See `src/psmpy/reporting/report.py` and
`docs/spec/postprocess.md` (the quantity catalog + IAMC export).

---

## 6. Extraction (M0) — build a dataset from a GAMS input GDX

To produce the canonical tables from the reference GAMS input GDX (needs
`gamspy` + the GDX file):

```bash
# one step (anywhere with the GDX + gamspy):
uv run --extra solve psmpy extract gams \
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

The unified `psmpy` CLI (prefix with `uv run`; add `--extra solve` for
`run`/`extract`, `--extra report` for `run --report`):

| Command | Purpose |
| --- | --- |
| `psmpy validate <dataset>` | load + validation tiers 1–3 (no solver); exit 2 on errors |
| `psmpy sweep <sweep.yaml> --dataset <dir> --out <dir>` | expand a sweep into N case dirs (no solver) |
| `psmpy extract gams --gdx <GDX> --out <dir>` | GDX → canonical dataset (needs `--extra solve`) |
| `psmpy run --dataset <dir> --case <dir> [--report]` | solve myopically, optionally emit IAMC (§4–5; `psmpy run --help` for solver knobs) |
| `uv run pytest tests/` | full suite (solver / report tests auto-skip without their extra) |
| `uv run python -m psmpy.units --dump [--out PATH]` | emit the canonical unit-token contract (JSON) |

`psmpy run` forwards to `scripts/solve_case.py` — the solve needs the GAMS licence,
so it is intentionally outside the importable package API. The five-category
scenario config + overlays (M3, `docs/spec/scenario.md`) drive `sweep`; a richer
scenario CLI surface is future work.

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
