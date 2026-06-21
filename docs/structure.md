---
icon: material/sitemap
---

# Model structure

!!! note "PSMpy v0.1 — a GAMS-equivalent rebuild"
    PSMpy is a Pythonic rebuild of the GAMS `PowerSystemModel`. The solving
    structure below (nodes, sectors, generators / links / storage, timeslices,
    stock) is kept equivalent to GAMS; in authoring these map to the fixed
    element types **Source / Conversion / Transmission / Storage / Sink**, built
    data-first (ADR-0003 / ADR-0008).

PSMpy v0.1 represents an energy system as a network of **nodes** ($n \in N$) and **sectors** ($i \in I$). Three technology types connect or serve these node-sector pairs: **generators**, **links**, and **storage**.

## Technology components

### Generators

Generators ($r \in R$) produce energy within a node-sector pair $(n,i)$. Their dispatch $VX_{n,i,r,t}$ is bounded by installed capacity $VS_{n,i,r}$ and time-dependent availability coefficients. Generator output feeds the nodal power balance and—via carrier efficiency coefficients $\eta_{n,i,r,k}$—the energy supply variable $VP_{n,i,k}$ used in emission accounting.

### Links

Links ($l \in L$) transfer or convert energy between node-sector pairs. Separate forward and backward flow variables ($VX^+_{l,t}$, $VX^-_{l,t}$) allow asymmetric efficiency and directional constraints. The same formulation covers unidirectional devices (e.g., electrolysers) and bidirectional devices (e.g., grid interconnectors).

### Storage

Storage ($s \in S$) couples dispatch across timeslices through the state-of-charge variable $VE_{n,i,s,t}$. Charge/discharge power $VX_{n,i,s,t}$ enters the nodal power balance, while energy capacity $VS_{n,i,s}$ is tracked separately. A capacity ratio parameter $\chi$ links the power capacity of an associated link (e.g., pump/turbine) to the energy capacity of the reservoir.

## Temporal structure

| Dimension | Symbol | Role |
|-----------|--------|------|
| Simulation years | $y \in \text{YEAR}(Y)$ | Investment, retirement, and cost annualization |
| Timeslices | $t \in T$ | Intra-year dispatch and storage dynamics |

Timeslice weight $\omega$ (hours) converts power to energy in annual sums; granularity $\Delta t$ (hours) governs rate-based constraints such as ramping limits and self-discharge.

## Technology stock

All three technology types share the same stock accounting structure. Installed capacity $VS$ is composed of new investment $VR$, replacement $VP$, surviving vintage stock $sc$ (exogenous), retrofit flows $VC$, and a stock-balance slack $\delta^{\text{stock}}$:

$$VS = (VR + VP) \cdot \Delta y + \sum_y \!\left( sc_y + \sum_{\text{retrofit in}} VC - \sum_{\text{retrofit out}} VC \right) - \delta^{\text{stock}}$$

| Term | Meaning |
|------|---------|
| $VR$ | New installation added per simulation step |
| $VP$ | Replacement of vintage stock at end of life |
| $sc_y$ | Surviving vintage stock (pre-computed from historical records) |
| $VC$ | Retrofit flows (endogenous technology conversion) |
| $\delta^{\text{stock}}$ | Stock-balance slack (penalized in the objective) |

## Objective function

$$VTC = C^{\text{CAPEX}} + C^{\text{Fixed OPEX}} + C^{\text{Variable OPEX}} + C^{\text{Emission tax}} \to \min$$

Capital costs ($C^{\text{CAPEX}}$) cover new installation, retrofit, and replacement, each annualized with technology-specific discount rates and lifetimes. Fixed O&M is proportional to installed capacity; variable O&M is proportional to dispatch. Emission taxes apply per unit of emission.

## Central constraint

The **nodal power balance** ties all components together. For each node-sector pair $(n,i)$ and timeslice group $m_t \in MT$, total generation, storage discharge, and net link imports must equal exogenous demand $d_{n,i,MT}$:

$$\sum_r \sum_{t \in MT} VX_{n,i,r,t} + \sum_s \sum_{t \in MT} VX_{n,i,s,t} + \text{(net link flows)} = d_{n,i,MT} + \delta_{n,i,MT}$$

See the [Equations](equation.md) page for the complete mathematical specification of all constraints.
