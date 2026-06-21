---
icon: material/draw-pen
---

# 式体系

!!! note "PSMpy の定式化 — GAMS PowerSystemModel と等価"
    本ページは PSMpy の数式定式化です。設計方針により**数式は参照 GAMS モデル
    `PowerSystemModel` と等価**に保たれ、記号名・ドメインも GAMS と 1:1 です — した
    がって以下の式と GAMS 記号対応は両者に当てはまります。意図的な数値逸脱（繰越し
    丸め・需要プロファイル正規化・枯渇性資源キャップ処理など）はモデルリポジトリの
    `docs/spec/deviations.md` に登録されています。

    以下で使う GAMS solving ドメインは PSMpy の authoring **要素型**に対応します：
    発電機 `R`/`g*` → **Source**、リンク `L`/`f*` → **Transmission**・**Conversion**、
    蓄電 `S`/`e*` → **Storage**、需要 → **Sink**（ADR-0008）。式は GAMS の `R`/`L`/`S`
    表記を保持します。

!!! note "表記規則"
    本文書では、大文字は内生変数（決定変数）を、小文字は外生パラメータを表します。

??? tip "GAMS シンボル命名規則"
    モデルは可読性と一貫性を確保するため、GAMS シンボルに体系的な命名規則を採用しています。

    **決定変数（ドキュメント ↔ GAMS 対応）：**

    - すべての決定変数はドキュメント・GAMS 実装ともに `V` で始まります
    - ドキュメントでは技術に依存しない汎用記法 $VX$, $VS$, $VR$, $VC$, $VP$ を使用します
    - GAMS では 2 文字目が技術種別を示すプレフィックスを使用します：
        - `VG*` 発電機： $VX_{n,i,r,t}$ → `VG(N,I,R,T)`, $VS_{n,i,r}$ → `VGS(N,I,R)`, $VR_{n,i,r}$ → `VGR(N,I,R)`, $VC_{n,i,r_0,r_1,y}$ → `VGC(N,I,R0,R1,Y)`, $VP_{n,i,r}$ → `VGP(N,I,R)`
        - `VF*` リンク： $VX^+_{l,t}$ → `VFF(L,T)`, $VX^-_{l,t}$ → `VFB(L,T)`, $VS_l$ → `VFS(L)`, $VR_l$ → `VFR(L)`, $VC_{l_0,l_1,y}$ → `VFC(L0,L1,Y)`, $VP_l$ → `VFP(L)`
        - `VE*` 蓄電（エネルギー）： $VE_{n,i,s,t}$ → `VE(N,I,S,T)`, $VS_{n,i,s}$ → `VES(N,I,S)`, $VR_{n,i,s}$ → `VER(N,I,S)`, $VC_{n,i,s_0,s_1,y}$ → `VEC(N,I,S0,S1,Y)`, $VP_{n,i,s}$ → `VEP(N,I,S)`
        - `VH*` 蓄電（電力）： $VX_{n,i,s,t}$ → `VH(N,I,S,T)`
        - `VP*` エネルギー供給： $VP_{n,i,k}$ → `VP(N,I,K)`
        - `VQ*` 排出量： $VQ_{n,i,g}$ → `VQ(N,I,G)`
    - GAMS の 3 文字目は変数種別を示します： `S`（ストック/容量）、`R`（新設）、`C`（レトロフィット）、`P`（更新）、`X`（運転/出力）、`F`/`B`（順方向/逆方向フロー）
    - 特例：総コストはドキュメントでは $VTC$、GAMS では `VTC`

    **パラメータ：**

    - 技術固有パラメータは技術種別の文字で始まります： `g`（発電機）、`f`（リンク）、`e`（蓄電）
    - コストパラメータ： `cc`（資本費）、`foc`（固定 O&M コスト）、`voc`（変動 O&M コスト）、`rc`（レトロフィット費用）、`pc`（更新費用）、`acc`（年率化資本費）、`arc`（年率化レトロフィット費用）、`apc`（年率化更新費用）
    - 技術パラメータ： `mn`（最小）、`mx`（最大）、`fx`（固定）、`lt`（耐用年数）、`alp`（割引率）、`eta`（効率）
    - ストックパラメータ： `sc`（設置年次別ストック）、`ssc`（残存ストック）、`sr`（更新可能ストック）、`ssr`（更新可能残存ストック）
    - 例： `gacc` = 発電機（g）＋年率化資本費（acc）、`gmx` = 発電機（g）＋最大（mx）、`eeta` = 蓄電（e）＋自己放電率（eta）

    **集合・マッピング：**

    - 集約集合は基本集合の文字に `M` を付加して命名： `MR`（発電機グループ）、`ME`（地域-セクター対）、`MK`（エネルギー種別グループ）等
    - マッピング集合は `M_` に集合名を付加： `M_MR`（MR→R マッピング）、`M_ME`（ME→N,I マッピング）
    - フラグ集合は `FL_` で始まります： `FL_IR`（有効な地域-セクター-技術の組み合わせ）、`FL_IRT`（有効な地域-セクター-技術-時間スライスの組み合わせ）

## 集合と添字

モデルは各次元に対して基本添字を使用し、これらの要素をグループ化した集約集合を持ちます。

### 基本添字

| 表記 | シンボル | 説明 |
| ---- | -------- | ---- |
| $n \in N$ | `N` | 地域 |
| $i \in I$ | `I` | セクター |
| $r \in R$ | `R` | 発電機技術 |
| $l \in L$ | `L` | リンク |
| $s \in S$ | `S` | 蓄電技術 |
| $t \in T$ | `T` | 時間スライス |
| $k \in K$ | `K` | エネルギー種別 |
| $g \in G$ | `G` | 排出ガス種別 |
| $y \in Y$ | `Y` | 年（過去・将来を含む全期間）；シミュレーション期間は部分集合 `YEAR(Y)` で定義 |

### 集約・マッピング

集約集合は制約を集約レベルで適用するためのグループを定義します。以下の表に集約集合・グループ添字・GAMS 実装をまとめます：

| グループ表記 | 説明 | GAMS 集合 | GAMS マッピング |
| ------------ | ---- | --------- | -------------- |
| $m_r \in MR$ | 発電機技術グループ（$MR \subseteq R$） | `MR` | `M_MR(MR,R)` |
| $m_l \in ML$ | ノード-セクター別リンクグループ | `ML` | `M_IMLL(N,I,ML,L)` |
| $m_e \in ME$ | ノード-セクターグループ（$ME \subseteq N \times I$） | `ME` | `M_ME(ME,N,I)` |
| $k_m \in MK$ | エネルギー種別グループ（$MK \subseteq K$） | `MK` | `M_MK(MK,K)` |
| $m_q \in MQ$ | 排出制約用ノード-セクターグループ（$MQ \subseteq N \times I$） | `MQ` | `M_MQ(MQ,N,I)` |
| $g_m \in MG$ | 排出ガスグループ（$MG \subseteq G$） | `MG` | `M_MG(MG,G)` |
| $m_t \in MT$ | 時間スライスグループ（$MT \subseteq T$） | `MT` | `M_MT(MT,T)` |
| $m_x \in MX$ | シェア制約用ノード-セクターグループ（$MX \subseteq N \times I$） | `MX` | `M_MX(MX,N,I)` |

方程式中では、グループへの所属をマッピング集合の添字で表記します。例えば $\sum_{(n,i) \in ME_{m_e}}$ はエネルギーグループ $m_e$ に属するすべてのノード-セクター対 $(n,i)$ について和をとり、$\sum_{k \in MK_{k_m}}$ はエネルギー種別グループ $k_m$ に属するすべてのエネルギー種別 $k$ について和をとります。

レトロフィットの実行可能性は専用のマッピング集合で表現されます：

| レトロフィットマッピング | 説明 | GAMS マッピング |
| ------------------------ | ---- | -------------- |
| $RC_{r_0,r_1}$ | 発電機技術の実行可能レトロフィット関係（$r_0$ から $r_1$ へ） | `M_RC(R0,R1)` |
| $LC_{l_0,l_1}$ | リンクの実行可能レトロフィット関係（$l_0$ から $l_1$ へ） | `M_LC(L0,L1)` |
| $SC_{s_0,s_1}$ | 蓄電技術の実行可能レトロフィット関係（$s_0$ から $s_1$ へ） | `M_SC(S0,S1)` |

方程式では、これらの関係を添字付き和の条件として使用します。例えば $\sum_{r_0:\,RC_{r_0,r}}$ は目的技術 $r$ にレトロフィット可能な転換元技術 $r_0$ すべてについて和をとることを意味します。

### 時間スライスパラメータ

本文書全体を通じて、以下の時間スライス関連パラメータを使用します：

| 表記 | シンボル | 説明 | 単位 |
| ---- | -------- | ---- | ---- |
| $\omega$ | `wt(T)` | 時間スライスの重み（各時間スライスの時間数） | 時間 |
| $\Delta t$ | `rt(T)` | 時間スライスの粒度（時間数；ランピングと蓄電ダイナミクスに使用） | 時間 |
| $\Delta y$ | `t_int` | シミュレーション年の間隔 | 年 |

!!! note "注記"

    2 つのパラメータは本質的に異なる役割を担います：$\omega$ は出力（GW）を時間スライスにわたって**積算**してエネルギー（GWh）を得るために用います（$\sum_t \omega \cdot VX$ の形）；$\Delta t$ はランプ制約（$\rho \cdot \Delta t_{t_0} \cdot VS$、単位 GW）や自己放電指数（$\eta^{\Delta t}$、無次元）のような**速度・継続時間**の表現に用います。

    $\Delta t$ は常に時間スライス固有の値です：ランピング制約は直前の時間スライスの継続時間 $\Delta t_{t_0} = rt(T0)$ を使用し、蓄電エネルギーバランスは自己放電指数に $\Delta t_{t_0}$、発電・充電の継続時間に $\Delta t_{t_1} = rt(T1)$ を使用します。GAMS 実装では `wt(T)` と `rt(T)` はともに時間スライス $t$ でインデックスされます。

## 目的関数

### 総システムコスト

目的関数は総システムコスト $VTC$ を最小化します。$VTC$ はすべての技術・地域・セクターにわたる CAPEX（資本的支出）$C^{\text{CAPEX}}$、固定 OPEX（固定運転費）$C^{\text{Fixed OPEX}}$、変動 OPEX $C^{\text{Variable OPEX}}$、排出税 $C^{\text{Emission tax}}$ の和です。

$$
{VTC} = {C^{\text{CAPEX}}} + {C^{\text{Fixed OPEX}}} + {C^{\text{Variable OPEX}}} + {C^{\text{Emission tax}}} \to \min
$$

資本的支出 $C^{\text{CAPEX}}$ は、新設コスト $C^{\text{CAPEX,new}}$、レトロフィットコスト $C^{\text{CAPEX,retrofit}}$、更新コスト $C^{\text{CAPEX,replace}}$ に分解されます。第 1 項は新設容量、第 2 項は設置年次または技術クラス間の機器転換、第 3 項は耐用年数超過あるいは更新可能なストックの置き換えに対するコストです。

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

固定運転費 $C^{\text{Fixed OPEX}}$ は設置済みストック変数に課されます。現在の実装では、ストックバランスのスラック変数 $\delta^{\text{stock}}$ にも同じ固定費係数でペナルティを与え、未解消のストック不整合が目的値に反映されるようにしています。

$$
{C^{\text{Fixed OPEX}}} =
\sum_{n,i,r}{o^{\text{fix}}}_{n,i,r}\cdot \left({VS}_{n,i,r}+{\delta^{\text{stock}}}_{n,i,r}\right)
+ \sum_{l}{o^{\text{fix}}}_{l}\cdot \left({VS}_{l}+{\delta^{\text{stock}}}_{l}\right)
+ \sum_{n,i,s}{o^{\text{fix}}}_{n,i,s}\cdot \left({VS}_{n,i,s}+{\delta^{\text{stock}}}_{n,i,s}\right)
$$

変動運転費 $C^{\text{Variable OPEX}}$ は、発電機出力 $VX_{n,i,r,t}$ およびリンクフロー $VX^+_{l,t}$, $VX^-_{l,t}$ にのみ発生します。現在の定式化では、蓄電の充放電電力 $VX_{n,i,s,t}$ はこの項に直接現れません。

$$
{C^{\text{Variable OPEX}}} = \sum_{n,i,r,t}{\omega}\cdot {o^{\text{variable}}}_{n,i,r}\cdot {VX}_{n,i,r,t}
+ \sum_{l,t}{\omega}\cdot {o^{\text{variable}}}_{l}\cdot \left({VX}^{+}_{l,t}+{VX}^{-}_{l,t}\right)
$$

排出税 $C^{\text{Emission tax}}$ は、排出グループに対して排出税率 $q^{\text{tax}}_{m_q,m_g}$ が指定されているときに発生します。各ノード-セクター-ガス組み合わせ $(n,i,g)$ について、$(n,i)$ と $g$ が属するすべての排出グループ $(m_q, m_g)$ にわたって税を合計します：

$$
{C^{\text{Emission tax}}} = \sum_{n,i,g} {VQ}_{n,i,g} \cdot \sum_{\substack{m_q:\,(n,i)\in MQ_{m_q} \\ m_g:\,g\in MG_{m_g}}} q^{\text{tax}}_{m_q,m_g}
$$

<details markdown="1">
<summary><b>総システムコストで使用するパラメータ</b></summary>

| パラメータ | シンボル | 説明 | 単位 |
| ---------- | -------- | ---- | ---- |
| $c^{\text{new}}_{n,i,r}$ | `gacc(N,I,R)` | 年率化資本費（新設） | 百万 USD/GW |
| $c^{\text{retrofit}}_{n,i,r_0,r_1,y}$ | `garc(N,I,R0,R1,Y)` | 年率化レトロフィット費用 | 百万 USD/GW |
| $o^{\text{fix}}_{n,i,r}$ | `gfoc(N,I,R)` | 固定 O&M コスト | 百万 USD/GW |
| $c^{\text{replace}}_{n,i,r}$ | `gapc(N,I,R)` | 年率化資本費（更新） | 百万 USD/GW |
| $o^{\text{variable}}_{n,i,r}$ | `gvoc(N,I,R)` | 変動 O&M コスト | 百万 USD/GWh |
| $c^{\text{new}}_{l}$ | `facc(L)` | 年率化資本費（新設） | 百万 USD/GW |
| $c^{\text{retrofit}}_{l_0,l_1,y}$ | `farc(L0,L1,Y)` | 年率化レトロフィット費用 | 百万 USD/GW |
| $o^{\text{fix}}_{l}$ | `ffoc(L)` | 固定 O&M コスト | 百万 USD/GW |
| $c^{\text{replace}}_{l}$ | `fapc(L)` | 年率化資本費（更新） | 百万 USD/GW |
| $o^{\text{variable}}_{l}$ | `fvoc(L)` | 変動 O&M コスト | 百万 USD/GWh |
| $q^{\text{tax}}_{m_q,m_g}$ | `qtax(MQ,MG)` | 排出税率 | 百万 USD/MtCO₂ |
| $c^{\text{new}}_{n,i,s}$ | `eacc(N,I,S)` | 年率化資本費（新設） | 百万 USD/GWh |
| $c^{\text{retrofit}}_{n,i,s_0,s_1,y}$ | `earc(N,I,S0,S1,Y)` | 年率化レトロフィット費用 | 百万 USD/GWh |
| $o^{\text{fix}}_{n,i,s}$ | `efoc(N,I,S)` | 固定 O&M コスト | 百万 USD/GWh |
| $c^{\text{replace}}_{n,i,s}$ | `eapc(N,I,S)` | 年率化資本費（更新） | 百万 USD/GWh |
| $\omega$ | `wt(T)` | 時間スライスの重み | 時間 |

</details>

<details markdown="1">
<summary><b>総システムコストで使用する決定変数</b></summary>

| 変数 | シンボル | 説明 | 単位 |
| ---- | -------- | ---- | ---- |
| $VTC$ | `VTC` | 最小化する総システムコスト | 百万 USD |
| $VR_{n,i,r}$ | `VGR(N,I,R)` | 発電機の新設容量 | GW/yr |
| $VS_{n,i,r}$ | `VGS(N,I,R)` | 発電機の設置済み容量 | GW |
| $VC_{n,i,r_0,r_1,y}$ | `VGC(N,I,R0,R1,Y)` | 発電機技術間のレトロフィットフロー | GW |
| $VP_{n,i,r}$ | `VGP(N,I,R)` | 発電機の更新容量 | GW/yr |
| $VX_{n,i,r,t}$ | `VG(N,I,R,T)` | 発電機の電力出力 | GW |
| $VR_{l}$ | `VFR(L)` | リンクの新設容量 | GW/yr |
| $VS_{l}$ | `VFS(L)` | リンクの設置済み容量 | GW |
| $VC_{l_0,l_1,y}$ | `VFC(L0,L1,Y)` | リンク間のレトロフィットフロー | GW |
| $VP_{l}$ | `VFP(L)` | リンクの更新容量 | GW/yr |
| $VX^{+}_{l,t}$ | `VFF(L,T)` | リンクの順方向エネルギーフロー | GW |
| $VX^{-}_{l,t}$ | `VFB(L,T)` | リンクの逆方向エネルギーフロー | GW |
| $VR_{n,i,s}$ | `VER(N,I,S)` | 蓄電装置の新設容量 | GWh/yr |
| $VS_{n,i,s}$ | `VES(N,I,S)` | 蓄電装置の設置済みエネルギー容量 | GWh |
| $VC_{n,i,s_0,s_1,y}$ | `VEC(N,I,S0,S1,Y)` | 蓄電技術間のレトロフィットフロー | GWh |
| $VP_{n,i,s}$ | `VEP(N,I,S)` | 蓄電装置の更新容量 | GWh/yr |
| $\delta^{\text{stock}}_{n,i,r}$ | `RES_GSCB(N,I,R)` | 発電機ストックバランスのスラック変数 | GW |
| $\delta^{\text{stock}}_{l}$ | `RES_FSCB(L)` | リンクストックバランスのスラック変数 | GW |
| $\delta^{\text{stock}}_{n,i,s}$ | `RES_ESCB(N,I,S)` | 蓄電ストックバランスのスラック変数 | GWh |
| $VQ_{n,i,g}$ | `VQ(N,I,G)` | 年間排出量（ガス種別 $g$） | MtCO₂ |

</details>

### 資本費の年率化

新設・レトロフィット・更新の各コストは、技術固有の割引率と耐用年数に基づいて年率化されます。以下では発電機を代表例として示します。

!!! note "$\Delta y$ の役割"
    $\Delta y$ はレート→ストックの単位換算であり、追加の割引ではありません。**新設** $VR$・**更新** $VP$ は年間レート（GW/年）であり、ストックバランス式では $\Delta y$ を掛けて積算されます。そのため $c\cdot\Delta y$ と対にすることで「追加 GW あたり 1 年分の年金」を正しく課します。一方**レトロフィット** $VC$ は既にストック（GW）でありコホートに直接加算されるため、その年率化費用 $c^{\text{retrofit}}$ には $\Delta y$ を**付けません**（レトロフィット GW あたり 1 年分の年金）。

発電機の年率化費用 $c^{\text{new}}_{n,i,r}$ および $c^{\text{replace}}_{n,i,r}$ は、初期資本費 $b^{\text{new}}_{n,i,r}$、$b^{\text{replace}}_{n,i,r}$、割引率 $\alpha_{n,i,r}$、耐用年数 $\tau_{n,i,r}$ から計算されます：

$$
{c^{\text{new}}}_{n, i, r} = {b^{\text{new}}}_{n, i, r} \cdot \frac{\alpha_{n, i, r} \cdot \left(1 + \alpha_{n, i, r}\right)^{\tau_{n, i, r}}}
{\left(1 + \alpha_{n, i, r}\right)^{\tau_{n, i, r}}-1} \cdot \Delta y
$$

$$
{c^{\text{replace}}}_{n, i, r} = {b^{\text{replace}}}_{n, i, r} \cdot \frac{\alpha_{n, i, r} \cdot \left(1 + \alpha_{n, i, r}\right)^{\tau_{n, i, r}}}
{\left(1 + \alpha_{n, i, r}\right)^{\tau_{n, i, r}}-1} \cdot \Delta y
$$

レトロフィットの年率化費用 $c^{\text{retrofit}}_{n,i,r_0,r_1,y}$ は、レトロフィット費用係数 $b^{\text{retrofit}}_{n,i,r_0,r_1}$、転換先の割引率 $\alpha_{n,i,r_1}$、レトロフィット返済期間 $\tau^{\text{retrofit}}_{n,i,r_0,y}$ を用います。返済期間は設置年次 $y$ における転換元技術の残余耐用年数（最低 1 年）であり、$y^*$ はシミュレーション年を表します：

$$
{\tau^{\text{retrofit}}}_{n,i,r_0,y}
= \max \left(1,\ \tau_{n,i,r_0} - \left(y^* - y\right)\right)
$$

発電機の場合：

$$
{c^{\text{retrofit}}}_{n, i, r_0, r_1, y}
= {b^{\text{retrofit}}}_{n, i, r_0, r_1}
\cdot
\frac{\alpha_{n, i, r_1}\cdot \left(1+\alpha_{n, i, r_1}\right)^{{\tau^{\text{retrofit}}}_{n,i,r_0,y}}}
{\left(1+\alpha_{n, i, r_1}\right)^{{\tau^{\text{retrofit}}}_{n,i,r_0,y}}-1}
$$

<details markdown="1">
<summary><b>資本費の年率化で使用するパラメータ</b></summary>

| パラメータ | シンボル | 説明 | 単位 |
| ---------- | -------- | ---- | ---- |
| $b^{\text{new}}_{n,i,r}$ | `gcc(N,I,R)` | 発電機新設の初期資本費（年率化前） | 百万 USD/GW |
| $b^{\text{retrofit}}_{n,i,r_0,r_1}$ | `grc(N,I,R0,R1)` | 発電機レトロフィットの初期費用 | 百万 USD/GW |
| $b^{\text{replace}}_{n,i,r}$ | `gpc(N,I,R)` | 発電機更新の初期資本費（年率化前） | 百万 USD/GW |
| $c^{\text{new}}_{n,i,r}$ | `gacc(N,I,R)` | 発電機新設の年率化資本費 | 百万 USD/GW |
| $c^{\text{retrofit}}_{n,i,r_0,r_1,y}$ | `garc(N,I,R0,R1,Y)` | 発電機の年率化レトロフィット費用 | 百万 USD/GW |
| $c^{\text{replace}}_{n,i,r}$ | `gapc(N,I,R)` | 発電機の年率化更新費用 | 百万 USD/GW |
| $\alpha_{n,i,r}$ | `galp(N,I,R)` | 発電機の割引率 | - |
| $\tau_{n,i,r}$ | `glt(N,I,R)` | 発電機の耐用年数 | 年 |
| ${\tau^{\text{retrofit}}}_{n,i,r_0,y}$ | `grt(N,I,R0,Y)` | 年率化に用いるレトロフィット返済期間 | 年 |
| $\Delta y$ | `t_int` | シミュレーション年の間隔 | 年 |

</details>

## 発電機の運転・容量

発電機制約は出力 $VX_{n,i,r,t}$ と新設投資 $VR_{n,i,r}$ を規定します。出力はノード電力バランスおよびエネルギー供給の方程式に現れ、設置済み容量 $VS_{n,i,r}$ は技術ストック方程式によって決定されます。

### 発電機の最小/最大出力

各発電機の出力は、設置済み容量と技術固有の出力制限係数によって上下限が設けられます。係数 $\gamma^{\min}_{n,i,r,t}$ と $\gamma^{\max}_{n,i,r,t}$ は、設置済み容量を各時間スライスの出力下限・上限に換算します。

$$
{{\gamma^{\min}}}_{n, i, r, t} \cdot {VS}_{n, i, r} \leq {VX}_{n, i, r, t} \leq {{\gamma^{\max}}}_{n, i, r, t} \cdot {VS}_{n, i, r}
$$

<details markdown="1">
<summary><b>最小/最大出力で使用するパラメータ</b></summary>

| パラメータ | シンボル | 説明 | 単位 |
| ---------- | -------- | ---- | ---- |
| $\gamma^{\min}_{n,i,r,t}$ | `gmn(N,I,R,T)` | 発電機の最小出力係数 | 0–1 |
| $\gamma^{\max}_{n,i,r,t}$ | `gmx(N,I,R,T)` | 最大出力係数（利用可能率） | 0–1 |

</details>

<details markdown="1">
<summary><b>最小/最大出力で使用する決定変数</b></summary>

| 変数 | シンボル | 説明 | 単位 |
| ---- | -------- | ---- | ---- |
| $VX_{n,i,r,t}$ | `VG(N,I,R,T)` | 発電機の電力出力 | GW |
| $VS_{n,i,r}$ | `VGS(N,I,R)` | 発電機の設置済み容量 | GW |

</details>

!!! tip "出力抑制の扱い"
    風力・太陽光などの非調整可能電源では、$VX_{n,i,r,t}$ は実際の発電量を表し、経済的な出力抑制により潜在発電量 $\gamma^{\max}_{n,i,r,t} \cdot VS_{n,i,r}$ を下回ることがあります。抑制量はポスト処理で $\gamma^{\max}_{n,i,r,t} \cdot VS_{n,i,r} - VX_{n,i,r,t}$ として計算されます。

### ランピング制約（オプション）

上昇・下降ランプ制約は、連続する時間スライス $t_0$（直前）と $t_1$（現在）の間における発電機出力の変化速度を制限します。この制約は連続する出力レベルの差分に課され、時間をまたいで発電機運転を連動させます。ランプ制限は、**直前**の時間スライスの継続時間 $\Delta t_{t_0}$ と利用可能率 $\gamma^{\max}_{n,i,r,t_0}$ でスケーリングされます。

$$
 -{{\Delta t_{t_0}}}\cdot {{\rho^{\text{down}}}}_{n,i,r}\cdot {{\gamma^{\max}}}_{n,i,r,t_0}\cdot {VS}_{n,i,r}
\leq {VX}_{n,i,r,t_1}-{VX}_{n,i,r,t_0}
\leq {{\Delta t_{t_0}}}\cdot {{\rho^{\text{up}}}}_{n,i,r}\cdot {{\gamma^{\max}}}_{n,i,r,t_0}\cdot {VS}_{n,i,r}
$$

<details markdown="1">
<summary><b>ランピング制約で使用するパラメータ</b></summary>

| パラメータ | シンボル | 説明 | 単位 |
| ---------- | -------- | ---- | ---- |
| $\Delta t_{t_0}$ | `rt(T0)` | 直前の時間スライス $t_0$ の継続時間 | 時間 |
| $\rho^{\text{up}}_{n,i,r}$ | `gru(N,I,R)` | 上昇ランプ率係数 | - |
| $\rho^{\text{down}}_{n,i,r}$ | `grd(N,I,R)` | 下降ランプ率係数 | - |
| $\gamma^{\max}_{n,i,r,t_0}$ | `gmx(N,I,R,T0)` | 直前の時間スライス $t_0$ における最大出力係数 | 0–1 |

</details>

<details markdown="1">
<summary><b>ランピング制約で使用する決定変数</b></summary>

| 変数 | シンボル | 説明 | 単位 |
| ---- | -------- | ---- | ---- |
| $VX_{n,i,r,t}$ | `VG(N,I,R,T)` | 発電機の電力出力 | GW |
| $VS_{n,i,r}$ | `VGS(N,I,R)` | 発電機の設置済み容量 | GW |

</details>

### 発電シェア制約

モデルは発電機グループ間の最小/最大発電シェアを課すことができます。左辺の発電機グループ $m_{r0}$ と右辺のグループ $m_{r1}$ をシェア係数 $\theta^{\min}$, $\theta^{\max}$ で比較します：ノード-セクターグループ $m_x$ にわたる $m_{r0}$ の集約出力が、$m_{r1}$ の出力の $\theta^{\min}$ 倍以上 $\theta^{\max}$ 倍以下となるよう制約します。

!!! tip "例"
    あるノード-セクターグループ内で太陽光発電が再エネ全体の 30% 以上を担うことを要件とするには、$m_{r0}$ = 太陽光技術、$m_{r1}$ = 全再エネ技術、$\theta^{\min}_{m_x, m_{r0}, m_{r1}} = 0.3$ と設定します。

$$
{{\theta^{\min}}}_{m_x,m_{r0},m_{r1}}\cdot \sum_{(n,i)\in MX_{m_x}}\sum_{r\in MR_{m_{r1}}}\sum_{t}{VX}_{n,i,r,t}
\leq \sum_{(n,i)\in MX_{m_x}}\sum_{r\in MR_{m_{r0}}}\sum_{t}{VX}_{n,i,r,t}
\leq {{\theta^{\max}}}_{m_x,m_{r0},m_{r1}}\cdot \sum_{(n,i)\in MX_{m_x}}\sum_{r\in MR_{m_{r1}}}\sum_{t}{VX}_{n,i,r,t}
$$

固定シェア制約も利用できます：

$$
{{\theta^{\text{fix}}}}_{m_x,m_{r0},m_{r1}}\cdot \sum_{(n,i)\in MX_{m_x}}\sum_{r\in MR_{m_{r1}}}\sum_{t}{VX}_{n,i,r,t}
= \sum_{(n,i)\in MX_{m_x}}\sum_{r\in MR_{m_{r0}}}\sum_{t}{VX}_{n,i,r,t}
$$

<details markdown="1">
<summary><b>発電シェア制約で使用するパラメータ</b></summary>

| パラメータ | シンボル | 説明 | 単位 |
| ---------- | -------- | ---- | ---- |
| $\theta^{\min}_{m_x,m_{r0},m_{r1}}$ | `gmmn(MX,MR0,MR1)` | 最小発電シェア係数 | - |
| $\theta^{\max}_{m_x,m_{r0},m_{r1}}$ | `gmmx(MX,MR0,MR1)` | 最大発電シェア係数 | - |
| $\theta^{\text{fix}}_{m_x,m_{r0},m_{r1}}$ | `gmfx(MX,MR0,MR1)` | 固定発電シェア係数 | - |

</details>

<details markdown="1">
<summary><b>発電シェア制約で使用する決定変数</b></summary>

| 変数 | シンボル | 説明 | 単位 |
| ---- | -------- | ---- | ---- |
| $VX_{n,i,r,t}$ | `VG(N,I,R,T)` | 発電機の電力出力 | GW |

</details>

### 発電機の容量上下限

発電機の設置済み容量 $VS_{n,i,r}$ は技術固有の下限 $\lambda^{\min}_{n,i,r}$ および上限 $\lambda^{\max}_{n,i,r}$ で制約されます。これらはストック変数 $VS_{n,i,r}$ に直接適用され、以下の新設容量 $VR_{n,i,r}$ への上下限とは別に課されます。

$$
{{\lambda^{\min}}}_{n, i, r} \leq {VS}_{n, i, r} \leq {{\lambda^{\max}}}_{n, i, r}
$$

新設容量 $VR_{n,i,r}$ も $\nu^{\min}_{n,i,r}$ および $\nu^{\max}_{n,i,r}$ で制約されます：

$$
{{\nu^{\min}}}_{n, i, r} \leq {VR}_{n, i, r} \leq {{\nu^{\max}}}_{n, i, r}
$$

<details markdown="1">
<summary><b>容量上下限で使用するパラメータ</b></summary>

| パラメータ | シンボル | 説明 | 単位 |
| ---------- | -------- | ---- | ---- |
| $\lambda^{\min}_{n,i,r}$ | `gsmn(N,I,R)` | 設置済み容量の下限 | GW |
| $\lambda^{\max}_{n,i,r}$ | `gsmx(N,I,R)` | 設置済み容量の上限 | GW |
| $\nu^{\min}_{n,i,r}$ | `grmn(N,I,R)` | 年間新設容量の下限 | GW |
| $\nu^{\max}_{n,i,r}$ | `grmx(N,I,R)` | 年間新設容量の上限 | GW |

</details>

<details markdown="1">
<summary><b>容量上下限で使用する決定変数</b></summary>

| 変数 | シンボル | 説明 | 単位 |
| ---- | -------- | ---- | ---- |
| $VS_{n,i,r}$ | `VGS(N,I,R)` | 発電機の設置済み容量 | GW |
| $VR_{n,i,r}$ | `VGR(N,I,R)` | 発電機の新設容量 | GW/yr |

</details>


## リンクの運転・容量

リンクはノード-セクター対間のエネルギー移送または変換を表します。リンクフロー $VX^+_{l,t}$ と $VX^-_{l,t}$ はノード電力バランスに純輸入・輸出として現れ、設置済み容量 $VS_l$ は技術ストック方程式によって決定されます。

### リンクフロー運転

各リンクのエネルギーフロー $VX^+_{l,t}$ と $VX^-_{l,t}$ は、設置済み容量 $VS_l$ と出力制限係数 $\gamma^{\min}_{+,l,t}$, $\gamma^{\max}_{+,l,t}$, $\gamma^{\min}_{-,l,t}$, $\gamma^{\max}_{-,l,t}$ で上下限が設けられます。順方向と逆方向のフローは損失のある輸送を表現するために別々に扱われ[^1]、2 方向のフロー変数により定式化を線形に保ちながら非対称な変換・輸送挙動を表現できます。

$$
{{\gamma^{\min}_{+}}}_{l, t} \cdot {VS}_{l} \leq {VX}^{+}_{l, t} \leq {{\gamma^{\max}_{+}}}_{l, t} \cdot {VS}_{l}
$$

$$
{{\gamma^{\min}_{-}}}_{l, t} \cdot {VS}_{l} \leq {VX}^{-}_{l, t} \leq {{\gamma^{\max}_{-}}}_{l, t} \cdot {VS}_{l}
$$

上下限を等値に設定することで固定フローを表現できます：

$$
{VX}^{+}_{l,t} = {{\gamma^{\text{fix}}_{+}}}_{l,t}\cdot {VS}_{l},\qquad {VX}^{-}_{l,t} = {{\gamma^{\text{fix}}_{-}}}_{l,t}\cdot {VS}_{l}
$$

<details markdown="1">
<summary><b>リンクフロー運転で使用するパラメータ</b></summary>

| パラメータ | シンボル | 説明 | 単位 |
| ---------- | -------- | ---- | ---- |
| $\gamma^{\min}_{+,l,t}$ | `ffmn(L,T)` | 順方向フローの最小係数 | 0–1 |
| $\gamma^{\max}_{+,l,t}$ | `ffmx(L,T)` | 順方向フローの最大係数 | 0–1 |
| $\gamma^{\min}_{-,l,t}$ | `fbmn(L,T)` | 逆方向フローの最小係数 | 0–1 |
| $\gamma^{\max}_{-,l,t}$ | `fbmx(L,T)` | 逆方向フローの最大係数 | 0–1 |
| $\gamma^{\text{fix}}_{+,l,t}$ | `fffx(L,T)` | 順方向フローの固定係数 | 0–1 |
| $\gamma^{\text{fix}}_{-,l,t}$ | `fbfx(L,T)` | 逆方向フローの固定係数 | 0–1 |

</details>

<details markdown="1">
<summary><b>リンクフロー運転で使用する決定変数</b></summary>

| 変数 | シンボル | 説明 | 単位 |
| ---- | -------- | ---- | ---- |
| $VX^{+}_{l,t}$ | `VFF(L,T)` | リンクの順方向エネルギーフロー | GW |
| $VX^{-}_{l,t}$ | `VFB(L,T)` | リンクの逆方向エネルギーフロー | GW |
| $VS_{l}$ | `VFS(L)` | リンクの設置済み容量 | GW |

</details>

!!! note "双方向リンクと一方向リンク"
    リンクはインバータなどの**双方向リンク**と、電解槽などの**一方向リンク**に分類されます。一方向リンクでは $\gamma^{\max}_{-,l,t} = 0$ と設定し、エネルギーフローを一方向のみに制限します。

### リンクの容量上下限

リンクの設置済み容量 $VS_l$ は下限 $\lambda^{\min}_l$ および上限 $\lambda^{\max}_l$ で制約されます。発電機と同様に、ストック上下限と新設上下限は別々に課されます：

$$
{{\lambda^{\min}}}_{l} \leq {VS}_{l} \leq {{\lambda^{\max}}}_{l}
$$

新設容量 $VR_l$ は $\nu^{\min}_l$ および $\nu^{\max}_l$ で制約されます：

$$
{{\nu^{\min}}}_{l} \leq {VR}_{l} \leq {{\nu^{\max}}}_{l}
$$

<details markdown="1">
<summary><b>リンク容量上下限で使用するパラメータ</b></summary>

| パラメータ | シンボル | 説明 | 単位 |
| ---------- | -------- | ---- | ---- |
| $\lambda^{\min}_{l}$ | `fsmn(L)` | リンク容量の下限 | GW |
| $\lambda^{\max}_{l}$ | `fsmx(L)` | リンク容量の上限 | GW |
| $\nu^{\min}_{l}$ | `frmn(L)` | 年間新設容量の下限 | GW |
| $\nu^{\max}_{l}$ | `frmx(L)` | 年間新設容量の上限 | GW |

</details>

<details markdown="1">
<summary><b>リンク容量上下限で使用する決定変数</b></summary>

| 変数 | シンボル | 説明 | 単位 |
| ---- | -------- | ---- | ---- |
| $VS_{l}$ | `VFS(L)` | リンクの設置済み容量 | GW |
| $VR_{l}$ | `VFR(L)` | リンクの新設容量 | GW/yr |

</details>

### リンク寄与シェア制約

モデルはリンクグループのノードフローへの寄与に制約を課すことができます。シェア係数 $\theta^{\min}_{m_x,m_{l0},m_{l1}}$ と $\theta^{\max}_{m_x,m_{l0},m_{l1}}$ は、ノード-セクターグループ $m_x \in MX$ にわたって集約された 2 つのリンクグループ $m_{l0}$ と $m_{l1}$ の接続行列加重寄与を比較します。

比較対象は各ノード-セクター対 $(n,i)$ が各リンクグループから受け取る**正味エネルギー**です。時間スライス $t$ におけるリンク $l$ からの受取エネルギーを次のように定義します：

$$
\Phi_{n,i,l,t} \;\equiv\;
-{k^{+}}_{n,i,l}\cdot {VX}^{+}_{l,t}
+{k^{-}}_{n,i,l}\cdot {VX}^{-}_{l,t}
-{k^{+}}_{n,i,l,t}\cdot {VX}^{+}_{l,t}
+{k^{-}}_{n,i,l,t}\cdot {VX}^{-}_{l,t}
$$

$\Phi > 0$ は $(n,i)$ への正味流入、$\Phi < 0$ は正味流出を示します。静的項 $k^{\pm}_{n,i,l}$ は時間に依存しないトポロジーと効率を、時変項 $k^{\pm}_{n,i,l,t}$ は時間スライス依存の変換を表します。

この略記を用いると、シェア制約は次のように表されます：

$$
\theta^{\min}_{m_x,m_{l0},m_{l1}}\cdot
\sum_{(n,i)\in MX_{m_x}}\sum_{l\in ML_{n,i,m_{l1}}}\sum_{t} \Phi_{n,i,l,t}
\leq
\sum_{(n,i)\in MX_{m_x}}\sum_{l\in ML_{n,i,m_{l0}}}\sum_{t} \Phi_{n,i,l,t}
\leq
\theta^{\max}_{m_x,m_{l0},m_{l1}}\cdot
\sum_{(n,i)\in MX_{m_x}}\sum_{l\in ML_{n,i,m_{l1}}}\sum_{t} \Phi_{n,i,l,t}
$$

固定シェア制約も利用できます：

$$
\sum_{(n,i)\in MX_{m_x}}\sum_{l\in ML_{n,i,m_{l0}}}\sum_{t} \Phi_{n,i,l,t}
= \theta^{\text{fix}}_{m_x,m_{l0},m_{l1}}\cdot
\sum_{(n,i)\in MX_{m_x}}\sum_{l\in ML_{n,i,m_{l1}}}\sum_{t} \Phi_{n,i,l,t}
$$

接続行列係数 $k^+_{n,i,l}$ と $k^-_{n,i,l}$ はネットワークトポロジーとリンク効率を順方向・逆方向について符号化します。この規則により、同じリンクフロー変数をノード電力バランスとリンクシェア制約の両方で一貫して解釈できます。

- **順方向フロー接続行列** $k^+_{n,i,l}$：リンク $l$ の順方向フローのノード-セクター対 $(n,i)$ への寄与
    - 送出ノード（`N0`, `I0`）：$k^+_{n_0,i_0,l} = 1$（エネルギーがノードから出る）
    - 受入ノード（`N1`, `I1`）：$k^+_{n_1,i_1,l} = -\eta_l$
    - 非接続ノード：$k^+_{n,i,l} = 0$

- **逆方向フロー接続行列** $k^-_{n,i,l}$：リンク $l$ の逆方向フローのノード-セクター対 $(n,i)$ への寄与
    - 送出ノード（`N0`, `I0`）：$k^-_{n_0,i_0,l} = \eta_l$
    - 受入ノード（`N1`, `I1`）：$k^-_{n_1,i_1,l} = -1$（エネルギーがノードに到着）
    - 非接続ノード：$k^-_{n,i,l} = 0$

<details markdown="1">
<summary><b>リンク寄与シェア制約で使用するパラメータ</b></summary>

| パラメータ | シンボル | 説明 | 単位 |
| ---------- | -------- | ---- | ---- |
| $ML_{n,i,m_l,l}$ | `M_IMLL(N,I,ML,L)` | リンク技術グループからノード-セクター対のアクティブリンクへのマッピング | binary |
| $k^{+}_{n,i,l}$ | `kff(N,I,L)` | 順方向フローの接続行列 | - |
| $k^{-}_{n,i,l}$ | `kbf(N,I,L)` | 逆方向フローの接続行列 | - |
| $k^{+}_{n,i,l,t}$ | `kffv(N,I,L,T)` | 順方向フローの時変接続行列 | - |
| $k^{-}_{n,i,l,t}$ | `kbfv(N,I,L,T)` | 逆方向フローの時変接続行列 | - |
| $\theta^{\min}_{m_x,m_{l0},m_{l1}}$ | `fmmn(MX,ML0,ML1)` | リンクグループシェアの最小係数 | - |
| $\theta^{\max}_{m_x,m_{l0},m_{l1}}$ | `fmmx(MX,ML0,ML1)` | リンクグループシェアの最大係数 | - |
| $\theta^{\text{fix}}_{m_x,m_{l0},m_{l1}}$ | `fmfx(MX,ML0,ML1)` | リンクグループシェアの固定係数 | - |

</details>

<details markdown="1">
<summary><b>リンク寄与シェア制約で使用する決定変数</b></summary>

| 変数 | シンボル | 説明 | 単位 |
| ---- | -------- | ---- | ---- |
| $VX^{+}_{l,t}$ | `VFF(L,T)` | リンクの順方向エネルギーフロー | GW |
| $VX^{-}_{l,t}$ | `VFB(L,T)` | リンクの逆方向エネルギーフロー | GW |

</details>

## 蓄電の運転・容量

蓄電方程式は充電状態 $VE_{n,i,s,t}$ を介して時間スライスをまたいだ運転を連動させます。充放電電力 $VX_{n,i,s,t}$ はノード電力バランスに現れ、設置済みエネルギー容量 $VS_{n,i,s}$ は技術ストック方程式によって決定されます。

### 蓄電の最小/最大運転

蓄電運転は充放電電力 $VX_{n,i,s,t}$ とエネルギーレベル $VE_{n,i,s,t}$ によって記述されます。電力 $VX_{n,i,s,t}$ は瞬時の充電または放電を制御し、エネルギーレベル $VE_{n,i,s,t}$ は時間をまたぐ充電状態（SOC）を追跡します。

$$
-\infty < {VX}_{n, i, s, t} < +\infty
$$

$$
{{\xi^{\min}}}_{n, i, s, t} \cdot {VS}_{n, i, s} \leq {VE}_{n, i, s, t} \leq {{\xi^{\max}}}_{n, i, s, t} \cdot {VS}_{n, i, s}
$$

<details markdown="1">
<summary><b>蓄電の最小/最大運転で使用するパラメータ</b></summary>

| パラメータ | シンボル | 説明 | 単位 |
| ---------- | -------- | ---- | ---- |
| $\xi^{\min}_{n,i,s,t}$ | `emn(N,I,S,T)` | 最小エネルギーレベル係数 | 0–1 |
| $\xi^{\max}_{n,i,s,t}$ | `emx(N,I,S,T)` | 最大エネルギーレベル係数 | 0–1 |

</details>

<details markdown="1">
<summary><b>蓄電の最小/最大運転で使用する決定変数</b></summary>

| 変数 | シンボル | 説明 | 単位 |
| ---- | -------- | ---- | ---- |
| $VE_{n,i,s,t}$ | `VE(N,I,S,T)` | 蓄電のエネルギーレベル（SOC）、$\xi^{\min}/\xi^{\max}$ で制約 | GWh |
| $VS_{n,i,s}$ | `VES(N,I,S)` | 蓄電の設置済みエネルギー容量 | GWh |
| $VX_{n,i,s,t}$ | `VH(N,I,S,T)` | 充放電電力（無制限：$-\infty$ から $+\infty$；正：放電） | GW |

</details>

### 蓄電の容量上下限

蓄電の設置済みエネルギー容量 $VS_{n,i,s}$ は下限 $\lambda^{\min}_{n,i,s}$ および上限 $\lambda^{\max}_{n,i,s}$ で制約されます。これらの上下限は充放電電力 $VX_{n,i,s,t}$ ではなく、エネルギー容量ストック $VS_{n,i,s}$ に適用されます。

$$
{{\lambda^{\min}}}_{n, i, s} \leq {VS}_{n, i, s} \leq {{\lambda^{\max}}}_{n, i, s}
$$

新設容量 $VR_{n,i,s}$ は $\nu^{\min}_{n,i,s}$ および $\nu^{\max}_{n,i,s}$ で制約されます。

$$
{{\nu^{\min}}}_{n, i, s} \leq {VR}_{n, i, s} \leq {{\nu^{\max}}}_{n, i, s}
$$

<details markdown="1">
<summary><b>蓄電容量上下限で使用するパラメータ</b></summary>

| パラメータ | シンボル | 説明 | 単位 |
| ---------- | -------- | ---- | ---- |
| $\lambda^{\min}_{n,i,s}$ | `esmn(N,I,S)` | 設置済みエネルギー容量の下限 | GWh |
| $\lambda^{\max}_{n,i,s}$ | `esmx(N,I,S)` | 設置済みエネルギー容量の上限 | GWh |
| $\nu^{\min}_{n,i,s}$ | `ermn(N,I,S)` | 新設容量の下限 | GWh |
| $\nu^{\max}_{n,i,s}$ | `ermx(N,I,S)` | 新設容量の上限 | GWh |

</details>

<details markdown="1">
<summary><b>蓄電容量上下限で使用する決定変数</b></summary>

| 変数 | シンボル | 説明 | 単位 |
| ---- | -------- | ---- | ---- |
| $VS_{n,i,s}$ | `VES(N,I,S)` | 蓄電の設置済みエネルギー容量 | GWh |
| $VR_{n,i,s}$ | `VER(N,I,S)` | 蓄電の新設容量 | GWh/yr |

</details>

### 蓄電の充電状態（時系列）

時間スライス $t_1$ における蓄電エネルギーレベル $VE_{n,i,s,t_1}$ は、直前の時間スライス $t_0$（$FL\_TT(t_0,t_1)$ が成立）でのエネルギーレベル、自己放電、および $t_1$ 中の充放電電力 $VX_{n,i,s,t_1}$ に依存します。この方程式は蓄電の時間間連動条件であり、ある時間スライスの運転が次のスライスの実行可能状態にどう影響するかを規定します。

$$
{VE}_{n, i, s, t_1} = {{\eta}_{n, i, s}}^{{\Delta t_{t_0}}} \cdot {VE}_{n, i, s, t_0} - {{\Delta t_{t_1}}} \cdot {VX}_{n, i, s, t_1}
$$

ここで、$\Delta t_{t_0} = rt(t_0)$ は直前の時間スライスの粒度（自己放電指数に使用）、$\Delta t_{t_1} = rt(t_1)$ は現在の時間スライスの粒度（充放電継続時間に使用）です。すべての時間スライスで継続時間が等しい場合、$VE_{n,i,s,t} = \eta_{n,i,s}^{\Delta t} \cdot VE_{n,i,s,t-1} - \Delta t \cdot VX_{n,i,s,t}$ に簡略化されます。

ここで使用する符号規則では、$VX_{n,i,s,t} > 0$ が放電（蓄電エネルギー減少）、$VX_{n,i,s,t} < 0$ が充電（蓄電エネルギー増加）を表します。

<details markdown="1">
<summary><b>蓄電充電状態（時系列）で使用するパラメータ</b></summary>

| パラメータ | シンボル | 説明 | 単位 |
| ---------- | -------- | ---- | ---- |
| $\eta_{n,i,s}$ | `eeta(N,I,S)` | 1 時間あたりの自己放電率 | - |
| $\Delta t_{t_0}$ | `rt(T0)` | 直前の時間スライスの粒度 | 時間 |
| $\Delta t_{t_1}$ | `rt(T1)` | 現在の時間スライスの粒度 | 時間 |

</details>

<details markdown="1">
<summary><b>蓄電充電状態（時系列）で使用する決定変数</b></summary>

| 変数 | シンボル | 説明 | 単位 |
| ---- | -------- | ---- | ---- |
| $VE_{n,i,s,t}$ | `VE(N,I,S,T)` | 蓄電のエネルギーレベル（SOC）、$\xi^{\min}/\xi^{\max}$ で制約 | GWh |
| $VX_{n,i,s,t}$ | `VH(N,I,S,T)` | 充放電電力（無制限；正：放電、負：充電） | GW |

</details>

!!! note "周期境界条件"

    この制約は最初の時間スライス（$t_{\text{first}}$）のエネルギーレベルが最後の時間スライス（$t_{\text{last}}$）のエネルギーレベルと等しくなることを強制し、モデル化期間にわたる周期的な運転を保証します。これにより、シミュレーション年をまたいで蓄電を恣意的に枯渇または充填することを防ぎます。

    $$
    {VE}_{n, i, s, t_{\text{first}}} = {VE}_{n, i, s, t_{\text{last}}}
    $$

### 蓄電の電力-エネルギー容量関係

揚水発電所のような一部の蓄電技術では、リンクの電力容量 $VS_l$（ポンプやタービン容量）と蓄電のエネルギー容量 $VS_{n,i,s}$（貯水池容量）が関連付けられます。関連するリンク $l$ ごとに：

$$
{VS}_{l} = \sum_{(n,i,s)\in ISL_l} {{\chi}_{n,i,s,l}}\cdot {VS}_{n,i,s}
$$

ここで $ISL_l$ はリンク $l$ に関連する蓄電技術インスタンス $(n,i,s)$ の集合です。

<details markdown="1">
<summary><b>電力-エネルギー容量関係で使用するパラメータ</b></summary>

| パラメータ | シンボル | 説明 | 単位 |
| ---------- | -------- | ---- | ---- |
| $\chi_{n,i,s,l}$ | `ecrt(N,I,S,L)` | 電力-エネルギー比率 | GW/GWh |

</details>

<details markdown="1">
<summary><b>電力-エネルギー容量関係で使用する決定変数</b></summary>

| 変数 | シンボル | 説明 | 単位 |
| ---- | -------- | ---- | ---- |
| $VS_{l}$ | `VFS(L)` | リンクの設置済み容量 | GW |
| $VS_{n,i,s}$ | `VES(N,I,S)` | 蓄電の設置済みエネルギー容量 | GWh |

</details>

## ノード電力バランス

ノード電力バランスは、すべての運転変数を結びつける中心的な制約です：発電機出力・蓄電充放電・リンクフローが合わさって、各ノード・各時間スライスグループの外生需要を満たす必要があります。発電機出力 $VX_{n,i,r,t}$、蓄電電力 $VX_{n,i,s,t}$、および正味リンク寄与が左辺に集約され、外生需要 $d_{n,i,MT}$ とスラック $\delta_{n,i,MT}$ の右辺に対して等式が成立します。スラック $\delta_{n,i,MT} \ge 0$ は**需要側**にあり（供給 $= d + \delta$）、**過剰供給**を吸収する自由処分バルブです（需要未充足の緩和ではありません）。上限 `res_npb_up(N,I,MT)` はほぼ全てで $\approx 0$（`eps`）＝供給を需要に一致させ、過剰供給を許容するバス（GAMS の `I_OS` フック、例：排熱処分）でのみ $+\infty$ に設定されます。

$$
\sum_{r}\sum_{t\in MT}{VX}_{n,i,r,t}+\sum_{s}\sum_{t\in MT}{VX}_{n,i,s,t}
-\sum_{l}\sum_{t\in MT}{k^{+}}_{n,i,l}{VX}^{+}_{l,t}
+\sum_{l}\sum_{t\in MT}{k^{-}}_{n,i,l}{VX}^{-}_{l,t}
-\sum_{l}\sum_{t\in MT}{k^{+}}_{n,i,l,t}{VX}^{+}_{l,t}
+\sum_{l}\sum_{t\in MT}{k^{-}}_{n,i,l,t}{VX}^{-}_{l,t}
= {d}_{n,i,MT}+\delta_{n,i,MT}
$$

左辺では $\sum_r\sum_{t\in MT} VX_{n,i,r,t}$ が発電機出力を、$\sum_s\sum_{t\in MT} VX_{n,i,s,t}$ が蓄電の放電・充電電力を集約し、接続行列加重リンク項がリンク経由の純輸入・輸出を表します。右辺では供給が外生需要 $d_{n,i,MT}$ ＋ 処分スラック $\delta_{n,i,MT}$ に等しく、上限が許す範囲で過剰供給が $\delta$ に吸収されます。

<details markdown="1">
<summary><b>ノード電力バランスで使用するパラメータ</b></summary>

| パラメータ | シンボル | 説明 | 単位 |
| ---------- | -------- | ---- | ---- |
| $d_{n,i,MT}$ | `dmd(N,I,MT)` | 外生需要（時間スライスグループ集計） | GW |
| $k^{+}_{n,i,l}$ | `kff(N,I,L)` | 順方向リンクフローの接続行列 | - |
| $k^{-}_{n,i,l}$ | `kbf(N,I,L)` | 逆方向リンクフローの接続行列 | - |
| $k^{+}_{n,i,l,t}$ | `kffv(N,I,L,T)` | 順方向リンクフローの時変接続行列 | - |
| $k^{-}_{n,i,l,t}$ | `kbfv(N,I,L,T)` | 逆方向リンクフローの時変接続行列 | - |
| $\delta^{\max}_{n,i,MT}$ | `res_npb_up(N,I,MT)` | ノード電力バランスのスラック上限 | GW |

</details>

<details markdown="1">
<summary><b>ノード電力バランスで使用する決定変数</b></summary>

| 変数 | シンボル | 説明 | 単位 |
| ---- | -------- | ---- | ---- |
| $VX_{n,i,r,t}$ | `VG(N,I,R,T)` | 発電機の電力出力 | GW |
| $VX_{n,i,s,t}$ | `VH(N,I,S,T)` | 蓄電の充放電電力 | GW |
| $VX^{+}_{l,t}$ | `VFF(L,T)` | リンクの順方向エネルギーフロー | GW |
| $VX^{-}_{l,t}$ | `VFB(L,T)` | リンクの逆方向エネルギーフロー | GW |
| $\delta_{n,i,MT}$ | `RES_NPB(N,I,MT)` | ノード電力バランスのスラック変数 | GW |

</details>

!!! tip "時間スライス集約"
    この制約は個々の時間スライス $t$ ではなく時間スライスグループ $MT$ にわたって集約されるため、時間分解能の柔軟な設定が可能です。代表的なグループ設定：

    - $MT$ = {全時間スライス}：年間エネルギーバランス
    - $MT$ = {単一時間スライス $t$}：時間・サブ時間バランス

    この設計により、モデルはアプリケーションに応じて異なる時間スケールでバランスを強制できます。

## 技術ストック

技術ストック方程式は設置済み容量 $VS$ を決定します。この $VS$ はすべての運転容量制約に現れます。ストックは新規投資・存続した歴史的設置年次・オプションのレトロフィットフローで構成されます。

設置済み容量 $VS$ は、新設 $VR$、更新 $VP$、存続設置年次別ストック $sc$、レトロフィットの流入・流出 $VC$、ストックバランスのスラック $\delta^{\text{stock}}$ から構成されます。存続ストック $sc_{n,i,r,y}$ は**外生パラメータ**であり、最適化の前に歴史的設置記録と残存関数から事前計算されます。投資項の $\Delta y$ は年間追加量（GW/年）をシミュレーションステップあたりの容量追加（GW/ステップ）に換算します。発電機の場合：

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

リンクと蓄電についても同様に：

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

ここで $RC_{r_0,r_1}$, $LC_{l_0,l_1}$, $SC_{s_0,s_1}$ は転換元から転換先への実行可能レトロフィット関係を表し、GAMS の `M_RC`, `M_LC`, `M_SC` に直接対応します。

更新容量 $VP_{n,i,r}$ は更新可能ストック $sr_{n,i,r}$ で制限されます。これにより更新変数 $VP_{n,i,r}$ がモデル化年に更新の対象となるストック量を超えないようにします。

$$
{sr}_{n,i,r} \geq {VP}_{n,i,r}
$$

設置年次別ストックは年 $y$ で追跡されます。発電機の場合、更新可能ストック $sr_{n,i,r}$ は耐用年数 $\tau_{n,i,r}$ に達した設置年次で構成され、経過年数 $a_y = y^* - y$（$y^*$：シミュレーション年）でマルチ年ステップにスケーリングされます：

$$
{sr}_{n,i,r} = \frac{\sum_{y\;:\; a_y \geq {\tau}_{n,i,r}} {sc}_{n,i,r,y}}{\Delta y}
$$

技術レトロフィットが有効な場合、集計されたレトロフィット量 $\sum_y VC_{n,i,r_0,r_1,y}$ にも上下限を設けることができます。これらの上下限は転換元技術 $r_0$ から転換先技術 $r_1$ へのレトロフィットフロー $VC_{n,i,r_0,r_1,y}$ に直接適用されます。発電機の場合：

$$
{\mu^{\min}}_{n,i,r_0,r_1}
\leq \sum_y {VC}_{n,i,r_0,r_1,y}
\leq {\mu^{\max}}_{n,i,r_0,r_1}
$$

リンクと蓄電についても同様です。

<details markdown="1">
<summary><b>技術ストックで使用するパラメータ</b></summary>

| パラメータ | シンボル | 説明 | 単位 |
| ---------- | -------- | ---- | ---- |
| $sc_{n,i,r,y}$ | `gssc(N,I,R,Y)` | 設置年次別の発電機存続ストック | GW |
| $sc_{l,y}$ | `fssc(L,Y)` | 設置年次別のリンク存続ストック | GW |
| $sc_{n,i,s,y}$ | `essc(N,I,S,Y)` | 設置年次別の蓄電存続ストック | GWh |
| $sr_{n,i,r}$ | `gssr(N,I,R)` | 更新可能発電機ストック | GW |
| $sr_{l}$ | `fssr(L)` | 更新可能リンクストック | GW |
| $sr_{n,i,s}$ | `essr(N,I,S)` | 更新可能蓄電ストック | GWh |
| $\mu^{\min}_{n,i,r_0,r_1}$ | `gcmn(N,I,R0,R1)` | 発電機レトロフィット量の下限 | GW |
| $\mu^{\max}_{n,i,r_0,r_1}$ | `gcmx(N,I,R0,R1)` | 発電機レトロフィット量の上限 | GW |
| $\mu^{\min}_{l_0,l_1}$ | `fcmn(L0,L1)` | リンクレトロフィット量の下限 | GW |
| $\mu^{\max}_{l_0,l_1}$ | `fcmx(L0,L1)` | リンクレトロフィット量の上限 | GW |
| $\mu^{\min}_{n,i,s_0,s_1}$ | `ecmn(N,I,S0,S1)` | 蓄電レトロフィット量の下限 | GWh |
| $\mu^{\max}_{n,i,s_0,s_1}$ | `ecmx(N,I,S0,S1)` | 蓄電レトロフィット量の上限 | GWh |
| $\tau_{n,i,r}$ | `glt(N,I,R)` | 発電機の耐用年数 | 年 |
| $\tau_{l}$ | `flt(L)` | リンクの耐用年数 | 年 |
| $\tau_{n,i,s}$ | `elt(N,I,S)` | 蓄電の耐用年数 | 年 |
| $\Delta y$ | `t_int` | シミュレーション年の間隔 | 年 |

</details>

<details markdown="1">
<summary><b>技術ストックで使用する決定変数</b></summary>

| 変数 | シンボル | 説明 | 単位 |
| ---- | -------- | ---- | ---- |
| $VS_{n,i,r}$ | `VGS(N,I,R)` | 発電機の設置済み容量 | GW |
| $VR_{n,i,r}$ | `VGR(N,I,R)` | 発電機の新設容量 | GW/yr |
| $VC_{n,i,r_0,r_1,y}$ | `VGC(N,I,R0,R1,Y)` | 発電機のレトロフィット量 | GW |
| $VP_{n,i,r}$ | `VGP(N,I,R)` | 発電機の更新容量 | GW/yr |
| $\delta^{\text{stock}}_{n,i,r}$ | `RES_GSCB(N,I,R)` | 発電機ストックバランスのスラック | GW |
| $VS_{l}$ | `VFS(L)` | リンクの設置済み容量 | GW |
| $VR_{l}$ | `VFR(L)` | リンクの新設容量 | GW/yr |
| $VC_{l_0,l_1,y}$ | `VFC(L0,L1,Y)` | リンクのレトロフィット量 | GW |
| $VP_{l}$ | `VFP(L)` | リンクの更新容量 | GW/yr |
| $\delta^{\text{stock}}_{l}$ | `RES_FSCB(L)` | リンクストックバランスのスラック | GW |
| $VS_{n,i,s}$ | `VES(N,I,S)` | 蓄電の設置済みエネルギー容量 | GWh |
| $VR_{n,i,s}$ | `VER(N,I,S)` | 蓄電の新設容量 | GWh/yr |
| $VC_{n,i,s_0,s_1,y}$ | `VEC(N,I,S0,S1,Y)` | 蓄電のレトロフィット量 | GWh |
| $VP_{n,i,s}$ | `VEP(N,I,S)` | 蓄電の更新容量 | GWh/yr |
| $\delta^{\text{stock}}_{n,i,s}$ | `RES_ESCB(N,I,S)` | 蓄電ストックバランスのスラック | GWh |

</details>

## エネルギー供給

エネルギー供給 $VP_{n,i,k}$ は発電機出力から導出される中間的な集計変数であり、排出量方程式に入力され、資源上下限制約を受けます。

年間エネルギー供給 $VP_{n,i,k}$ は、発電機出力 $VX_{n,i,r,t}$ を係数 $\eta_{n,i,r,k}$ でエネルギー種別 $k$ に変換し、時間スライスの重みで積算して得られます。

$$
{VP}_{n, i, k} = \sum_{r,t} {\omega} \cdot {\eta}_{n, i, r, k} \cdot {VX}_{n, i, r, t}
$$

エネルギー供給 $VP_{n,i,k}$ は、集約された地域-セクターグループとエネルギーグループに対して最小/最大上下限または固定値で制約されます。集約集合 $ME$ と $MK$ により、これらの制約を基本のノード-セクター・エネルギー種別よりも粗い粒度で課すことができます。

$$
{p^{\min}}_{m_e,k_m} \leq \sum_{(n,i)\in ME_{m_e}}\sum_{k\in MK_{k_m}} {VP}_{n,i,k}
\leq {p^{\max}}_{m_e,k_m}
$$

$$
{p^{\text{fix}}}_{m_e,k_m} = \sum_{(n,i)\in ME_{m_e}}\sum_{k\in MK_{k_m}} {VP}_{n,i,k}
$$

<details markdown="1">
<summary><b>エネルギー供給で使用するパラメータ</b></summary>

| パラメータ | シンボル | 説明 | 単位 |
| ---------- | -------- | ---- | ---- |
| $\eta_{n,i,r,k}$ | `geta(N,I,R,K)` | 発電機出力単位あたりのエネルギーキャリア出力（例：熱量係数）；主要出力キャリアでは 1、副次キャリア（CHP 熱出力・電解槽水素など）では異なる値をとる | TWh/GWh |
| $\omega$ | `wt(T)` | 時間スライスの重み | 時間 |
| $p^{\min}_{m_e,k_m}$ | `pmin(ME,MK)` | 集約グループの年間エネルギー供給下限 | TWh |
| $p^{\max}_{m_e,k_m}$ | `pmax(ME,MK)` | 集約グループの年間エネルギー供給上限 | TWh |
| $p^{\text{fix}}_{m_e,k_m}$ | `pfix(ME,MK)` | 集約グループの年間エネルギー供給固定値 | TWh |

</details>

<details markdown="1">
<summary><b>エネルギー供給で使用する決定変数</b></summary>

| 変数 | シンボル | 説明 | 単位 |
| ---- | -------- | ---- | ---- |
| $VP_{n,i,k}$ | `VP(N,I,K)` | エネルギー種別 $k$ の年間エネルギー供給 | TWh |
| $VX_{n,i,r,t}$ | `VG(N,I,R,T)` | 発電機の電力出力 | GW |

</details>

## 排出量

排出量 $VQ_{n,i,g}$ はエネルギー供給から導出され、2 つの補完的なメカニズムで制御できます：**排出上限**（ここで定義するハード制約）と**排出税**（目的関数に追加する価格シグナル。目的関数セクション参照）。どちらか一方または両方を同時に有効にできます。

排出量は排出係数 $\epsilon_{n,i,k,g}$ を用いて計算されるため、発電出力から直接ではなく年間エネルギー供給からの変換後集計量として表現されます。

$$
{VQ}_{n,i,g} = \sum_{k} {{\epsilon}_{n,i,k,g}}\cdot {VP}_{n,i,k}
$$

排出上限は、集約された地域とガスグループに対して排出量 $VQ_{n,i,g}$ の合計を制限します。エネルギー供給制約と同様に、集約集合が上限を課す粒度を決定します。

$$
\sum_{(n,i)\in MQ_{m_q}}\sum_{g\in MG_{g_m}} {VQ}_{n,i,g} \leq {q^{\max}}_{m_q,g_m}
$$

<details markdown="1">
<summary><b>排出量で使用するパラメータ</b></summary>

| パラメータ | シンボル | 説明 | 単位 |
| ---------- | -------- | ---- | ---- |
| $\epsilon_{n,i,k,g}$ | `gas(N,I,K,G)` | エネルギー種別 $k$ およびガス $g$ の排出係数 | 例：MtCO₂/TWh |
| $q^{\max}_{m_q,g_m}$ | `qmax(MQ,MG)` | 集約グループの年間排出上限 | 例：MtCO₂ |

</details>

<details markdown="1">
<summary><b>排出量で使用する決定変数</b></summary>

| 変数 | シンボル | 説明 | 単位 |
| ---- | -------- | ---- | ---- |
| $VQ_{n,i,g}$ | `VQ(N,I,G)` | ガス種別 $g$ の年間排出量 | 例：MtCO₂ |
| $VP_{n,i,k}$ | `VP(N,I,K)` | エネルギー種別 $k$ の年間エネルギー供給 | TWh |

</details>

[^1]: [Neumann, F., Hagenmeyer, V. & Brown, T. Assessments of linear power flow and transmission loss approximations in coordinated capacity expansion problems. Appl. Energy 314, 118859 (2022).](https://doi.org/10.1016/j.apenergy.2022.118859)
