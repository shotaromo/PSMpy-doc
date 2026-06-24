# PSMpy ユーザーガイド

PSMpy を**動かすための実践的な how-to** です。`docs/spec/` と `docs/adr/` は
モデルの設計理由（*なぜ*）を説明しますが、本書は*どう*インストールし、入力
データセットを作成・検証・コンパイル・求解・レポートするかを示します。

> 現状のスコープ：最初の縦断スライスは **global2050 · Japan · `power`**
> （ADR-0007）。authoring/solving/reporting 層と toy ラダーは `main` 上にあり、
> 実 Japan ラン は HPC ゲート待ち（`docs/STATUS.md`）。統一 `psmpy` CLI
> （`validate` / `sweep` / `extract` / `run`）が shell ワークフローを担い、
> 下記の Python API はそのインプロセス版（CLI が内部で呼ぶもの）です。

---

## 1. インストールと環境

PSMpy は [`uv`](https://docs.astral.sh/uv/) ＋ リポジトリ毎の `.venv` を使います
（HPC ではユーザーローカル＝ホームのみ。`CLAUDE.md` 参照）。

```bash
uv sync                 # コア依存 (numpy / pandas / pyarrow / pyyaml)
uv sync --extra solve   # gamspy も導入 — SOLVE（および GDX 読み込み）に必要
uv sync --extra report  # pyreport も導入 — IAMC レポート（§5）に必要
uv run pytest tests/    # テスト（solver / report テストは extra 無しで自動スキップ）
```

- **GDX 読み込みと求解には `gamspy`**（`solve` extra）が必要です。パッケージ自体
  は無償で、*求解* には既存の GAMS/solver ライセンス（HPC 等）を使います。
  authoring（load / validate / **compile**）はどちらも不要です。
- **IAMC レポートには `pyreport`**（`report` extra）が必要 — 共有 Track P
  レポーティングコア。純 pandas（solver・ライセンス不要）。§5 参照。
- すべて**ユーザーローカル**で動作（sudo 不要・システム Python 不使用）。

---

## 2. 5分でできる経路 — toy を load・validate・compile（solver不要）

最小の end-to-end フィクスチャは `tests/data/toy/`（1地域・1キャリア・2 source・
需要1つ）。**GAMS ライセンス無し**で動きます：

```python
from pathlib import Path
from psmpy.authoring import EnergyModel

model = EnergyModel.from_dataset(Path("tests/data/toy"))   # tables/ + YAML -> 検証済み IR
print(len(model.sources), "sources,", len(model.sinks), "sink(s)")

report = model.validate()          # tier-1 スキーマ + tier-2/3 要素・横断ルール
assert report.ok, [f"{e.id}: {e.message}" for e in report.errors]

sf = model.compile(time_resol="12DAY")   # IR -> SymbolFrames（年非依存シンボル）
print("compiled:", type(sf).__name__)
```

`validate()` は `ValidationReport`（`.ok`, `.errors`, `.warnings`, `.issues`,
`.summary`, `.tier_errors`）を返します。`compile()` は不透明な `SymbolFrames`
オブジェクトを返し、求解ループ（§4）へ渡します — **dict ではありません**。

---

## 3. 自前データセットの作成

入力は**コードではなくデータ**（ADR-0003）。データセットは正準テーブル
（CSV/Parquet）＋ buses/groups の YAML から成るディレクトリです。
`tests/data/toy/` を雛形に。`power` スライスのクロージャ：

| ファイル | 内容 |
| --- | --- |
| `regions.csv` | 地域 id |
| `carriers.csv` | `carrier, kind, unit`（全数値列に単位を宣言） |
| `buses.yaml` | 構造化バス `(carrier, segment)`（ADR-0004） |
| `acct_categories.csv` | 勘定カテゴリ（`K`） |
| `years.csv` | `year, is_vintage_only` |
| `source_techs.csv` / `sources.csv` | 技術カタログ＋インスタンス（catalog ⊕ instance override） |
| `source_accounting.csv` | source 毎の勘定（`geta`） |
| `dispatch_profiles.csv` | タイムスライス毎の利用可能性（任意） |
| `demand_annual.csv` / `demand_profile.csv` | Sink 需要＋形状 |
| `links*`, `storages*`, `groups`, `emission_*`, `supply_limits`, `share_constraints` | リッチなケース用（すべて任意） |

- 正準スキーマ（キー/単位/dtype）は `src/psmpy/authoring/schema.py`
  （`CANONICAL_SCHEMAS`）にあり、`docs/spec/data_schema.md` で規定されます。
- **唯一の正準ライター**は `psmpy.authoring.write.write_dataset`（R 生成器と
  GDX 抽出経路の両方が同一レイアウトを出力）。
- フラット配置でも `tables/` サブディレクトリでもロード可能。検証は
  エラー（参照切れ・`min>max`・到達不能 sink）を `AUTH-*` id で局所化します。

---

## 4. 求解（`gamspy` ＋ GAMS ライセンスが必要）

**シェルから — `psmpy run`**（実データセットの通常経路。`scripts/solve_case.py`
へ forward し、そちらが求解＋結果書き出しを担う）：

```bash
uv run --extra solve psmpy run \
    --dataset datasets/japan_power \
    --case runs/japan_power_300C \
    --years 2020:2050:5            # start:stop:step
    # --report                     # IAMC レイヤも出力（§5; --extra report が必要）
```

validate → compile → myopic ループ求解を行い、`runs/<case>/solve/<year>/*.parquet`
（求解済み level）・`state/<year>/`（carry-over）・`summary.json` を書きます。
`psmpy run --help` に solver ノブ（`--solver` / `--crossover` / `--threads` …）。
`--dry-run` は compile 後で停止（ライセンス不要）。

**Python から** — 2 経路あります。高レベルのファサード：

```python
from psmpy.runtime.loop import RunOptions

result = model.run(RunOptions(
    years=[2030, 2035],      # 求解グリッド（既定は非 vintage 年）
    start_year=2030,
    t_int=5,                 # 区間長（年）
    case_dir="runs/toy",     # 年毎の carry-over 状態の書き出し先
))
```

明示経路（一度 compile してから myopic ループ）：

```python
from psmpy.runtime.loop import run_case

result = run_case(
    model.compile(),
    years=[2030, 2035], t_int=5, start_year=2030, case_dir="runs/toy",
)
```

- `run_case` は**求解年毎に新しい GAMSpy コンテナ**を構築します。年間 carry-over
  （コホート `gsc/fsc/esc`、枯渇性台帳 `pmax_ex`）は pandas で保持され
  `runs/<case>/state/<year>/*.parquet` にダンプ（ADR-0010）。`from_year=` で任意
  の年から再開できます。
- 戻り値は `RunResult`：`.results`（`{year: SolveResult}`。求解済みの
  `vgs`/`vgr`/`vg`/… と目的値）、`.statuses`、`.reports`（post-solve CHK-3xx）、
  `.year_inf`（最初の実行不能年）、`.degraded`。
- **`gamspy`/バックエンドが無い**場合は `compile()`（§2）で綺麗に止まります。
  ライセンスが要るのは `run`/`run_case` です。ループは実行不能年を記録するだけで
  ラン全体を中断しません。

---

## 5. レポート — IAMC レイヤ（`report` extra が必要）

PSMpy はレポート計算を再実装せず、**共有 Track P レポーティングコア**（`pyreport`
パッケージ）を**再利用**します — 「one core, two adapters」（`docs/spec/
postprocess.md` §4）：`io_gdx` が GAMS 側アダプタ、`psmpy.reporting.report` が PSMpy
側。純 pandas（gamspy 不要・ライセンス不要）。`uv sync --extra report` で導入します。

**ターンキー — `psmpy run --report`。** 最短経路：求解（§4）に `--report` を付ける
だけ。求解後、結果をインメモリで共有コアに通し、wide な IAMC 表を
`runs/<case>/report/iamc/<case>.csv` に書きます：

```bash
uv run --extra solve --extra report psmpy run \
    --dataset datasets/japan_power --case runs/japan_power_300C --report
```

emit は non-fatal — `report` extra 欠如（や任意のレポートエラー）はログ出力のみで、
結果が既にディスクにある求解を中断しません。

**Python から — インプロセス。** `from_solution` がライブの `RunResult` を共有コア
経由で IAMC long フレームに変換し、`write_iamc` が wide CSV を書きます：

```python
from psmpy.runtime.loop import run_case
from psmpy.reporting import from_solution, write_iamc

sf = model.compile()
result = run_case(sf, years=[2030, 2035], t_int=5, start_year=2030, case_dir="runs/toy")

iamc = from_solution(result, model._ir(), sf)   # -> IAMC long [var, region, year, value, label, unit]
write_iamc(iamc, "runs/toy", scenario="toy")     # -> runs/toy/report/iamc/toy.csv（wide pyam レイアウト）
```

**事後 — `from_run_dir`。** 既にディスクにある求解済みケースから IAMC を再生成
（`runs/<case>/solve/<year>/*.parquet` を読み、データセットから再コンパイル — **再求解
なし・ライセンス不要**）：

```python
from psmpy.reporting import from_run_dir
iamc = from_run_dir("runs/japan_power_300C", "datasets/japan_power")
```

**現在出るもの。** データが入るファミリ：**capacity**・**capacity_addition**・
**emission**・**energy_use / carbon_capture / Primary Energy**（最後はマップ済みの
`P_*` 一次供給キャリアのみ）。`capex` は出ますが ~0（コスト params 未投入）。
`generation` は計算されますが中間量で IAMC 変数ではありません。per-slice の
**dispatch** ファミリ（supply / withdrawal / losses / curtailment）は HPC parity
ゲート待ちで保留 — parity に敏感な Phase-1b 量です（`docs/STATUS.md`）。プロット/
ダッシュボードは merged IAMC に対し `pyreport` の plot CLI を再利用（follow-up）。

> 疎結合の代替 — `psmpy.reporting.to_report` / `write_report` は生の Report スキーマ
> フレームを parquet に書き、**別の** `pyreport` プロセスが消費（`pyreport` を import
> しない）。2 リポジトリをプロセス分離したい場合のみ使用。

`model._ir()` が現状の IR アクセサ。`src/psmpy/reporting/report.py` と
`docs/spec/postprocess.md`（数量カタログ＋IAMC 出力）を参照。

---

## 6. 抽出（M0）— GAMS 入力 GDX から データセットを作る

参照 GAMS 入力 GDX から正準テーブルを生成する（`gamspy` ＋ GDX ファイルが必要）：

```bash
# 1ステップ（GDX と gamspy がある環境で）：
uv run --extra solve psmpy extract gams \
    --gdx path/to/input/300C.gdx --out datasets/japan_power --format parquet
```

HPC では読み込みを分割：計算ノードで `dump_symbols`（~352 MB GDX を読みシンボル
parquet を書く）→ どこでも `extract_dataset`（ライセンス不要）。抽出本体は
GDX 非依存で、`run_extraction` が `symbols_provider` 経由で読み込みを注入する
ため、純変換は合成シンボル dict でユニットテスト可能（実 GDX 不要）。
`docs/STATUS.md`（M0）と `docs/hpc_runbook.md` を参照。

---

## 7. コマンドリファレンス

統一 `psmpy` CLI（`uv run` を前置。`run`/`extract` は `--extra solve`、
`run --report` は `--extra report` を追加）：

| コマンド | 用途 |
| --- | --- |
| `psmpy validate <dataset>` | load ＋ 検証 tier 1–3（solver不要）。エラー時 exit 2 |
| `psmpy sweep <sweep.yaml> --dataset <dir> --out <dir>` | sweep を N ケースディレクトリに展開（solver不要） |
| `psmpy extract gams --gdx <GDX> --out <dir>` | GDX → 正準データセット（`--extra solve` 必要） |
| `psmpy run --dataset <dir> --case <dir> [--report]` | myopic 求解、任意で IAMC 出力（§4–5; ノブは `psmpy run --help`） |
| `uv run pytest tests/` | 全テスト（solver / report テストは extra 無しで自動スキップ） |
| `uv run python -m psmpy.units --dump [--out PATH]` | 正準 unit-token 契約を出力（JSON） |

`psmpy run` は `scripts/solve_case.py` へ forward します — 求解は GAMS ライセンスが
要るため、意図的に import 可能なパッケージ API の外に置いています。5カテゴリの
シナリオ設定＋オーバーレイ（M3、`docs/spec/scenario.md`）が `sweep` を駆動します。
より充実したシナリオ CLI は今後の課題です。

---

## 8. 次に読むべきもの

| 知りたいこと | 参照 |
| --- | --- |
| 全体像 / ルール | `CLAUDE.md`, `docs/adr/` |
| 実装の現状 | `docs/STATUS.md` |
| 入力テーブル＋単位 | `docs/spec/data_schema.md` |
| IR / 検証 / compile | `docs/spec/authoring.md` |
| 方程式 / 求解 | `docs/spec/solving.md` |
| シナリオ / オーバーレイ | `docs/spec/scenario.md` |
| レポート（Track P） | `docs/spec/postprocess.md` |
| 名前（GAMS ↔ authoring ↔ reporting） | `docs/naming.md` |
| 意図的な GAMS 逸脱 | `docs/spec/deviations.md` |
