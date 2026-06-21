---
icon: material/database
---

# Model data

!!! note "Shared with PSMpy v0.1 via the canonical tables"
    These are the data assumptions of the reference GAMS `PowerSystemModel`.
    PSMpy consumes the **same** data as validated canonical tables (see the model
    repo's `docs/spec/data_schema.md`), extracted from the GAMS inputs or produced
    by the data pipeline — so the assumptions below apply to PSMpy too.

This page documents the data assumptions underlying each sector, covering cost and efficiency parameters, potential estimates, and existing capacity assumptions.

## Power sector

### Cost and efficiency

!!! warning "TODO"
    Describe the sources and methodology used to derive capital costs, fixed/variable O&M costs,
    and efficiencies for each generation technology (e.g., coal, gas, solar, wind, nuclear).
    Include the base year, any learning curves applied, and regional differentiation if applicable.

**Sources** *(datasets the PSM-Data pipeline draws on; specific values, base year, and methodology to be finalized by the data team)*

- **IEA World Energy Outlook (WEO 2023)** — overnight capital and fixed/variable O&M cost by technology (`prog/inc_prog/power_cost.R`).
- **DEA Technology Catalogue** — efficiency / heat-technology parameters for selected technologies.

**Methodology**

*(How capital costs, fixed/variable O&M costs, and efficiencies were derived from the sources.)*

**Notes**

*(Base year, currency year, learning curves, regional differentiation, known limitations.)*

For the full list of cost and efficiency parameters and their GAMS names, see the [Appendix](appendix.md).

### Potential

!!! warning "TODO"
    Describe how the available potential for variable renewables (solar PV, wind) was estimated.
    Cover the geographic scope, data sources (e.g., reanalysis data, GIS), and how the
    capacity factor profiles (`gmx`) were derived from resource data.

**Sources** *(datasets the PSM-Data pipeline draws on; processing details to be finalized by the data team)*

- **Reanalysis-derived gridded capacity factors** (NetCDF) — hourly solar-PV and wind capacity-factor profiles feeding `gmx(N,I,R,T)` (`prog/inc_prog/potential.R`).
- **Franzmann et al. (2025)** renewable resource/plant dataset + PV/wind **resource-grade curves** — siting potential and per-grade capacity limits.

**Methodology**

*(Geographic scope, resource data processing, and how capacity limits and the capacity factor profiles `gmx(N,I,R,T)` were derived.)*

**Notes**

*(Exclusion criteria, land-use assumptions, known limitations.)*

### Existing capacity

!!! warning "TODO"
    Describe how the existing installed capacity stock was constructed.
    Cover the data sources for historical installation records, the survival function used
    to compute surviving vintage stock `gssc(N,I,R,Y)`, and any adjustments made.

**Sources** *(datasets the PSM-Data pipeline draws on; survival function + calibration to be finalized by the data team)*

- **IEA Energy Balances** (`ieaeb`) — historical generation/capacity by technology (`prog/inc_prog/plants.R`).
- Per-technology **capacity datasets** (renewable / hydro / geothermal) combined with **survival functions** to build the surviving vintage stock `gssc(N,I,R,Y)`.

**Methodology**

*(Historical installation records and the survival function used to compute the surviving vintage stock `gssc(N,I,R,Y)`.)*

**Notes**

*(Adjustments made, base-year calibration, known limitations.)*

---

## Hydrogen sector

### Cost and efficiency

!!! warning "TODO"
    Describe the cost and efficiency assumptions for hydrogen production technologies
    (e.g., electrolysers, SMR with/without CCS).
    Include capital costs, fixed O&M, and conversion efficiencies (`geta` or `feta`).

**Sources** *(to be documented by the data team)*

- Hydrogen production splits by element type (see [Model structure](structure.md)): **Source**-type routes — SMR, coal-to-H2 (GAMS generators `g*`) — are parameterised in `data/parameter/generator.gms`, while **Conversion**-type **electrolysers** (GAMS links `f*`, `L2.gms` tech `electrolysis`) carry their cost/efficiency (`fcc`/`facc`/`feta`) in `data/parameter/link.gms`. Their external cost/efficiency sources are **not yet documented** in the PSM-Data pipeline.

**Methodology**

*(How capital costs, fixed O&M costs, and conversion efficiencies (`geta` or `feta`) were derived from the sources.)*

**Notes**

*(Base year, currency year, regional differentiation, known limitations.)*

For the full list of cost and efficiency parameters and their GAMS names, see the [Appendix](appendix.md).

### Potential

!!! warning "TODO"
    Describe any constraints on hydrogen production potential, such as limits on
    renewable electricity available for electrolysis or geological storage capacity.

**Sources** *(to be documented by the data team)*

- Production/storage constraints are set in the **GAMS data / scenario layer**; specific sources are **not yet documented**.

**Methodology**

*(How production limits and storage capacity constraints were derived.)*

**Notes**

*(Scope of constraints, known limitations.)*

### Existing capacity

!!! warning "TODO"
    Describe how existing hydrogen production capacity was estimated,
    including data sources and vintage stock construction.

**Sources** *(to be documented by the data team)*

- Existing hydrogen production capacity is generally **negligible at the base year**; where present, vintage stock follows the same survival-function construction as the power sector. **To be confirmed.**

**Methodology**

*(Data sources and vintage stock construction for existing hydrogen production capacity.)*

**Notes**

*(Adjustments made, known limitations.)*
