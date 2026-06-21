---
icon: material/book-open-outline
---

# Appendix

!!! note "PSMpy v0.1 parameter reference"
    The data values and the **GAMS parameter reference** below apply to PSMpy:
    its compiled symbols are 1:1 with these GAMS symbols.

## Power sector

### Cost and efficiency

<!-- One row per entry; add or remove rows as needed. Cite the source in the last column. -->

| Technology | Capital cost (million USD/GW) | Fixed O&M (million USD/GW/yr) | Variable O&M (million USD/GWh) | Efficiency | Lifetime (yr) | Source |
|------------|-------------------------------|-------------------------------|-------------------------------|------------|---------------|--------|
| | | | | | | |
| | | | | | | |
| | | | | | | |
| | | | | | | |
| | | | | | | |

### Potential

<!-- One row per entry; add or remove rows as needed. Cite the source in the last column. -->

| Technology | Region | Available potential (GW) | Capacity factor | Source |
|------------|--------|--------------------------|-----------------|--------|
| | | | | |
| | | | | |
| | | | | |
| | | | | |
| | | | | |

### Existing capacity

<!-- One row per entry; add or remove rows as needed. Cite the source in the last column. -->

| Technology | Region | Installed capacity (GW) | Source |
|------------|--------|------------------------|--------|
| | | | |
| | | | |
| | | | |
| | | | |
| | | | |

---

## Hydrogen sector

### Cost and efficiency

<!-- One row per entry; add or remove rows as needed. Cite the source in the last column. -->

| Technology | Capital cost (million USD/GW) | Fixed O&M (million USD/GW/yr) | Conversion efficiency | Lifetime (yr) | Source |
|------------|-------------------------------|-------------------------------|----------------------|---------------|--------|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |

### Potential

<!-- One row per entry; add or remove rows as needed. Cite the source in the last column. -->

| Technology | Region | Available potential (GW) | Source |
|------------|--------|--------------------------|--------|
| | | | |
| | | | |
| | | | |
| | | | |
| | | | |

### Existing capacity

<!-- One row per entry; add or remove rows as needed. Cite the source in the last column. -->

| Technology | Region | Installed capacity (GW) | Source |
|------------|--------|------------------------|--------|
| | | | |
| | | | |
| | | | |
| | | | |
| | | | |

---

## GAMS parameter reference

### Generators

| Symbol | GAMS name | Description | Unit |
|--------|-----------|-------------|------|
| $\gamma^{\min}_{n,i,r,t}$ | `gmn(N,I,R,T)` | Minimum dispatch per unit of installed capacity | GW/GW |
| $\gamma^{\max}_{n,i,r,t}$ | `gmx(N,I,R,T)` | Maximum dispatch per unit of installed capacity | GW/GW |
| $\rho^{\text{up}}_{n,i,r}$ | `gru(N,I,R)` | Maximum ramp-up rate per hour | GW/GW/h |
| $\rho^{\text{down}}_{n,i,r}$ | `grd(N,I,R)` | Maximum ramp-down rate per hour | GW/GW/h |
| $\eta_{n,i,r,k}$ | `geta(N,I,R,K)` | Output per unit of dispatch for energy carrier $k$ | TWh/GWh |
| $b^{\text{new}}_{n,i,r}$ | `gcc(N,I,R)` | Overnight capital cost of new installation | million USD/GW |
| $c^{\text{new}}_{n,i,r}$ | `gacc(N,I,R)` | Annualized capital cost of new installation | million USD/GW |
| $o^{\text{fix}}_{n,i,r}$ | `gfoc(N,I,R)` | Fixed O&M cost | million USD/GW/yr |
| $o^{\text{variable}}_{n,i,r}$ | `gvoc(N,I,R)` | Variable O&M cost | million USD/GWh |
| $b^{\text{retrofit}}_{n,i,r_0,r_1}$ | `grc(N,I,R,R)` | Overnight retrofit cost from $r_0$ to $r_1$ | million USD/GW |
| $c^{\text{retrofit}}_{n,i,r_0,r_1,y}$ | `garc(N,I,R,R,Y)` | Annualized retrofit cost | million USD/GW |
| $b^{\text{replace}}_{n,i,r}$ | `gpc(N,I,R)` | Overnight replacement cost | million USD/GW |
| $c^{\text{replace}}_{n,i,r}$ | `gapc(N,I,R)` | Annualized replacement cost | million USD/GW |
| $\alpha_{n,i,r}$ | `galp(N,I,R)` | Discount rate | – |
| $\tau_{n,i,r}$ | `glt(N,I,R)` | Technical lifetime | yr |
| $sc_{n,i,r,y}$ | `gssc(N,I,R,Y)` | Surviving stock installed in year $y$ | GW |
| $sr_{n,i,r}$ | `gssr(N,I,R)` | Replaceable stock | GW |
| $\lambda^{\min}_{n,i,r}$ | `gsmn(N,I,R)` | Lower bound on installed capacity | GW |
| $\lambda^{\max}_{n,i,r}$ | `gsmx(N,I,R)` | Upper bound on installed capacity | GW |
| $\nu^{\min}_{n,i,r}$ | `grmn(N,I,R)` | Lower bound on annual new installation | GW |
| $\nu^{\max}_{n,i,r}$ | `grmx(N,I,R)` | Upper bound on annual new installation | GW |
| $\mu^{\min}_{n,i,r_0,r_1}$ | `gcmn(N,I,R,R)` | Lower bound on cumulative retrofit | GW |
| $\mu^{\max}_{n,i,r_0,r_1}$ | `gcmx(N,I,R,R)` | Upper bound on cumulative retrofit | GW |
| $\theta^{\min}_{m_x,m_{r0},m_{r1}}$ | `gmmn(MX,MR,MR)` | Minimum generation share | – |
| $\theta^{\max}_{m_x,m_{r0},m_{r1}}$ | `gmmx(MX,MR,MR)` | Maximum generation share | – |
| $\theta^{\text{fix}}_{m_x,m_{r0},m_{r1}}$ | `gmfx(MX,MR,MR)` | Fixed generation share | – |

### Links

| Symbol | GAMS name | Description | Unit |
|--------|-----------|-------------|------|
| $\gamma^{\min}_{+,l,t}$ | `ffmn(L,T)` | Minimum forward flow per unit of capacity | GW/GW |
| $\gamma^{\max}_{+,l,t}$ | `ffmx(L,T)` | Maximum forward flow per unit of capacity | GW/GW |
| $\gamma^{\text{fix}}_{+,l,t}$ | `fffx(L,T)` | Fixed forward flow per unit of capacity | GW/GW |
| $\gamma^{\min}_{-,l,t}$ | `fbmn(L,T)` | Minimum backward flow per unit of capacity | GW/GW |
| $\gamma^{\max}_{-,l,t}$ | `fbmx(L,T)` | Maximum backward flow per unit of capacity | GW/GW |
| $\gamma^{\text{fix}}_{-,l,t}$ | `fbfx(L,T)` | Fixed backward flow per unit of capacity | GW/GW |
| $k^{+}_{n,i,l}$ | `kff(N,I,L)` | Forward-flow incidence matrix | – |
| $k^{-}_{n,i,l}$ | `kbf(N,I,L)` | Backward-flow incidence matrix | – |
| $k^{+}_{n,i,l,t}$ | `kffv(N,I,L,T)` | Time-varying forward-flow incidence matrix | – |
| $k^{-}_{n,i,l,t}$ | `kbfv(N,I,L,T)` | Time-varying backward-flow incidence matrix | – |
| $b^{\text{new}}_{l}$ | `fcc(L)` | Overnight capital cost of new installation | million USD/GW |
| $c^{\text{new}}_{l}$ | `facc(L)` | Annualized capital cost of new installation | million USD/GW |
| $o^{\text{fix}}_{l}$ | `ffoc(L)` | Fixed O&M cost | million USD/GW/yr |
| $o^{\text{variable}}_{l}$ | `fvoc(L)` | Variable O&M cost | million USD/GWh |
| $b^{\text{retrofit}}_{l_0,l_1}$ | `frc(L,L)` | Overnight retrofit cost | million USD/GW |
| $c^{\text{retrofit}}_{l_0,l_1,y}$ | `farc(L,L,Y)` | Annualized retrofit cost | million USD/GW |
| $b^{\text{replace}}_{l}$ | `fpc(L)` | Overnight replacement cost | million USD/GW |
| $c^{\text{replace}}_{l}$ | `fapc(L)` | Annualized replacement cost | million USD/GW |
| $\alpha_{l}$ | `falp(L)` | Discount rate | – |
| $\tau_{l}$ | `flt(L)` | Technical lifetime | yr |
| $sc_{l,y}$ | `fssc(L,Y)` | Surviving stock installed in year $y$ | GW |
| $sr_{l}$ | `fssr(L)` | Replaceable stock | GW |
| $\lambda^{\min}_{l}$ | `fsmn(L)` | Lower bound on installed capacity | GW |
| $\lambda^{\max}_{l}$ | `fsmx(L)` | Upper bound on installed capacity | GW |
| $\nu^{\min}_{l}$ | `frmn(L)` | Lower bound on annual new installation | GW |
| $\nu^{\max}_{l}$ | `frmx(L)` | Upper bound on annual new installation | GW |
| $\theta^{\min}_{m_x,m_{l0},m_{l1}}$ | `fmmn(MX,ML,ML)` | Minimum link share | – |
| $\theta^{\max}_{m_x,m_{l0},m_{l1}}$ | `fmmx(MX,ML,ML)` | Maximum link share | – |
| $\theta^{\text{fix}}_{m_x,m_{l0},m_{l1}}$ | `fmfx(MX,ML,ML)` | Fixed link share | – |

### Storage

| Symbol | GAMS name | Description | Unit |
|--------|-----------|-------------|------|
| $\xi^{\min}_{n,i,s,t}$ | `emn(N,I,S,T)` | Minimum state-of-charge per unit of energy capacity | GWh/GWh |
| $\xi^{\max}_{n,i,s,t}$ | `emx(N,I,S,T)` | Maximum state-of-charge per unit of energy capacity | GWh/GWh |
| $\eta_{n,i,s}$ | `eeta(N,I,S)` | Per-hour self-discharge retention rate | GWh/GWh/h |
| $\chi_{n,i,s,l}$ | `ecrt(N,I,S,L)` | Power-to-energy ratio (C-rate) | GW/GWh |
| $b^{\text{new}}_{n,i,s}$ | `ecc(N,I,S)` | Overnight capital cost of new installation | million USD/GWh |
| $c^{\text{new}}_{n,i,s}$ | `eacc(N,I,S)` | Annualized capital cost of new installation | million USD/GWh |
| $o^{\text{fix}}_{n,i,s}$ | `efoc(N,I,S)` | Fixed O&M cost | million USD/GWh/yr |
| $b^{\text{retrofit}}_{n,i,s_0,s_1}$ | `erc(N,I,S,S)` | Overnight retrofit cost | million USD/GWh |
| $c^{\text{retrofit}}_{n,i,s_0,s_1,y}$ | `earc(N,I,S,S,Y)` | Annualized retrofit cost | million USD/GWh |
| $b^{\text{replace}}_{n,i,s}$ | `epc(N,I,S)` | Overnight replacement cost | million USD/GWh |
| $c^{\text{replace}}_{n,i,s}$ | `eapc(N,I,S)` | Annualized replacement cost | million USD/GWh |
| $\alpha_{n,i,s}$ | `ealp(N,I,S)` | Discount rate | – |
| $\tau_{n,i,s}$ | `elt(N,I,S)` | Technical lifetime | yr |
| $sc_{n,i,s,y}$ | `essc(N,I,S,Y)` | Surviving stock installed in year $y$ | GWh |
| $sr_{n,i,s}$ | `essr(N,I,S)` | Replaceable stock | GWh |
| $\lambda^{\min}_{n,i,s}$ | `esmn(N,I,S)` | Lower bound on installed energy capacity | GWh |
| $\lambda^{\max}_{n,i,s}$ | `esmx(N,I,S)` | Upper bound on installed energy capacity | GWh |
| $\nu^{\min}_{n,i,s}$ | `ermn(N,I,S)` | Lower bound on annual new installation | GWh |
| $\nu^{\max}_{n,i,s}$ | `ermx(N,I,S)` | Upper bound on annual new installation | GWh |
| $\mu^{\min}_{n,i,s_0,s_1}$ | `ecmn(N,I,S,S)` | Lower bound on cumulative retrofit | GWh |
| $\mu^{\max}_{n,i,s_0,s_1}$ | `ecmx(N,I,S,S)` | Upper bound on cumulative retrofit | GWh |

### Demand, energy supply, and emissions

| Symbol | GAMS name | Description | Unit |
|--------|-----------|-------------|------|
| $d_{n,i,MT}$ | `dmd(N,I,MT)` | Demand per timeslice group | GW |
| $d^{\text{annual}}_{n,i}$ | `dmda(N,I)` | Annual energy demand | GWh/yr |
| $h_{n,i,MT}$ | `dmdh(N,I,MT)` | Hourly demand profile | GW per GWh/yr |
| $\epsilon_{n,i,k,g}$ | `gas(N,I,K,G)` | Emission factor for carrier $k$ and gas $g$ | MtCO₂/TWh |
| $p^{\min}_{m_e,k_m}$ | `pmin(ME,MK)` | Minimum annual energy supply | TWh |
| $p^{\max}_{m_e,k_m}$ | `pmax(ME,MK)` | Maximum annual energy supply | TWh |
| $p^{\text{fix}}_{m_e,k_m}$ | `pfix(ME,MK)` | Fixed annual energy supply | TWh |
| $q^{\max}_{m_q,g_m}$ | `qmax(MQ,MG)` | Annual emission cap | MtCO₂ |
| $q^{\text{tax}}_{m_q,g_m}$ | `qtax(MQ,MG)` | Emission tax rate | million USD/MtCO₂ |

### Timeslice parameters

| Symbol | GAMS name | Description | Unit |
|--------|-----------|-------------|------|
| $\omega$ | `wt(T)` | Hours represented by each timeslice | h |
| $\Delta t$ | `rt(T)` | Timeslice duration for rate-based constraints | h |
| $\Delta y$ | `t_int` | Simulation step interval | yr |
