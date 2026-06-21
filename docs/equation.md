---
icon: material/draw-pen
---

# Equations

!!! note "PSMpy formulation — equivalent to the GAMS PowerSystemModel"
    This is PSMpy's mathematical formulation. By design the **math is kept
    equivalent to the reference GAMS `PowerSystemModel`**, and the symbol names /
    domains stay 1:1 with GAMS — so these equations and the GAMS symbol
    correspondence apply to both. The few deliberate numerical deviations
    (carry-over rounding, demand-profile normalization, exhaustible-resource cap
    handling, …) are registered in the model repo's `docs/spec/deviations.md`.

!!! note "Notation conventions"
    Here, capital characters refer to endogenous variables, while lowercase characters indicate exogenous parameters.

??? tip "GAMS symbol naming conventions"
    The model follows systematic naming rules for GAMS symbols to ensure consistency and readability:
    
    **Decision variables (Documentation ↔ GAMS correspondence):**
    
    - All decision variables start with `V` in both documentation and GAMS implementation
    - Documentation uses generic notation $VX$, $VS$, $VR$, $VC$, and $VP$ with technology-agnostic subscripts
    - GAMS uses technology-specific prefixes where the second letter indicates technology type:
        - `VG*` for generators: $VX_{n,i,r,t}$ → `VG(N,I,R,T)`, $VS_{n,i,r}$ → `VGS(N,I,R)`, $VR_{n,i,r}$ → `VGR(N,I,R)`, $VC_{n,i,r_0,r_1,y}$ → `VGC(N,I,R0,R1,Y)`, $VP_{n,i,r}$ → `VGP(N,I,R)`
        - `VF*` for links: $VX^+_{l,t}$ → `VFF(L,T)`, $VX^-_{l,t}$ → `VFB(L,T)`, $VS_l$ → `VFS(L)`, $VR_l$ → `VFR(L)`, $VC_{l_0,l_1,y}$ → `VFC(L0,L1,Y)`, $VP_l$ → `VFP(L)`
        - `VE*` for storage (energy): $VE_{n,i,s,t}$ → `VE(N,I,S,T)`, $VS_{n,i,s}$ → `VES(N,I,S)`, $VR_{n,i,s}$ → `VER(N,I,S)`, $VC_{n,i,s_0,s_1,y}$ → `VEC(N,I,S0,S1,Y)`, $VP_{n,i,s}$ → `VEP(N,I,S)`
        - `VH*` for storage (power): $VX_{n,i,s,t}$ → `VH(N,I,S,T)`
        - `VP*` for energy supply: $VP_{n,i,k}$ → `VP(N,I,K)`
        - `VQ*` for emissions: $VQ_{n,i,g}$ → `VQ(N,I,G)`
    - Third letter in GAMS indicates variable type: `S` (stock/capacity), `R` (new installation/renewal), `C` (retrofit), `P` (replacement), `X` (operation/dispatch), `F`/`B` (forward/backward flow)
    - Special case: total cost uses $VTC$ in the documentation and `VTC` in GAMS
    
    **Parameters:**
    
    - Technology-specific parameters start with technology letter: `g` (generator), `f` (link), `e` (storage)
    - Cost parameters: `cc` (capital cost), `foc` (fixed O&M cost), `voc` (variable O&M cost), `rc` (retrofit cost), `pc` (replacement cost), `acc` (annualized capital cost), `arc` (annualized retrofit cost), `apc` (annualized replacement cost)
    - Technical parameters: `mn` (minimum), `mx` (maximum), `fx` (fixed), `lt` (lifetime), `alp` (alpha/discount rate), `eta` (efficiency)
    - Stock parameters: `sc` (stock by vintage), `ssc` (surviving stock), `sr` (replaceable stock), `ssr` (surviving replaceable stock)
    - Examples: `gacc` = generator + annualized capital cost, `gmx` = generator + maximum, `eeta` = storage (energy) + efficiency (self-discharge rate)
    
    **Sets and mappings:**
    
    - Aggregation sets start with `M` followed by base set letter: `MR` (generator group), `ME` (region-sector pair), `MK` (energy type group), etc.
    - Mapping sets start with `M_` followed by set names: `M_MR` (mapping from MR to R), `M_ME` (mapping from ME to N,I)
    - Flag sets start with `FL_`: `FL_IR` (valid region-sector-technology combinations), `FL_IRT` (valid region-sector-technology-timeslice combinations)

## Sets and Indices

The model uses base indices for different dimensions, and aggregation sets that group elements from these base indices.

### Base indices

| Notation  | Symbol | Description                                                                                  |
| --------- | ------ | -------------------------------------------------------------------------------------------- |
| $n \in N$ | `N`    | region                                                                                       |
| $i \in I$ | `I`    | sector                                                                                       |
| $r \in R$ | `R`    | generator technology                                                                         |
| $l \in L$ | `L`    | link                                                                                         |
| $s \in S$ | `S`    | storage technology                                                                           |
| $t \in T$ | `T`    | timeslice                                                                                    |
| $k \in K$ | `K`    | energy type                                                                                  |
| $g \in G$ | `G`    | emission type                                                                                |
| $y \in Y$ | `Y`    | year (all years including past and future); simulation period is defined by subset `YEAR(Y)` |

### Aggregation and mapping

Aggregation sets define grouped constraints for applying restrictions at the aggregated level. The table below summarizes aggregation sets, their group indices, and GAMS implementation:

| Group Notation | Description                                                  | GAMS Set | GAMS Mapping       |
| -------------- | ------------------------------------------------------------ | -------- | ------------------ |
| $m_r \in MR$   | Generator technology groups ($MR \subseteq R$)               | `MR`     | `M_MR(MR,R)`       |
| $m_l \in ML$   | Link groups per node-sector                                  | `ML`     | `M_IMLL(N,I,ML,L)` |
| $m_e \in ME$   | Node-sector groups ($ME \subseteq N \times I$)               | `ME`     | `M_ME(ME,N,I)`     |
| $k_m \in MK$   | Energy type groups ($MK \subseteq K$)                        | `MK`     | `M_MK(MK,K)`       |
| $m_q \in MQ$   | Node-sector groups for emissions ($MQ \subseteq N \times I$) | `MQ`     | `M_MQ(MQ,N,I)`     |
| $g_m \in MG$   | Emission gas groups ($MG \subseteq G$)                       | `MG`     | `M_MG(MG,G)`       |
| $m_t \in MT$   | Timeslice groups ($MT \subseteq T$)                          | `MT`     | `M_MT(MT,T)`       |
| $m_x \in MX$   | Node-sector groups for share constraints ($MX \subseteq N \times I$) | `MX`     | `M_MX(MX,N,I)`     |

In equations, group membership is denoted using mapping sets as subscripts. For example, $\sum_{(n,i) \in ME_{m_e}}$ sums over all node-sector pairs $(n,i)$ that belong to energy group $m_e$, and $\sum_{k \in MK_{k_m}}$ sums over all energy types $k$ that belong to energy type group $k_m$.

Retrofit feasibility is represented by dedicated mapping sets:

| Retrofit Mapping | Description                                              | GAMS Mapping  |
| ---------------- | -------------------------------------------------------- | ------------- |
| $RC_{r_0,r_1}$   | Feasible generator retrofit relation from $r_0$ to $r_1$ | `M_RC(R0,R1)` |
| $LC_{l_0,l_1}$   | Feasible link retrofit relation from $l_0$ to $l_1$      | `M_LC(L0,L1)` |
| $SC_{s_0,s_1}$   | Feasible storage retrofit relation from $s_0$ to $s_1$   | `M_SC(S0,S1)` |

In the equations below, these relations are used as conditions in indexed sums; for example, $\sum_{r_0:\,RC_{r_0,r}}$ means summing over all origin technologies $r_0$ that can be retrofitted into destination technology $r$.

### Timeslice parameters

Throughout this document, the following timeslice-related parameters are used:

| Notation   | Symbol  | Description                                                                        | Unit  |
| ---------- | ------- | ---------------------------------------------------------------------------------- | ----- |
| $\omega$   | `wt(T)` | Timeslice weight (hours in each timeslice)                                         | hours |
| $\Delta t$ | `rt(T)` | Timeslice granularity (hours per timeslice, used for ramping and storage dynamics) | hours |
| $\Delta y$ | `t_int` | Time interval between simulation years                                             | years |

!!! note "Note"

    The two parameters serve fundamentally different roles: $\omega$ **integrates** dispatch (GW) over a timeslice to obtain energy (GWh), as in $\sum_t \omega \cdot VX$; $\Delta t$ represents a **rate or duration** used in differential expressions such as ramp limits ($\rho \cdot \Delta t_{t_0} \cdot VS$, units GW) or self-discharge exponents ($\eta^{\Delta t}$, dimensionless).

    $\Delta t$ is always timeslice-specific: the ramping constraints use $\Delta t_{t_0} = rt(T0)$, the duration of the preceding timeslice; the storage energy balance uses $\Delta t_{t_0}$ for the self-discharge exponent and $\Delta t_{t_1} = rt(T1)$ for the dispatch duration. In the GAMS implementation, both `wt(T)` and `rt(T)` are indexed by timeslice $t$.

## Objective function

### Total system cost

The objective function minimizes total system cost ${VTC}$, expressed as the sum of CAPEX ${C^{\text{CAPEX}}}$, Fixed OPEX ${C^{\text{Fixed OPEX}}}$, Variable OPEX ${C^{\text{Variable OPEX}}}$, and Emission tax ${C^{\text{Emission tax}}}$ across all technologies, regions, and sectors.

$$
{VTC} = {C^{\text{CAPEX}}} + {C^{\text{Fixed OPEX}}} + {C^{\text{Variable OPEX}}} + {C^{\text{Emission tax}}} \to \min
$$

Capital expenditure ${C^{\text{CAPEX}}}$ is decomposed into new-installation cost ${C^{\text{CAPEX,new}}}$, retrofit cost ${C^{\text{CAPEX,retrofit}}}$, and replacement cost ${C^{\text{CAPEX,replace}}}$. The first term applies to newly added capacity, the second to technology conversion between vintages or technology classes, and the third to replacement of expired or replaceable stock.

$$
{C^{\text{CAPEX}}} = {C^{\text{CAPEX,new}}} + {C^{\text{CAPEX,retrofit}}} + {C^{\text{CAPEX,replace}}}
$$

$$
{C^{\text{CAPEX,new}}} = \sum_{n,i,r}{c^{\text{new}}}_{n,i,r}\cdot {VR}_{n,i,r} + \sum_{l}{c^{\text{new}}}_{l}\cdot {VR}_{l} + \sum_{n,i,s}{c^{\text{new}}}_{n,i,s}\cdot {VR}_{n,i,s}
$$

$$
{C^{\text{CAPEX,retrofit}}} =
\sum_{n,i,r_0,r_1,y}{c^{\text{retrofit}}}_{n,i,r_0,r_1,y}\cdot {VC}_{n,i,r_0,r_1,y}
+ \sum_{l_0,l_1,y}{c^{\text{retrofit}}}_{l_0,l_1,y}\cdot {VC}_{l_0,l_1,y}
+ \sum_{n,i,s_0,s_1,y}{c^{\text{retrofit}}}_{n,i,s_0,s_1,y}\cdot {VC}_{n,i,s_0,s_1,y}
$$

$$
{C^{\text{CAPEX,replace}}} = \sum_{n,i,r}{c^{\text{replace}}}_{n,i,r}\cdot {VP}_{n,i,r} + \sum_{l}{c^{\text{replace}}}_{l}\cdot {VP}_{l} + \sum_{n,i,s}{c^{\text{replace}}}_{n,i,s}\cdot {VP}_{n,i,s}
$$

Fixed operating expenditure ${C^{\text{Fixed OPEX}}}$ is applied to installed stock variables. In the current implementation, stock-balance slack variables $\delta^{\text{stock}}$ are penalized with the same fixed-cost coefficients so that unresolved stock inconsistencies are reflected in the objective value.

$$
{C^{\text{Fixed OPEX}}} =
\sum_{n,i,r}{o^{\text{fix}}}_{n,i,r}\cdot \left({VS}_{n,i,r}+{\delta^{\text{stock}}}_{n,i,r}\right)
+ \sum_{l}{o^{\text{fix}}}_{l}\cdot \left({VS}_{l}+{\delta^{\text{stock}}}_{l}\right)
+ \sum_{n,i,s}{o^{\text{fix}}}_{n,i,s}\cdot \left({VS}_{n,i,s}+{\delta^{\text{stock}}}_{n,i,s}\right)
$$

Variable operating expenditure ${C^{\text{Variable OPEX}}}$ is incurred only for operational variables, namely generator dispatch ${VX}_{n,i,r,t}$ and link flow ${VX}^{+}_{l,t}, {VX}^{-}_{l,t}$. Storage charge and discharge power ${VX}_{n,i,s,t}$ do not appear directly in this term in the current formulation.

$$
{C^{\text{Variable OPEX}}} = \sum_{n,i,r,t}{\omega}\cdot {o^{\text{variable}}}_{n,i,r}\cdot {VX}_{n,i,r,t}
+ \sum_{l,t}{\omega}\cdot {o^{\text{variable}}}_{l}\cdot \left({VX}^{+}_{l,t}+{VX}^{-}_{l,t}\right)
$$

Emission tax ${C^{\text{Emission tax}}}$ is incurred when an emission tax rate $q^{\text{tax}}_{m_q,m_g}$ is specified for emission groups. For each node-sector-gas combination $(n,i,g)$, the tax is summed over all emission groups $(m_q, m_g)$ to which $(n,i)$ and $g$ belong:

$$
{C^{\text{Emission tax}}} = \sum_{n,i,g} {VQ}_{n,i,g} \cdot \sum_{\substack{m_q:\,(n,i)\in MQ_{m_q} \\ m_g:\,g\in MG_{m_g}}} q^{\text{tax}}_{m_q,m_g}
$$

<details markdown="1">
<summary><b>Parameters used in Total system cost</b></summary>

| Parameter                             | Symbol              | Description                                | Unit            |
| ------------------------------------- | ------------------- | ------------------------------------------ | --------------- |
| $c^{\text{new}}_{n,i,r}$              | `gacc(N,I,R)`       | Annualized capital cost - new installation | million USD/GW  |
| $c^{\text{retrofit}}_{n,i,r_0,r_1,y}$ | `garc(N,I,R0,R1,Y)` | Annualized retrofit cost                   | million USD/GW  |
| $o^{\text{fix}}_{n,i,r}$              | `gfoc(N,I,R)`       | Fixed O&M cost                             | million USD/GW  |
| $c^{\text{replace}}_{n,i,r}$          | `gapc(N,I,R)`       | Annualized capital cost - replacement      | million USD/GW  |
| $o^{\text{variable}}_{n,i,r}$         | `gvoc(N,I,R)`       | Variable O&M cost                          | million USD/GWh |
| $c^{\text{new}}_{l}$                  | `facc(L)`           | Annualized capital cost - new installation | million USD/GW  |
| $c^{\text{retrofit}}_{l_0,l_1,y}$     | `farc(L0,L1,Y)`     | Annualized retrofit cost                   | million USD/GW  |
| $o^{\text{fix}}_{l}$                  | `ffoc(L)`           | Fixed O&M cost                             | million USD/GW  |
| $c^{\text{replace}}_{l}$              | `fapc(L)`           | Annualized capital cost - replacement      | million USD/GW  |
| $o^{\text{variable}}_{l}$             | `fvoc(L)`           | Variable O&M cost                          | million USD/GWh |
| $q^{\text{tax}}_{m_q,m_g}$            | `qtax(MQ,MG)`       | Emission tax rate                          | million USD/MtCO₂ |
| $c^{\text{new}}_{n,i,s}$              | `eacc(N,I,S)`       | Annualized capital cost - new installation | million USD/GWh |
| $c^{\text{retrofit}}_{n,i,s_0,s_1,y}$ | `earc(N,I,S0,S1,Y)` | Annualized retrofit cost                   | million USD/GWh |
| $o^{\text{fix}}_{n,i,s}$              | `efoc(N,I,S)`       | Fixed O&M cost                             | million USD/GWh |
| $c^{\text{replace}}_{n,i,s}$          | `eapc(N,I,S)`       | Annualized capital cost - replacement      | million USD/GWh |
| $\omega$                              | `wt(T)`             | Timeslice weight (hours in each timeslice) | hours           |

</details>

<details markdown="1">
<summary><b>Decision variables used in Total system cost</b></summary>

| Variable                        | Symbol             | Description                                  | Unit        |
| ------------------------------- | ------------------ | -------------------------------------------- | ----------- |
| $VTC$                           | `VTC`              | Total system cost to be minimized            | million USD |
| $VR_{n,i,r}$                    | `VGR(N,I,R)`       | New installation of generator                | GW/yr          |
| $VS_{n,i,r}$                    | `VGS(N,I,R)`       | Installed capacity of generator              | GW          |
| $VC_{n,i,r_0,r_1,y}$            | `VGC(N,I,R0,R1,Y)` | Retrofit flow between generator technologies | GW          |
| $VP_{n,i,r}$                    | `VGP(N,I,R)`       | Replacement capacity of generator            | GW/yr          |
| $VX_{n,i,r,t}$                  | `VG(N,I,R,T)`      | Power output of generator                    | GW          |
| $VR_{l}$                        | `VFR(L)`           | New installation of link                     | GW/yr          |
| $VS_{l}$                        | `VFS(L)`           | Installed capacity of link                   | GW          |
| $VC_{l_0,l_1,y}$                | `VFC(L0,L1,Y)`     | Retrofit flow between links                  | GW          |
| $VP_{l}$                        | `VFP(L)`           | Replacement capacity of link                 | GW/yr          |
| $VX^{+}_{l,t}$                  | `VFF(L,T)`         | Forward energy flow via link                 | GW          |
| $VX^{-}_{l,t}$                  | `VFB(L,T)`         | Backward energy flow via link                | GW          |
| $VR_{n,i,s}$                    | `VER(N,I,S)`       | New installation of storage                  | GWh/yr         |
| $VS_{n,i,s}$                    | `VES(N,I,S)`       | Installed energy capacity of storage         | GWh         |
| $VC_{n,i,s_0,s_1,y}$            | `VEC(N,I,S0,S1,Y)` | Retrofit flow between storage technologies   | GWh         |
| $VP_{n,i,s}$                    | `VEP(N,I,S)`       | Replacement capacity of storage              | GWh/yr         |
| $\delta^{\text{stock}}_{n,i,r}$ | `RES_GSCB(N,I,R)`  | Slack variable for generator stock balance   | GW          |
| $\delta^{\text{stock}}_{l}$     | `RES_FSCB(L)`      | Slack variable for link stock balance        | GW          |
| $\delta^{\text{stock}}_{n,i,s}$ | `RES_ESCB(N,I,S)`  | Slack variable for storage stock balance     | GWh         |
| $VQ_{n,i,g}$                    | `VQ(N,I,G)`        | Annual emissions of type $g$                 | MtCO₂       |

</details>

### Annualization of capital costs

New installation, retrofit, and replacement costs are annualized based on technology discount rate and lifetime. The equations below show the generator case as a representative example.

!!! note "Role of $\Delta y$ in annualized costs"
    The $\Delta y$ factor is a rate-to-stock units conversion, not extra discounting. The **new-installation** and **replacement** variables ${VR}, {VP}$ are annual rates (GW/year) that enter the stock multiplied by $\Delta y$ (see the stock-balance equations), so pairing them with ${c}\cdot\Delta y$ charges exactly one year's annuity per GW added. The **retrofit** variable ${VC}$ is already a stock (GW) that enters the cohort directly, so its annualized cost ${c^{\text{retrofit}}}$ carries **no** $\Delta y$ factor — one year's annuity per retrofitted GW.

For generators, annualized costs ${c^{\text{new}}}_{n,i,r}$ and ${c^{\text{replace}}}_{n,i,r}$ are computed from initial capital costs ${b^{\text{new}}}_{n,i,r}$ and ${b^{\text{replace}}}_{n,i,r}$ using discount rate $\alpha_{n, i, r}$ and lifetime $\tau_{n, i, r}$:

$$
{c^{\text{new}}}_{n, i, r} = {b^{\text{new}}}_{n, i, r} \cdot \frac{\alpha_{n, i, r} \cdot \left(1 + \alpha_{n, i, r}\right)^{\tau_{n, i, r}}}
{\left(1 + \alpha_{n, i, r}\right)^{\tau_{n, i, r}}-1} \cdot \Delta y
$$

$$
{c^{\text{replace}}}_{n, i, r} = {b^{\text{replace}}}_{n, i, r} \cdot \frac{\alpha_{n, i, r} \cdot \left(1 + \alpha_{n, i, r}\right)^{\tau_{n, i, r}}}
{\left(1 + \alpha_{n, i, r}\right)^{\tau_{n, i, r}}-1} \cdot \Delta y
$$

For retrofit, annualized retrofit cost ${c^{\text{retrofit}}}_{n,i,r_0,r_1,y}$ uses retrofit cost coefficient ${b^{\text{retrofit}}}_{n,i,r_0,r_1}$ together with destination discount rate $\alpha_{n,i,r_1}$ and retrofit repayment period ${\tau^{\text{retrofit}}}_{n,i,r_0,y}$. In the current implementation, repayment period ${\tau^{\text{retrofit}}}_{n,i,r_0,y}$ is the remaining lifetime at vintage year $y$, bounded below by 1 year, and $y^*$ denotes the simulation year:

$$
{\tau^{\text{retrofit}}}_{n,i,r_0,y}
= \max \left(1,\ \tau_{n,i,r_0} - \left(y^* - y\right)\right)
$$

For generators:

$$
{c^{\text{retrofit}}}_{n, i, r_0, r_1, y}
= {b^{\text{retrofit}}}_{n, i, r_0, r_1}
\cdot
\frac{\alpha_{n, i, r_1}\cdot \left(1+\alpha_{n, i, r_1}\right)^{{\tau^{\text{retrofit}}}_{n,i,r_0,y}}}
{\left(1+\alpha_{n, i, r_1}\right)^{{\tau^{\text{retrofit}}}_{n,i,r_0,y}}-1}
$$

<details markdown="1">
<summary><b>Parameters used in Annualization of capital costs</b></summary>

| Parameter                              | Symbol              | Description                                                                | Unit           |
| -------------------------------------- | ------------------- | -------------------------------------------------------------------------- | -------------- |
| $b^{\text{new}}_{n,i,r}$               | `gcc(N,I,R)`        | Initial capital cost for generator new installation (before annualization) | million USD/GW |
| $b^{\text{retrofit}}_{n,i,r_0,r_1}$    | `grc(N,I,R0,R1)`    | Initial retrofit cost for generator transition                             | million USD/GW |
| $b^{\text{replace}}_{n,i,r}$           | `gpc(N,I,R)`        | Initial capital cost for generator replacement (before annualization)      | million USD/GW |
| $c^{\text{new}}_{n,i,r}$               | `gacc(N,I,R)`       | Annualized capital cost for generator new installation                     | million USD/GW |
| $c^{\text{retrofit}}_{n,i,r_0,r_1,y}$  | `garc(N,I,R0,R1,Y)` | Annualized retrofit cost of generator                                      | million USD/GW |
| $c^{\text{replace}}_{n,i,r}$           | `gapc(N,I,R)`       | Annualized replacement cost of generator                                   | million USD/GW |
| $\alpha_{n,i,r}$                       | `galp(N,I,R)`       | Discount rate for generator                                                | -              |
| $\tau_{n,i,r}$                         | `glt(N,I,R)`        | Lifetime of generator                                                      | years          |
| ${\tau^{\text{retrofit}}}_{n,i,r_0,y}$ | `grt(N,I,R0,Y)`     | Retrofit repayment period used for annualization                           | years          |
| $\Delta y$                             | `t_int`             | Time interval between simulation years                                     | years          |

</details>

## Generator operation & capacity

Generator constraints govern dispatch $VX_{n,i,r,t}$ and new investment $VR_{n,i,r}$. Dispatch enters the nodal power balance and energy supply equations; installed capacity $VS_{n,i,r}$ is determined by the technology stock equations.

### Minimum/Maximum operation of generator

The output of each generator is bounded by its installed capacity and technology-specific dispatch limits.
The coefficients ${\gamma^{\min}}_{n,i,r,t}$ and ${\gamma^{\max}}_{n,i,r,t}$ scale installed capacity into lower and upper dispatch bounds for each timeslice.

$$
{{\gamma^{\min}}}_{n, i, r, t} \cdot {VS}_{n, i, r} \leq {VX}_{n, i, r, t} \leq {{\gamma^{\max}}}_{n, i, r, t} \cdot {VS}_{n, i, r}
$$

<details markdown="1">
<summary><b>Parameters used in Minimum/Maximum operation</b></summary>

| Parameter                 | Symbol         | Description                          | Unit |
| ------------------------- | -------------- | ------------------------------------ | ---- |
| $\gamma^{\min}_{n,i,r,t}$ | `gmn(N,I,R,T)` | Minimum output factor of generator   | 0-1  |
| $\gamma^{\max}_{n,i,r,t}$ | `gmx(N,I,R,T)` | Maximum output factor (availability) | 0-1  |

</details>

<details markdown="1">
<summary><b>Decision variables used in Minimum/Maximum operation</b></summary>

| Variable       | Symbol        | Description                     | Unit |
| -------------- | ------------- | ------------------------------- | ---- |
| $VX_{n,i,r,t}$ | `VG(N,I,R,T)` | Power output of generator       | GW   |
| $VS_{n,i,r}$   | `VGS(N,I,R)`  | Installed capacity of generator | GW   |

</details>

!!! tip "Handling of curtailment"
    For non-dispatchable generators such as wind and solar, ${VX}_{n,i,r,t}$ represents actual generation, which may be less than potential generation ${{\gamma^{\max}}}_{n,i,r,t} \cdot {VS}_{n,i,r}$ due to economic curtailment. The curtailed capacity is calculated in postprocessing as ${{\gamma^{\max}}}_{n,i,r,t} \cdot {VS}_{n,i,r} - {VX}_{n,i,r,t}$.

### Ramping constraints (optional)

Ramp-up and ramp-down constraints limit the rate of change of generator output between consecutive timeslices $t_0$ (preceding) and $t_1$ (current).
These constraints are written on differences between consecutive dispatch levels and therefore couple generator operation across time.
The ramp limit is scaled by the duration $\Delta t_{t_0}$ and the availability $\gamma^{\max}_{n,i,r,t_0}$ of the *preceding* timeslice.

$$
 -{{\Delta t_{t_0}}}\cdot {{\rho^{\text{down}}}}_{n,i,r}\cdot {{\gamma^{\max}}}_{n,i,r,t_0}\cdot {VS}_{n,i,r}
\leq {VX}_{n,i,r,t_1}-{VX}_{n,i,r,t_0}
\leq {{\Delta t_{t_0}}}\cdot {{\rho^{\text{up}}}}_{n,i,r}\cdot {{\gamma^{\max}}}_{n,i,r,t_0}\cdot {VS}_{n,i,r}
$$

<details markdown="1">
<summary><b>Parameters used in Ramping constraints</b></summary>

| Parameter                    | Symbol         | Description                                          | Unit  |
| ---------------------------- | -------------- | ---------------------------------------------------- | ----- |
| $\Delta t_{t_0}$             | `rt(T0)`       | Duration of the preceding timeslice $t_0$ (hours)   | hours |
| $\rho^{\text{up}}_{n,i,r}$   | `gru(N,I,R)`   | Ramp-up rate coefficient                             | -     |
| $\rho^{\text{down}}_{n,i,r}$ | `grd(N,I,R)`   | Ramp-down rate coefficient                           | -     |
| $\gamma^{\max}_{n,i,r,t_0}$  | `gmx(N,I,R,T0)` | Maximum output factor at preceding timeslice $t_0$  | 0-1   |

</details>

<details markdown="1">
<summary><b>Decision variables used in Ramping constraints</b></summary>

| Variable       | Symbol        | Description                     | Unit |
| -------------- | ------------- | ------------------------------- | ---- |
| $VX_{n,i,r,t}$ | `VG(N,I,R,T)` | Power output of generator       | GW   |
| $VS_{n,i,r}$   | `VGS(N,I,R)`  | Installed capacity of generator | GW   |

</details>

### Generation share constraints

The model can impose minimum/maximum generation shares between generator groups. The left-hand generator group $m_{r0}$ is compared against the right-hand group $m_{r1}$ via share coefficients $\theta^{\min}$ and $\theta^{\max}$: $m_{r0}$'s aggregated dispatch must lie between $\theta^{\min}$ and $\theta^{\max}$ times $m_{r1}$'s dispatch, summed over node-sector group $m_x$.

!!! tip "Example"
    To require that solar generation accounts for at least 30% of total renewable generation within a node-sector group, set $m_{r0}$ = solar technologies, $m_{r1}$ = all renewable technologies, and $\theta^{\min}_{m_x, m_{r0}, m_{r1}} = 0.3$.

$$
{{\theta^{\min}}}_{m_x,m_{r0},m_{r1}}\cdot \sum_{(n,i)\in MX_{m_x}}\sum_{r\in MR_{m_{r1}}}\sum_{t}{VX}_{n,i,r,t}
\leq \sum_{(n,i)\in MX_{m_x}}\sum_{r\in MR_{m_{r0}}}\sum_{t}{VX}_{n,i,r,t}
\leq {{\theta^{\max}}}_{m_x,m_{r0},m_{r1}}\cdot \sum_{(n,i)\in MX_{m_x}}\sum_{r\in MR_{m_{r1}}}\sum_{t}{VX}_{n,i,r,t}
$$

A fixed generation share constraint is also available:

$$
{{\theta^{\text{fix}}}}_{m_x,m_{r0},m_{r1}}\cdot \sum_{(n,i)\in MX_{m_x}}\sum_{r\in MR_{m_{r1}}}\sum_{t}{VX}_{n,i,r,t}
= \sum_{(n,i)\in MX_{m_x}}\sum_{r\in MR_{m_{r0}}}\sum_{t}{VX}_{n,i,r,t}
$$

<details markdown="1">
<summary><b>Parameters used in Generation share constraints</b></summary>

| Parameter                              | Symbol           | Description                          | Unit |
| -------------------------------------- | ---------------- | ------------------------------------ | ---- |
| $\theta^{\min}_{m_x,m_{r0},m_{r1}}$   | `gmmn(MX,MR0,MR1)` | Minimum generation share coefficient | -    |
| $\theta^{\max}_{m_x,m_{r0},m_{r1}}$   | `gmmx(MX,MR0,MR1)` | Maximum generation share coefficient | -    |
| $\theta^{\text{fix}}_{m_x,m_{r0},m_{r1}}$ | `gmfx(MX,MR0,MR1)` | Fixed generation share coefficient   | -    |

</details>

<details markdown="1">
<summary><b>Decision variables used in Generation share constraints</b></summary>

| Variable       | Symbol        | Description               | Unit |
| -------------- | ------------- | ------------------------- | ---- |
| $VX_{n,i,r,t}$ | `VG(N,I,R,T)` | Power output of generator | GW   |

</details>

### Capacity bounds for generator

Installed capacity ${VS}_{n,i,r}$ of generators is constrained by technology-specific lower and upper bounds ${\lambda^{\min}}_{n,i,r}$ and ${\lambda^{\max}}_{n,i,r}$. These bounds apply directly to stock variable ${VS}_{n,i,r}$, whereas the bounds below apply to new-installation variable ${VR}_{n,i,r}$.

$$
{{\lambda^{\min}}}_{n, i, r} \leq {VS}_{n, i, r} \leq {{\lambda^{\max}}}_{n, i, r}
$$

New installation ${VR}_{n,i,r}$ is also bounded by ${\nu^{\min}}_{n,i,r}$ and ${\nu^{\max}}_{n,i,r}$:

$$
{{\nu^{\min}}}_{n, i, r} \leq {VR}_{n, i, r} \leq {{\nu^{\max}}}_{n, i, r}
$$

<details markdown="1">
<summary><b>Parameters used in Capacity bounds</b></summary>

| Parameter                | Symbol        | Description                       | Unit |
| ------------------------ | ------------- | --------------------------------- | ---- |
| $\lambda^{\min}_{n,i,r}$ | `gsmn(N,I,R)` | Minimum installed capacity        | GW   |
| $\lambda^{\max}_{n,i,r}$ | `gsmx(N,I,R)` | Maximum installed capacity        | GW   |
| $\nu^{\min}_{n,i,r}$     | `grmn(N,I,R)` | Minimum new installation per year | GW   |
| $\nu^{\max}_{n,i,r}$     | `grmx(N,I,R)` | Maximum new installation per year | GW   |

</details>

<details markdown="1">
<summary><b>Decision variables used in Capacity bounds</b></summary>

| Variable     | Symbol       | Description                     | Unit |
| ------------ | ------------ | ------------------------------- | ---- |
| $VS_{n,i,r}$ | `VGS(N,I,R)` | Installed capacity of generator | GW   |
| $VR_{n,i,r}$ | `VGR(N,I,R)` | New installation of generator   | GW/yr   |

</details>


## Link operation & capacity

Links represent energy transfer or conversion between node-sector pairs. Link flows $VX^+_{l,t}$ and $VX^-_{l,t}$ appear in the nodal power balance as net imports or exports; installed capacity $VS_l$ is determined by the technology stock equations.

### Link flow operation

Energy flows ${VX}^{+}_{l,t}$ and ${VX}^{-}_{l,t}$ via each link are bounded by installed capacity ${VS}_{l}$ and dispatch-limit coefficients ${\gamma^{\min}_{+}}_{l,t}$, ${\gamma^{\max}_{+}}_{l,t}$, ${\gamma^{\min}_{-}}_{l,t}$, and ${\gamma^{\max}_{-}}_{l,t}$. Forward and backward flows are treated separately to represent lossy transport[^1], and the two directional flow variables make it possible to represent asymmetric conversion or transport behavior while keeping the formulation linear.

$$
{{\gamma^{\min}_{+}}}_{l, t} \cdot {VS}_{l} \leq {VX}^{+}_{l, t} \leq {{\gamma^{\max}_{+}}}_{l, t} \cdot {VS}_{l}
$$

$$
{{\gamma^{\min}_{-}}}_{l, t} \cdot {VS}_{l} \leq {VX}^{-}_{l, t} \leq {{\gamma^{\max}_{-}}}_{l, t} \cdot {VS}_{l}
$$

Fixed flows can be represented by setting bounds equal:

$$
{VX}^{+}_{l,t} = {{\gamma^{\text{fix}}_{+}}}_{l,t}\cdot {VS}_{l},\qquad {VX}^{-}_{l,t} = {{\gamma^{\text{fix}}_{-}}}_{l,t}\cdot {VS}_{l}
$$

<details markdown="1">
<summary><b>Parameters used in Link flow operation</b></summary>

| Parameter                     | Symbol      | Description                  | Unit |
| ----------------------------- | ----------- | ---------------------------- | ---- |
| $\gamma^{\min}_{+,l,t}$       | `ffmn(L,T)` | Minimum forward flow factor  | 0-1  |
| $\gamma^{\max}_{+,l,t}$       | `ffmx(L,T)` | Maximum forward flow factor  | 0-1  |
| $\gamma^{\min}_{-,l,t}$       | `fbmn(L,T)` | Minimum backward flow factor | 0-1  |
| $\gamma^{\max}_{-,l,t}$       | `fbmx(L,T)` | Maximum backward flow factor | 0-1  |
| $\gamma^{\text{fix}}_{+,l,t}$ | `fffx(L,T)` | Fixed forward flow factor    | 0-1  |
| $\gamma^{\text{fix}}_{-,l,t}$ | `fbfx(L,T)` | Fixed backward flow factor   | 0-1  |

</details>

<details markdown="1">
<summary><b>Decision variables used in Link flow operation</b></summary>

| Variable       | Symbol     | Description                   | Unit |
| -------------- | ---------- | ----------------------------- | ---- |
| $VX^{+}_{l,t}$ | `VFF(L,T)` | Forward energy flow via link  | GW   |
| $VX^{-}_{l,t}$ | `VFB(L,T)` | Backward energy flow via link | GW   |
| $VS_{l}$       | `VFS(L)`   | Installed capacity of link    | GW   |

</details>

!!! note "Bidirectional links and Unidirectional links"
    Links are classified as either **bidirectional links**, such as inverters, or **unidirectional links**, such as electrolysers. For unidirectional links, ${{\gamma^{\max}_{-}}}_{l,t} = 0$, restricting energy flow to one direction only.

### Link capacity bounds

Installed link capacity ${VS}_{l}$ is constrained by lower and upper bounds ${\lambda^{\min}}_{l}$ and ${\lambda^{\max}}_{l}$. As with generators, link stock bounds and link new-installation bounds are imposed separately:

$$
{{\lambda^{\min}}}_{l} \leq {VS}_{l} \leq {{\lambda^{\max}}}_{l}
$$

New installation ${VR}_{l}$ is constrained by ${\nu^{\min}}_{l}$ and ${\nu^{\max}}_{l}$:

$$
{{\nu^{\min}}}_{l} \leq {VR}_{l} \leq {{\nu^{\max}}}_{l}
$$

<details markdown="1">
<summary><b>Parameters used in Link capacity bounds</b></summary>

| Parameter            | Symbol    | Description                       | Unit |
| -------------------- | --------- | --------------------------------- | ---- |
| $\lambda^{\min}_{l}$ | `fsmn(L)` | Minimum link capacity             | GW   |
| $\lambda^{\max}_{l}$ | `fsmx(L)` | Maximum link capacity             | GW   |
| $\nu^{\min}_{l}$     | `frmn(L)` | Minimum new installation per year | GW   |
| $\nu^{\max}_{l}$     | `frmx(L)` | Maximum new installation per year | GW   |

</details>

<details markdown="1">
<summary><b>Decision variables used in Link capacity bounds</b></summary>

| Variable | Symbol   | Description                | Unit |
| -------- | -------- | -------------------------- | ---- |
| $VS_{l}$ | `VFS(L)` | Installed capacity of link | GW   |
| $VR_{l}$ | `VFR(L)` | New installation of link   | GW/yr   |

</details>

### Link contribution share constraints

The model can impose constraints on the contribution of link groups to nodal flows. Share coefficients $\theta^{\min}_{m_x,m_{l0},m_{l1}}$ and $\theta^{\max}_{m_x,m_{l0},m_{l1}}$ compare incidence-weighted contributions of two link groups $m_{l0}$ and $m_{l1}$, aggregated over a node-sector group $m_x \in MX$.

The quantity compared is the **net energy received at each node-sector pair $(n,i)$ via each link group**. Define the per-link received energy at timeslice $t$ as:

$$
\Phi_{n,i,l,t} \;\equiv\;
-{k^{+}}_{n,i,l}\cdot {VX}^{+}_{l,t}
+{k^{-}}_{n,i,l}\cdot {VX}^{-}_{l,t}
-{k^{+}}_{n,i,l,t}\cdot {VX}^{+}_{l,t}
+{k^{-}}_{n,i,l,t}\cdot {VX}^{-}_{l,t}
$$

Positive $\Phi$ indicates net inflow to $(n,i)$; negative indicates net outflow. The static terms $k^{\pm}_{n,i,l}$ capture time-independent topology and efficiency, while $k^{\pm}_{n,i,l,t}$ capture timeslice-dependent conversion.

Using this shorthand, the share constraints are:

$$
\theta^{\min}_{m_x,m_{l0},m_{l1}}\cdot
\sum_{(n,i)\in MX_{m_x}}\sum_{l\in ML_{n,i,m_{l1}}}\sum_{t} \Phi_{n,i,l,t}
\leq
\sum_{(n,i)\in MX_{m_x}}\sum_{l\in ML_{n,i,m_{l0}}}\sum_{t} \Phi_{n,i,l,t}
\leq
\theta^{\max}_{m_x,m_{l0},m_{l1}}\cdot
\sum_{(n,i)\in MX_{m_x}}\sum_{l\in ML_{n,i,m_{l1}}}\sum_{t} \Phi_{n,i,l,t}
$$

A fixed link contribution share constraint is also available:

$$
\sum_{(n,i)\in MX_{m_x}}\sum_{l\in ML_{n,i,m_{l0}}}\sum_{t} \Phi_{n,i,l,t}
= \theta^{\text{fix}}_{m_x,m_{l0},m_{l1}}\cdot
\sum_{(n,i)\in MX_{m_x}}\sum_{l\in ML_{n,i,m_{l1}}}\sum_{t} \Phi_{n,i,l,t}
$$

Incidence matrix coefficients ${k^{+}}_{n,i,l}$ and ${k^{-}}_{n,i,l}$ encode both network topology and link efficiency for forward and backward directions. These coefficients are constructed from link efficiency and topology information.
With this convention, the same link-flow variables can be interpreted consistently in both the nodal balance and link-share constraints.

- **Forward flow incidence matrix**: ${k^{+}}_{n,i,l}$ represents the contribution of forward flow via link $l$ to node-sector pair ${n,i}$
    - Source node (`N0`, `I0`): ${k^{+}}_{n_0,i_0,l} = 1$ (energy leaves the node)
    - Destination node (`N1`, `I1`): ${k^{+}}_{n_1,i_1,l} = -\eta_l$
    - Unconnected nodes: ${k^{+}}_{n,i,l} = 0$

- **Backward flow incidence matrix**: ${k^{-}}_{n,i,l}$ represents the contribution of backward flow via link $l$ to node-sector pair ${n,i}$
    - Source node (`N0`, `I0`): ${k^{-}}_{n_0,i_0,l} = \eta_l$
    - Destination node (`N1`, `I1`): ${k^{-}}_{n_1,i_1,l} = -1$ (energy arrives at the node)
    - Unconnected nodes: ${k^{-}}_{n,i,l} = 0$

<details markdown="1">
<summary><b>Parameters used in Link contribution share constraints</b></summary>

| Parameter                           | Symbol                | Description                                                            | Unit   |
| ----------------------------------- | --------------------- | ---------------------------------------------------------------------- | ------ |
| $ML_{n,i,m_l,l}$                    | `M_IMLL(N,I,ML,L)`    | Mapping from link-technology group to active links at node-sector pair | binary |
| $k^{+}_{n,i,l}$                     | `kff(N,I,L)`          | Incidence matrix for forward flow                                      | -      |
| $k^{-}_{n,i,l}$                     | `kbf(N,I,L)`          | Incidence matrix for backward flow                                     | -      |
| $k^{+}_{n,i,l,t}$                   | `kffv(N,I,L,T)`       | Time-dependent incidence matrix for forward flow                       | -      |
| $k^{-}_{n,i,l,t}$                   | `kbfv(N,I,L,T)`       | Time-dependent incidence matrix for backward flow                      | -      |
| $\theta^{\min}_{m_x,m_{l0},m_{l1}}$          | `fmmn(MX,ML0,ML1)` | Minimum link group share coefficient                                   | -      |
| $\theta^{\max}_{m_x,m_{l0},m_{l1}}$          | `fmmx(MX,ML0,ML1)` | Maximum link group share coefficient                                   | -      |
| $\theta^{\text{fix}}_{m_x,m_{l0},m_{l1}}$    | `fmfx(MX,ML0,ML1)` | Fixed link group share coefficient                                     | -      |

</details>

<details markdown="1">
<summary><b>Decision variables used in Link contribution share constraints</b></summary>

| Variable       | Symbol     | Description                   | Unit |
| -------------- | ---------- | ----------------------------- | ---- |
| $VX^{+}_{l,t}$ | `VFF(L,T)` | Forward energy flow via link  | GW   |
| $VX^{-}_{l,t}$ | `VFB(L,T)` | Backward energy flow via link | GW   |

</details>

## Storage operation & capacity

Storage equations couple operation across timeslices via the state-of-charge ${VE}_{n,i,s,t}$. Storage charge/discharge power $VX_{n,i,s,t}$ enters the nodal power balance; installed energy capacity $VS_{n,i,s}$ is determined by the technology stock equations.

### Minimum/Maximum operation of storage

Storage operation involves charge/discharge power ${VX}_{n,i,s,t}$ and energy level ${VE}_{n,i,s,t}$. Power ${VX}_{n,i,s,t}$ governs instantaneous charging or discharging, while energy level ${VE}_{n,i,s,t}$ tracks the intertemporal state of charge.

$$
-\infty < {VX}_{n, i, s, t} < +\infty
$$

$$
{{\xi^{\min}}}_{n, i, s, t} \cdot {VS}_{n, i, s} \leq {VE}_{n, i, s, t} \leq {{\xi^{\max}}}_{n, i, s, t} \cdot {VS}_{n, i, s}
$$

<details markdown="1">
<summary><b>Parameters used in Minimum/Maximum operation of storage</b></summary>

| Parameter              | Symbol         | Description                 | Unit |
| ---------------------- | -------------- | --------------------------- | ---- |
| $\xi^{\min}_{n,i,s,t}$ | `emn(N,I,S,T)` | Minimum energy level factor | 0-1  |
| $\xi^{\max}_{n,i,s,t}$ | `emx(N,I,S,T)` | Maximum energy level factor | 0-1  |

</details>

<details markdown="1">
<summary><b>Decision variables used in Minimum/Maximum operation of storage</b></summary>

| Variable       | Symbol        | Description                                                                       | Unit |
| -------------- | ------------- | --------------------------------------------------------------------------------- | ---- |
| $VE_{n,i,s,t}$ | `VE(N,I,S,T)` | Energy level (state-of-charge) of storage, constrained by $\xi^{\min}/\xi^{\max}$ | GWh  |
| $VS_{n,i,s}$   | `VES(N,I,S)`  | Installed energy capacity of storage                                              | GWh  |
| $VX_{n,i,s,t}$ | `VH(N,I,S,T)` | Charge/discharge power (unbounded: $-\infty$ to $+\infty$; positive: discharge)   | GW   |

</details>

### Storage capacity bounds

Installed energy capacity ${VS}_{n,i,s}$ of storage is constrained by lower and upper bounds ${\lambda^{\min}}_{n,i,s}$ and ${\lambda^{\max}}_{n,i,s}$. These bounds apply to energy-capacity stock ${VS}_{n,i,s}$ rather than to charge/discharge power ${VX}_{n,i,s,t}$.

$$
{{\lambda^{\min}}}_{n, i, s} \leq {VS}_{n, i, s} \leq {{\lambda^{\max}}}_{n, i, s}
$$

New installation ${VR}_{n,i,s}$ is constrained by ${\nu^{\min}}_{n,i,s}$ and ${\nu^{\max}}_{n,i,s}$.

$$
{{\nu^{\min}}}_{n, i, s} \leq {VR}_{n, i, s} \leq {{\nu^{\max}}}_{n, i, s}
$$

<details markdown="1">
<summary><b>Parameters used in Storage capacity bounds</b></summary>

| Parameter                | Symbol        | Description                       | Unit |
| ------------------------ | ------------- | --------------------------------- | ---- |
| $\lambda^{\min}_{n,i,s}$ | `esmn(N,I,S)` | Minimum installed energy capacity | GWh  |
| $\lambda^{\max}_{n,i,s}$ | `esmx(N,I,S)` | Maximum installed energy capacity | GWh  |
| $\nu^{\min}_{n,i,s}$     | `ermn(N,I,S)` | Minimum new installation          | GWh  |
| $\nu^{\max}_{n,i,s}$     | `ermx(N,I,S)` | Maximum new installation          | GWh  |

</details>

<details markdown="1">
<summary><b>Decision variables used in Storage capacity bounds</b></summary>

| Variable     | Symbol       | Description                          | Unit |
| ------------ | ------------ | ------------------------------------ | ---- |
| $VS_{n,i,s}$ | `VES(N,I,S)` | Installed energy capacity of storage | GWh  |
| $VR_{n,i,s}$ | `VER(N,I,S)` | New installation of storage          | GWh/yr  |

</details>

### Time-series state of charge of storage

Energy level ${VE}_{n,i,s,t_1}$ at timeslice $t_1$ depends on the energy level at the preceding timeslice $t_0$ (where $FL\_TT(t_0,t_1)$ holds), self-discharge, and charge/discharge power ${VX}_{n,i,s,t_1}$ during $t_1$. This equation is the intertemporal coupling condition for storage and determines how operation in one timeslice affects the feasible state in the next.

$$
{VE}_{n, i, s, t_1} = {{\eta}_{n, i, s}}^{{\Delta t_{t_0}}} \cdot {VE}_{n, i, s, t_0} - {{\Delta t_{t_1}}} \cdot {VX}_{n, i, s, t_1}
$$

where $\Delta t_{t_0} = rt(t_0)$ is the granularity of the preceding timeslice (used for the self-discharge exponent) and $\Delta t_{t_1} = rt(t_1)$ is the granularity of the current timeslice (used for the dispatch duration). When all timeslices share the same duration, this simplifies to ${VE}_{n,i,s,t} = \eta_{n,i,s}^{\Delta t} \cdot {VE}_{n,i,s,t-1} - \Delta t \cdot {VX}_{n,i,s,t}$.

With the sign convention used here, positive ${VX}_{n,i,s,t}$ decreases stored energy, while negative values represent charging and increase the energy level.

<details markdown="1">
<summary><b>Parameters used in Time-series state of charge of storage</b></summary>

| Parameter          | Symbol        | Description                                                           | Unit  |
| ------------------ | ------------- | --------------------------------------------------------------------- | ----- |
| $\eta_{n,i,s}$     | `eeta(N,I,S)` | Self-discharge rate per hour                                          | -     |
| $\Delta t_{t_0}$   | `rt(T0)`      | Timeslice granularity of the preceding timeslice (hours)              | hours |
| $\Delta t_{t_1}$   | `rt(T1)`      | Timeslice granularity of the current timeslice (hours)                | hours |

</details>

<details markdown="1">
<summary><b>Decision variables used in Time-series state of charge of storage</b></summary>

| Variable       | Symbol        | Description                                                                   | Unit |
| -------------- | ------------- | ----------------------------------------------------------------------------- | ---- |
| $VE_{n,i,s,t}$ | `VE(N,I,S,T)` | Energy level (state-of-charge) of storage, bounded by $\xi^{\min}/\xi^{\max}$ | GWh  |
| $VX_{n,i,s,t}$ | `VH(N,I,S,T)` | Charge/discharge power (unbounded; positive: discharge, negative: charging)   | GW   |

</details>

!!! note "Cyclic boundary condition"
    
    This constraint enforces that the energy level at the first timeslice ($t_{\text{first}}$) equals the energy level at the last timeslice ($t_{\text{last}}$), ensuring cyclic operation across the modeled period. This prevents the model from arbitrarily draining or filling storage between simulation years, maintaining inter-annual consistency.

    $$
    {VE}_{n, i, s, t_{\text{first}}} = {VE}_{n, i, s, t_{\text{last}}}
    $$

### Power-Energy capacity of storage

Some storage technologies, such as pumped hydro storage, relate link power capacity ${VS}_{l}$, for example pump or turbine capacity, to storage energy capacity ${VS}_{n,i,s}$, for example reservoir capacity. For each associated link $l$:

$$
{VS}_{l} = \sum_{(n,i,s)\in ISL_l} {{\chi}_{n,i,s,l}}\cdot {VS}_{n,i,s}
$$

where $ISL_l$ is the set of storage technology instances ${n,i,s}$ related to link $l$.

<details markdown="1">
<summary><b>Parameters used in Power-Energy capacity of storage</b></summary>

| Parameter        | Symbol          | Description           | Unit   |
| ---------------- | --------------- | --------------------- | ------ |
| $\chi_{n,i,s,l}$ | `ecrt(N,I,S,L)` | Power-to-energy ratio | GW/GWh |

</details>

<details markdown="1">
<summary><b>Decision variables used in Power-Energy capacity of storage</b></summary>

| Variable     | Symbol       | Description                          | Unit |
| ------------ | ------------ | ------------------------------------ | ---- |
| $VS_{l}$     | `VFS(L)`     | Installed capacity of link           | GW   |
| $VS_{n,i,s}$ | `VES(N,I,S)` | Installed energy capacity of storage | GWh  |

</details>

## Nodal power balance

The nodal power balance is the central constraint that ties together all operation variables: generator output, storage charge/discharge, and link flows must collectively meet exogenous demand at every node and timeslice group. Generator output ${VX}_{n,i,r,t}$, storage power ${VX}_{n,i,s,t}$, and net link contributions are aggregated on the left-hand side and matched to exogenous demand ${d}_{n,i,MT}$ and slack $\delta_{n,i,MT}$ on the right-hand side. Because the slack $\delta_{n,i,MT} \ge 0$ sits on the **demand side** (supply $= d + \delta$), it absorbs **excess supply** — a free-disposal valve, *not* unmet-demand relief. Its upper bound `res_npb_up(N,I,MT)` is $\approx 0$ (`eps`) almost everywhere, pinning supply to demand, and is raised to $+\infty$ only on oversupply-allowed buses (the GAMS `I_OS` hook, e.g. heat dumping).

$$
\sum_{r}\sum_{t\in MT}{VX}_{n,i,r,t}+\sum_{s}\sum_{t\in MT}{VX}_{n,i,s,t}
-\sum_{l}\sum_{t\in MT}{k^{+}}_{n,i,l}{VX}^{+}_{l,t}
+\sum_{l}\sum_{t\in MT}{k^{-}}_{n,i,l}{VX}^{-}_{l,t}
-\sum_{l}\sum_{t\in MT}{k^{+}}_{n,i,l,t}{VX}^{+}_{l,t}
+\sum_{l}\sum_{t\in MT}{k^{-}}_{n,i,l,t}{VX}^{-}_{l,t}
= {d}_{n,i,MT}+\delta_{n,i,MT}
$$

On the left-hand side, $\sum_{r}\sum_{t\in MT}{VX}_{n,i,r,t}$ aggregates generator output, $\sum_{s}\sum_{t\in MT}{VX}_{n,i,s,t}$ aggregates storage discharge and charge power, and the incidence-weighted link terms represent net imports and exports via links. On the right-hand side, supply equals exogenous demand ${d}_{n,i,MT}$ plus the disposal slack $\delta_{n,i,MT}$, so any oversupply is absorbed by $\delta$ where its bound permits.

<details markdown="1">
<summary><b>Parameters used in Nodal power balance</b></summary>

| Parameter                          | Symbol              | Description                                             | Unit |
| ---------------------------------- | ------------------- | ------------------------------------------------------- | ---- |
| ${d}_{n,i,MT}$                     | `dmd(N,I,MT)`       | Exogenous demand for power (timeslice-group aggregated) | GW   |
| $k^{+}_{n,i,l}$                    | `kff(N,I,L)`        | Incidence matrix for forward link flows                 | -    |
| $k^{-}_{n,i,l}$                    | `kbf(N,I,L)`        | Incidence matrix for backward link flows                | -    |
| $k^{+}_{n,i,l,t}$                  | `kffv(N,I,L,T)`     | Time-dependent incidence matrix for forward link flows  | -    |
| $k^{-}_{n,i,l,t}$                  | `kbfv(N,I,L,T)`     | Time-dependent incidence matrix for backward link flows | -    |
| $\delta^{\max}_{n,i,MT}$           | `res_npb_up(N,I,MT)` | Upper bound on nodal power balance slack               | GW   |

</details>

<details markdown="1">
<summary><b>Decision variables used in Nodal power balance</b></summary>

| Variable          | Symbol            | Description                            | Unit |
| ----------------- | ----------------- | -------------------------------------- | ---- |
| $VX_{n,i,r,t}$    | `VG(N,I,R,T)`     | Power output of generator              | GW   |
| $VX_{n,i,s,t}$    | `VH(N,I,S,T)`     | Storage charge/discharge power         | GW   |
| $VX^{+}_{l,t}$    | `VFF(L,T)`        | Forward energy flow via link           | GW   |
| $VX^{-}_{l,t}$    | `VFB(L,T)`        | Backward energy flow via link          | GW   |
| $\delta_{n,i,MT}$ | `RES_NPB(N,I,MT)` | Slack variable for nodal power balance | GW   |

</details>

!!! tip "Timeslice aggregation"
    The constraint aggregates over timeslice group $MT$ rather than individual timeslices $t$, allowing flexibility in temporal resolution. Common groupings:
    
    - $MT$ = {all timeslices}: annual energy balance
    - $MT$ = {single timeslice $t$}: hourly or sub-hourly balances
    
    This design enables the model to enforce balances at different temporal scales depending on the application.

## Technology stock

Technology stock equations determine installed capacity $VS$, which appears in all operational capacity-bound constraints. Stock comprises new investments, surviving historical vintages, and optional retrofit flows.

Installed capacity ${VS}$ comprises new installations ${VR}$, replacements ${VP}$, surviving vintage stock ${sc}$, net retrofit inflows and outflows ${VC}$, and stock-balance slack $\delta^{\text{stock}}$. The surviving vintage stock $sc_{n,i,r,y}$ is an **exogenous parameter** pre-computed from historical installation records and survival functions before the optimization. The factor $\Delta y$ on the investment terms converts annual addition (GW/year) to capacity added per simulation step (GW/step). For generators:

$$
{VS}_{n,i,r}
= ({VR}_{n,i,r}+{VP}_{n,i,r})\cdot \Delta y
+ \sum_{y}\left(
{sc}_{n,i,r,y}
+ \sum_{r_0:\,RC_{r_0,r}} {VC}_{n,i,r_0,r,y}
- \sum_{r_1:\,RC_{r,r_1}} {VC}_{n,i,r,r_1,y}
\right)
- {\delta^{\text{stock}}}_{n,i,r}
$$

Analogously, links and storage satisfy:

$$
{VS}_{l}
= ({VR}_{l}+{VP}_{l})\cdot \Delta y
+ \sum_{y}\left(
{sc}_{l,y}
+ \sum_{l_0:\,LC_{l_0,l}} {VC}_{l_0,l,y}
- \sum_{l_1:\,LC_{l,l_1}} {VC}_{l,l_1,y}
\right)
- {\delta^{\text{stock}}}_{l}
$$

$$
{VS}_{n,i,s}
= ({VR}_{n,i,s}+{VP}_{n,i,s})\cdot \Delta y
+ \sum_{y}\left(
{sc}_{n,i,s,y}
+ \sum_{s_0:\,SC_{s_0,s}} {VC}_{n,i,s_0,s,y}
- \sum_{s_1:\,SC_{s,s_1}} {VC}_{n,i,s,s_1,y}
\right)
- {\delta^{\text{stock}}}_{n,i,s}
$$

Here, $RC_{r_0,r_1}$, $LC_{l_0,l_1}$, and $SC_{s_0,s_1}$ denote feasible retrofit relations from origin technologies to destination technologies, corresponding directly to `M_RC`, `M_LC`, and `M_SC` in GAMS.

Replacement capacity ${VP}_{n,i,r}$ is limited by replaceable stock ${sr}_{n,i,r}$. This restriction prevents replacement variable ${VP}_{n,i,r}$ from exceeding the amount of stock that is eligible for replacement in the modeled year.

$$
{sr}_{n,i,r} \geq {VP}_{n,i,r}
$$

Vintage stocks are tracked by year $y$. For generators, replaceable stock ${sr}_{n,i,r}$ comprises vintages that have reached lifetime $\tau_{n,i,r}$, with age $a_y = y^* - y$ and simulation year $y^*$, scaled to the multi-year step:

$$
{sr}_{n,i,r} = \frac{\sum_{y\;:\; a_y \geq {\tau}_{n,i,r}} {sc}_{n,i,r,y}}{\Delta y}
$$

When technology retrofit is enabled, aggregated retrofit volume $\sum_y {VC}_{n,i,r_0,r_1,y}$ can also be bounded. These bounds act directly on retrofit flow ${VC}_{n,i,r_0,r_1,y}$ between origin technology $r_0$ and destination technology $r_1$. For generators:

$$
{\mu^{\min}}_{n,i,r_0,r_1}
\leq \sum_y {VC}_{n,i,r_0,r_1,y}
\leq {\mu^{\max}}_{n,i,r_0,r_1}
$$

and analogously for links and storage.

<details markdown="1">
<summary><b>Parameters used in Technology stock</b></summary>

| Parameter                  | Symbol            | Description                               | Unit  |
| -------------------------- | ----------------- | ----------------------------------------- | ----- |
| $sc_{n,i,r,y}$             | `gssc(N,I,R,Y)`   | Surviving generator stock by vintage year | GW    |
| $sc_{l,y}$                 | `fssc(L,Y)`       | Surviving link stock by vintage year      | GW    |
| $sc_{n,i,s,y}$             | `essc(N,I,S,Y)`   | Surviving storage stock by vintage year   | GWh   |
| $sr_{n,i,r}$               | `gssr(N,I,R)`     | Replaceable generator stock               | GW    |
| $sr_{l}$                   | `fssr(L)`         | Replaceable link stock                    | GW    |
| $sr_{n,i,s}$               | `essr(N,I,S)`     | Replaceable storage stock                 | GWh   |
| $\mu^{\min}_{n,i,r_0,r_1}$ | `gcmn(N,I,R0,R1)` | Minimum generator retrofit volume         | GW    |
| $\mu^{\max}_{n,i,r_0,r_1}$ | `gcmx(N,I,R0,R1)` | Maximum generator retrofit volume         | GW    |
| $\mu^{\min}_{l_0,l_1}$     | `fcmn(L0,L1)`     | Minimum link retrofit volume              | GW    |
| $\mu^{\max}_{l_0,l_1}$     | `fcmx(L0,L1)`     | Maximum link retrofit volume              | GW    |
| $\mu^{\min}_{n,i,s_0,s_1}$ | `ecmn(N,I,S0,S1)` | Minimum storage retrofit volume           | GWh   |
| $\mu^{\max}_{n,i,s_0,s_1}$ | `ecmx(N,I,S0,S1)` | Maximum storage retrofit volume           | GWh   |
| $\tau_{n,i,r}$             | `glt(N,I,R)`      | Lifetime of generator                     | years |
| $\tau_{l}$                 | `flt(L)`          | Lifetime of link                          | years |
| $\tau_{n,i,s}$             | `elt(N,I,S)`      | Lifetime of storage                       | years |
| $\Delta y$                 | `t_int`           | Time interval between simulation years    | years |

</details>

<details markdown="1">
<summary><b>Decision variables used in Technology stock</b></summary>

| Variable                        | Symbol             | Description                          | Unit |
| ------------------------------- | ------------------ | ------------------------------------ | ---- |
| $VS_{n,i,r}$                    | `VGS(N,I,R)`       | Installed capacity of generator      | GW   |
| $VR_{n,i,r}$                    | `VGR(N,I,R)`       | New installation of generator        | GW/yr   |
| $VC_{n,i,r_0,r_1,y}$            | `VGC(N,I,R0,R1,Y)` | Generator retrofit volume            | GW   |
| $VP_{n,i,r}$                    | `VGP(N,I,R)`       | Replacement capacity of generator    | GW/yr   |
| $\delta^{\text{stock}}_{n,i,r}$ | `RES_GSCB(N,I,R)`  | Generator stock-balance slack        | GW   |
| $VS_{l}$                        | `VFS(L)`           | Installed capacity of link           | GW   |
| $VR_{l}$                        | `VFR(L)`           | New installation of link             | GW/yr   |
| $VC_{l_0,l_1,y}$                | `VFC(L0,L1,Y)`     | Link retrofit volume                 | GW   |
| $VP_{l}$                        | `VFP(L)`           | Replacement capacity of link         | GW/yr   |
| $\delta^{\text{stock}}_{l}$     | `RES_FSCB(L)`      | Link stock-balance slack             | GW   |
| $VS_{n,i,s}$                    | `VES(N,I,S)`       | Installed energy capacity of storage | GWh  |
| $VR_{n,i,s}$                    | `VER(N,I,S)`       | New installation of storage          | GWh/yr  |
| $VC_{n,i,s_0,s_1,y}$            | `VEC(N,I,S0,S1,Y)` | Storage retrofit volume              | GWh  |
| $VP_{n,i,s}$                    | `VEP(N,I,S)`       | Replacement capacity of storage      | GWh/yr  |
| $\delta^{\text{stock}}_{n,i,s}$ | `RES_ESCB(N,I,S)`  | Storage stock-balance slack          | GWh  |

</details>

## Energy supply

Energy supply $VP_{n,i,k}$ is an intermediate accounting variable derived from generator dispatch; it feeds into the emission equations and is constrained by resource bounds.

Annual energy supply ${VP}_{n,i,k}$ is obtained by converting generator dispatch ${VX}_{n,i,r,t}$ into energy type $k$ with coefficient ${\eta}_{n,i,r,k}$ and weighting by timeslice duration.

$$
{VP}_{n, i, k} = \sum_{r,t} {\omega} \cdot {\eta}_{n, i, r, k} \cdot {VX}_{n, i, r, t}
$$

Energy supply ${VP}_{n,i,k}$ is constrained by minimum/maximum bounds or fixed values for aggregated region-sector groups and energy groups. Aggregation sets $ME$ and $MK$ allow these accounting constraints to be imposed at a level coarser than the base node-sector and energy-type indices.

$$
{p^{\min}}_{m_e,k_m} \leq \sum_{(n,i)\in ME_{m_e}}\sum_{k\in MK_{k_m}} {VP}_{n,i,k}
\leq {p^{\max}}_{m_e,k_m}
$$

$$
{p^{\text{fix}}}_{m_e,k_m} = \sum_{(n,i)\in ME_{m_e}}\sum_{k\in MK_{k_m}} {VP}_{n,i,k}
$$

<details markdown="1">
<summary><b>Parameters used in Energy supply</b></summary>

| Parameter                  | Symbol          | Description                                                                  | Unit     |
| -------------------------- | --------------- | ---------------------------------------------------------------------------- | -------- |
| $\eta_{n,i,r,k}$           | `geta(N,I,R,K)` | Energy-carrier output per unit of generator dispatch (e.g. heat rate factor); equals 1 for the primary output carrier, differs from 1 when converting to a secondary carrier (e.g. heat from a CHP plant or hydrogen from an electrolyser) | TWh/GWh  |
| $\omega$                   | `wt(T)`         | Timeslice weight                                                             | hours    |
| $p^{\min}_{m_e,k_m}$       | `pmin(ME,MK)`   | Minimum annual energy supply for aggregated group                            | TWh      |
| $p^{\max}_{m_e,k_m}$       | `pmax(ME,MK)`   | Maximum annual energy supply for aggregated group                            | TWh      |
| $p^{\text{fix}}_{m_e,k_m}$ | `pfix(ME,MK)`   | Fixed annual energy supply for aggregated group                              | TWh      |

</details>

<details markdown="1">
<summary><b>Decision variables used in Energy supply</b></summary>

| Variable       | Symbol        | Description                      | Unit |
| -------------- | ------------- | -------------------------------- | ---- |
| $VP_{n,i,k}$   | `VP(N,I,K)`   | Annual energy supply of type $k$ | TWh  |
| $VX_{n,i,r,t}$ | `VG(N,I,R,T)` | Power output of generator        | GW   |

</details>

## Emission

Emissions $VQ_{n,i,g}$ are derived from energy supply and can be controlled through two complementary mechanisms: a **cap** (hard upper bound, defined here) and/or an **emission tax** (price signal added to the objective function, defined in the Objective function section). Either or both can be active simultaneously.

Emissions are computed using emission factor ${\epsilon}_{n,i,k,g}$ and are therefore represented as a post-conversion accounting quantity derived from annual energy supply rather than directly from dispatch.

$$
{VQ}_{n,i,g} = \sum_{k} {{\epsilon}_{n,i,k,g}}\cdot {VP}_{n,i,k}
$$

Emission caps limit total emissions ${VQ}_{n,i,g}$ for aggregated regions and gas groups. As in the energy-supply constraints, aggregation sets determine the level at which these caps are imposed.

$$
\sum_{(n,i)\in MQ_{m_q}}\sum_{g\in MG_{g_m}} {VQ}_{n,i,g} \leq {q^{\max}}_{m_q,g_m}
$$

<details markdown="1">
<summary><b>Parameters used in Emission</b></summary>

| Parameter            | Symbol         | Description                                     | Unit            |
| -------------------- | -------------- | ----------------------------------------------- | --------------- |
| $\epsilon_{n,i,k,g}$ | `gas(N,I,K,G)` | Emission factor for energy type $k$ and gas $g$ | e.g., MtCO₂/TWh |
| $q^{\max}_{m_q,g_m}$ | `qmax(MQ,MG)`  | Maximum annual emissions for aggregated group   | e.g., MtCO₂     |

</details>

<details markdown="1">
<summary><b>Decision variables used in Emission</b></summary>

| Variable     | Symbol      | Description                      | Unit        |
| ------------ | ----------- | -------------------------------- | ----------- |
| $VQ_{n,i,g}$ | `VQ(N,I,G)` | Annual emissions of type $g$     | e.g., MtCO₂ |
| $VP_{n,i,k}$ | `VP(N,I,K)` | Annual energy supply of type $k$ | TWh         |

</details>

[^1]: [Neumann, F., Hagenmeyer, V. & Brown, T. Assessments of linear power flow and transmission loss approximations in coordinated capacity expansion problems. Appl. Energy 314, 118859 (2022).](https://doi.org/10.1016/j.apenergy.2022.118859)
