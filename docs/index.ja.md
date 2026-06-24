---
icon: material/home
---

# PSMpy

**PSMpy v0.5** は GAMS の `PowerSystemModel` を [GAMSpy](https://gamspy.readthedocs.io/)
モデルとして再構築した Python 実装です — エネルギーシステムの容量拡張・運用最適化
を行う多年度線形計画モデル。エネルギーシステムは固定された物理要素型 — **Source /
Conversion / Transmission / Storage**（＋需要の **Sink**）— で表現し、空間スコープ・
時間解像度・セクタースコープはラン毎に設定できます。

## 主要な考え方

- **データ駆動の authoring** — 入力はコードではなく検証済みテーブル＋YAML。
  コンポーネントクラスはメモリ上の IR（ADR-0003）。
- **GAMS 等価の数式・Python 的な構造** — ドメイン/シンボル名は GAMS と 1:1。
  GAMS が3回繰り返す約30本のライフサイクル/投資方程式は、単一の汎用 stock ブロックを
  ×3 インスタンス化して一度だけ生成（ADR-0001）。
- **構造化バス** `(carrier, segment)` を authoring/reporting で使用。GAMS の `I`
  ラベルはコンパイル済みシンボルにのみ存在（ADR-0004）。
- **スコープ＝非インスタンス化** — スコープ外の要素は生成しない（eps 容量も
  fix-to-zero も無し）。スコープ軸は設定＋データであり、コード分岐ではない（ADR-0012）。
- **レポート** は純 Python の Track P コア（`tools/pyreport`）を再利用 — xarray/
  pandas、solver 不要。

## はじめに

| ページ | 内容 |
| --- | --- |
| [ユーザーガイド](user_guide.md) | インストール · データセット作成 · 検証 · compile · 求解 · レポート |

設計の*理由*は `CLAUDE.md`、`docs/adr/` の決定記録、`docs/spec/` の仕様書を参照。
実装の現状は `docs/STATUS.md`、マイルストーン計画は `docs/roadmap.md`。

> 最初の縦断スライス：**global2050 · Japan · `power`**（ADR-0007）。
> authoring/solving/reporting 層と需要セクター toy ラダーは構築済みで、
> 実 Japan ラン は HPC ゲート待ち。
