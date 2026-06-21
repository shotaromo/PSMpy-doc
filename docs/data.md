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

**Sources**

- *(source 1)*
- *(source 2)*

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

**Sources**

- *(source 1)*
- *(source 2)*

**Methodology**

*(Geographic scope, resource data processing, and how capacity limits and the capacity factor profiles `gmx(N,I,R,T)` were derived.)*

**Notes**

*(Exclusion criteria, land-use assumptions, known limitations.)*

### Existing capacity

!!! warning "TODO"
    Describe how the existing installed capacity stock was constructed.
    Cover the data sources for historical installation records, the survival function used
    to compute surviving vintage stock `gssc(N,I,R,Y)`, and any adjustments made.

**Sources**

- *(source 1)*
- *(source 2)*

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

**Sources**

- *(source 1)*
- *(source 2)*

**Methodology**

*(How capital costs, fixed O&M costs, and conversion efficiencies (`geta` or `feta`) were derived from the sources.)*

**Notes**

*(Base year, currency year, regional differentiation, known limitations.)*

For the full list of cost and efficiency parameters and their GAMS names, see the [Appendix](appendix.md).

### Potential

!!! warning "TODO"
    Describe any constraints on hydrogen production potential, such as limits on
    renewable electricity available for electrolysis or geological storage capacity.

**Sources**

- *(source 1)*
- *(source 2)*

**Methodology**

*(How production limits and storage capacity constraints were derived.)*

**Notes**

*(Scope of constraints, known limitations.)*

### Existing capacity

!!! warning "TODO"
    Describe how existing hydrogen production capacity was estimated,
    including data sources and vintage stock construction.

**Sources**

- *(source 1)*
- *(source 2)*

**Methodology**

*(Data sources and vintage stock construction for existing hydrogen production capacity.)*

**Notes**

*(Adjustments made, known limitations.)*
