---
icon: material/home
---

# PSMpy

**PSMpy v0.5** is a Pythonic rebuild of the GAMS `PowerSystemModel` as a
[GAMSpy](https://gamspy.readthedocs.io/) model — a multi-year linear program for
capacity-expansion and dispatch optimization of energy systems. The energy
system is expressed with fixed physical element types — **Source / Conversion /
Transmission / Storage** (+ **Sink** for demand) — and the spatial scope,
temporal resolution and sector scope are configurable per run.

## Key ideas

- **Data-first authoring** — inputs are validated tables + YAML, not code; the
  component classes are the in-memory IR (ADR-0003).
- **GAMS-equivalent math, Pythonic structure** — domains/symbol names stay 1:1
  with GAMS; the ~30 lifecycle/investment equations GAMS repeats three times are
  generated once by a single generic stock block, instantiated ×3 (ADR-0001).
- **Structured buses** `(carrier, segment)` in authoring/reporting; GAMS `I`
  labels exist only in compiled symbols (ADR-0004).
- **Scope = non-instantiation** — out-of-scope elements are never created (no
  eps capacities, no fix-to-zero); scope axes are config + data, never code
  forks (ADR-0012).
- **Reporting** reuses the pure-Python Track P core (`tools/pyreport`) — xarray
  /pandas, no solver needed.

## Start here

| Page | Contents |
| --- | --- |
| [User guide](user_guide.md) | install · author a dataset · validate · compile · solve · report |

For the *why* behind the design, read `CLAUDE.md`, the decision records under
`docs/adr/`, and the specifications under `docs/spec/`. Implementation state is
tracked in `docs/STATUS.md`; the milestone plan is `docs/roadmap.md`.

> First vertical slice: **global2050 · Japan · `power`** (ADR-0007). The
> authoring/solving/reporting layers and a demand-sector toy ladder are built;
> the real Japan run is HPC-gated.
