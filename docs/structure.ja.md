---
icon: material/sitemap
---

# モデル構造

!!! note "PSMpy v0.1 — GAMS 等価の再構築"
    PSMpy は GAMS `PowerSystemModel` の Python 再構築です。以下の solving 構造
    （ノード・セクター・発電機 / リンク / 蓄電・タイムスライス・stock）は GAMS と
    等価に保たれ、authoring では固定要素型 **Source / Conversion / Transmission /
    Storage / Sink** にデータ駆動で対応します（ADR-0003 / ADR-0008）。

PSMpy v0.1 はエネルギーシステムを**ノード**（$n \in N$）と**セクター**（$i \in I$）のネットワークとして表現します。固定された**要素型** — **Source**・**Conversion**・**Transmission**・**Storage**・**Sink**（需要）— がノード-セクター対に接続・供給します（ADR-0008）。これらは方程式が使う GAMS の solving ドメインに対応します：発電機（$r \in R$, `g*`）・リンク（$l \in L$, `f*`）・蓄電（$s \in S$, `e*`）。

## 要素型

!!! note "authoring 名と GAMS/solving 名"
    authoring・reporting 層は以下の**要素型**名を用い、solving 層（および [式体系](equation.md) ページ）は GAMS のドメイン/記号 `R`/`L`/`S`・`g*`/`f*`/`e*` を 1:1 で保持します。対応：**Source** ↔ 発電機 `R`/`g*`；**Transmission**・**Conversion** ↔ リンク `L`/`f*`；**Storage** ↔ `S`/`e*`；**Sink** ↔ 需要 `dmd`。

### Source — GAMS 発電機（$r \in R$, `g*`）

**Source** はノード-セクター対 $(n,i)$ 内でエネルギーを生産します。出力 $VX_{n,i,r,t}$ は設置済み容量 $VS_{n,i,r}$ と時変の利用可能率係数で上下限が設けられます。Source 出力はノード電力バランスに入り、キャリア係数 $\eta_{n,i,r,k}$（`geta`）を介してエネルギー供給変数 $VP_{n,i,k}$（排出量計算に使用）にも反映されます。*消費側* Source（出力 $\le 0$）は燃料の引き込みを表します。

### Transmission・Conversion — GAMS リンク（$l \in L$, `f*`）

GAMS の *リンク* ドメインは、1つの solving クラス上に **2つ**の PSMpy 要素型として authoring されます：**Transmission**（同一キャリアの双方向移送）と **Conversion**（符号付き多出力デバイス、例：電解槽）。順方向 $VX^+_{l,t}$・逆方向 $VX^-_{l,t}$ を別々に扱うことで非対称効率と方向制約を表現し、ポート係数（`feta`・`kff`/`kbf`）が効率／多出力構造を担います。

### Storage — GAMS 蓄電（$s \in S$, `e*`）

**Storage** は充電状態変数 $VE_{n,i,s,t}$ を介して時間スライスをまたぐ運転を連動させます。充放電電力 $VX_{n,i,s,t}$ はノード電力バランスに現れ、エネルギー容量 $VS_{n,i,s}$ は別途追跡されます。容量比率 $\chi$ が関連リンク（ポンプ・タービン）の電力容量と貯水池のエネルギー容量を結びつけます。

### Sink — 境界需要（`dmd`）

**Sink** はバス上の外生需要で、投資対象の資産ではありません。サービス／エネルギー需要は上記の要素によりノード電力バランスを通じて満たされます。

## 時間構造

| 次元 | シンボル | 役割 |
|------|--------|------|
| シミュレーション年 | $y \in \text{YEAR}(Y)$ | 投資・廃棄決定、コスト年率化 |
| 時間スライス | $t \in T$ | 年内ディスパッチ、蓄電ダイナミクス |

時間スライスの重み $\omega$（時間）は電力を年間エネルギーに積算するために用います。粒度 $\Delta t$（時間）はランピング制約や自己放電といった速度依存の制約に使用します。

## 技術ストック

**Source**・**Transmission**/**Conversion**・**Storage** は共通のストック方程式を持ちます — 単一の汎用 stock ブロックを要素型ごとにインスタンス化して一度だけ生成します（ADR-0001）。設置済み容量 $VS$ は新設投資 $VR$、更新 $VP$、存続設置年次別ストック $sc$（外生パラメータ）、レトロフィットフロー $VC$、ストックバランスのスラック $\delta^{\text{stock}}$ から構成されます：

$$VS = (VR + VP) \cdot \Delta y + \sum_y \!\left( sc_y + \sum_{\text{流入}} VC - \sum_{\text{流出}} VC \right) - \delta^{\text{stock}}$$

| 項 | 意味 |
|----|------|
| $VR$ | シミュレーションステップあたりの新設容量 |
| $VP$ | 耐用年数超過ストックの更新容量 |
| $sc_y$ | 存続設置年次別ストック（歴史的記録から事前計算） |
| $VC$ | レトロフィットフロー（内生的技術転換） |
| $\delta^{\text{stock}}$ | ストックバランスのスラック（目的関数でペナルティ） |

## 目的関数

$$VTC = C^{\text{CAPEX}} + C^{\text{Fixed OPEX}} + C^{\text{Variable OPEX}} + C^{\text{Emission tax}} \to \min$$

CAPEX（$C^{\text{CAPEX}}$）は新設・レトロフィット・更新のコストを技術固有の割引率と耐用年数で年率化したものです。固定 O&M は設置済み容量に比例し、変動 O&M は出力に比例します。排出税は排出量単位で課されます。

## 中心的制約

**ノード電力バランス**がすべての要素を結びつける制約です。各ノード-セクター対 $(n,i)$ と時間スライスグループ $m_t \in MT$ に対し、Source 出力・Storage 放電・正味 Transmission/Conversion フローの合計が外生需要 $d_{n,i,MT}$（＋処分スラック $\delta_{n,i,MT}$）と等しくなる必要があります：

$$\sum_r \sum_{t \in MT} VX_{n,i,r,t} + \sum_s \sum_{t \in MT} VX_{n,i,s,t} + \text{（正味リンクフロー）} = d_{n,i,MT} + \delta_{n,i,MT}$$

すべての制約の完全な数学的定式化は [式体系](equation.md) ページを参照してください。
