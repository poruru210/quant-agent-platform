# Execution Result Storage Contract

本書は、`pi-research` と `pi-live` で実行される `NautilusTrader` の結果を、どのように保存し、どこまでを `Hermes` と `LLM-WIKI` に渡すかを定義する。

---

## 1. 原則

- raw result は execution node に残す
- run ごとの source of truth は file artifact とする
- 初期保存形式は `parquet` と `json` を優先する
- `DuckDB` は primary store ではなく query layer として使う
- `LLM-WIKI` には raw event log を直接入れず、`Hermes` が抽象化した lesson / rationale / comparison を戻す
- `DuckDB` は `pi-research` / `pi-live` の両方で使ってよい
- `DuckDB` は常駐 service ではなく、node-local query container として扱ってよい
- parquet 列や DuckDB view は hard IF ではない
- それらは必要時に Hermes が決め、後から改善できる余地を残す

---

## 2. 保存レイヤ

### 2.1 raw artifact layer

`NautilusTrader` 実行の直接成果物を run / instance 単位で保存する。

例:

- `summary.json`
- `metrics.parquet`
- `orders.parquet`
- `fills.parquet`
- `positions.parquet`
- `events.parquet`
- `logs/`

### 2.2 query layer

複数 run / instance を横断確認するために `DuckDB` を置く。

ここでやること:

- `parquet` 群への横断クエリ
- run 間比較
- shadow instance 間比較
- regime / venue / instrument ごとの再集計
- Hermes が wiki 化前に確認する数値の再計算

重要:

- まず `parquet` を truth とし、`DuckDB` はそれを読む
- 初期は `DuckDB` へ全件を import しなくてもよい
- `read_parquet(...)` や view を中心に使い、必要になってから materialize する

### 2.3 knowledge layer

`LLM-WIKI` は knowledge layer として使う。

ここで残すもの:

- 何が効いたか
- どの regime で崩れたか
- 次に試すべきこと
- どの validation を通したか
- incident から得た一般化 lesson

ここで残さないもの:

- 全 fills
- 全 orders
- 全 event log
- 単発 run の巨大 csv/parquet 本体

---

## 3. 初期ディレクトリ規約

`pi-research` 上の初期配置:

```text
/srv/quant/research/
├─ workspaces/
├─ runs/
│  └─ <run_id>/
│     ├─ summary.json
│     ├─ metrics.parquet
│     ├─ orders.parquet
│     ├─ fills.parquet
│     ├─ positions.parquet
│     ├─ events.parquet
│     ├─ exports/
│     └─ logs/
├─ duckdb/
│  ├─ research.duckdb
│  └─ views/
└─ catalogs/
```

`pi-live` 上の初期配置:

```text
/srv/quant/live/
├─ instances/
│  └─ <instance_id>/
│     ├─ summary.json
│     ├─ ticks.parquet
│     ├─ spreads.parquet
│     ├─ orders.parquet
│     ├─ fills.parquet
│     ├─ events.parquet
│     ├─ exports/
│     └─ logs/
├─ spool/
├─ reports/
└─ duckdb/
   ├─ live.duckdb
   └─ views/
```

原則:

- `run_id` は `Paperclip issue documents` の `execution_request.run_id` と一致させる
- 1 run 1 directory を守る
- live は 1 instance 1 directory を守る
- `exports/` は Hermes や人間向けの派生物を置く
- 大容量 raw と要約 export を混ぜない

---

## 4. 最小 artifact contract

初期実装で最低限そろえる成果物の例:

- `summary.json`
  - 実行メタデータと headline metrics
- `metrics.parquet`
  - strategy / instrument / venue / period ごとの集計表
- `orders.parquet`
- `fills.parquet`
- `positions.parquet`
- `logs/nautilus.log`

live 側で最低限そろえたい成果物の例:

- `summary.json`
- `ticks.parquet`
- `spreads.parquet`
- `orders.parquet`
- `fills.parquet`
- `events.parquet`
- `logs/nautilus.log`

`summary.json` の最小形:

```json
{
  "run_id": "bt-20260418-001",
  "strategy_id": "mean-reversion-001",
  "mode": "backtest",
  "revision": "gitsha-or-tag",
  "node": "pi-research",
  "status": "succeeded",
  "started_at": "2026-04-18T10:30:00+09:00",
  "finished_at": "2026-04-18T10:42:11+09:00",
  "headline_metrics": {
    "sharpe": 1.42,
    "max_drawdown": -0.083,
    "turnover": 5.8
  }
}
```

---

## 5. DuckDB の使い方

### 5.1 初期方針

- `DuckDB` は `pi-research` と `pi-live` の両方に置く
- node ごとに別 DB ファイルでよい
- `DuckDB` 自体は docker container で使ってよい
- Hermes は必要なときだけ `DuckDB` query を実行する
- query 結果は `summary export` として `exports/` に落としてよい

### 5.2 初期ユースケース

- 同一 strategy の revision 比較
- train / validation / shadow 候補 window 比較
- venue ごとの fill / cost / slippage 集計
- drawdown の regime 別比較
- live の実 tick / 実 spread / fill quality 確認
- shadow instance 間比較

### 5.3 位置づけ

`DuckDB` は次のどちらでもよい。

- `parquet` を直接読む query engine
- 比較用の軽い index / derived table store

初期は前者を推奨する。

理由:

- import 手順を増やさずに始められる
- run 完了直後にそのまま確認できる
- truth を `parquet` に残せる

### 5.4 事前固定しないこと

- `metrics.parquet` の詳細列
- `ticks.parquet` / `spreads.parquet` の詳細列
- view 命名規約
- export の細かな粒度

これらは通常の事前設計対象ではなく、Hermes が issue 文脈と改善ループに応じて決める余地を残す。

---

## 6. Hermes との接続

Hermes skill は、remote 実行後に最低限以下を扱えるようにする。

- `artifact_ref`
- `summary.json`
- 必要に応じた `DuckDB` query result

Hermes がやること:

1. `execution_result` / `validation_report` / `shadow_report` を `Paperclip issue documents` に返す
2. 必要なら `DuckDB` で追加確認する
3. wiki に戻す価値のある lesson だけを抽象化する
4. `LLM-WIKI` へ summary / rationale / next step を反映する

Hermes がやらないこと:

- raw `parquet` を wiki へ貼る
- 巨大 event log を issue document 本文へ流し込む

---

## 7. Paperclip へ返す ref

`Paperclip issue documents` へ返すべき最小 ref:

- `artifact_ref`
- `summary_ref`
- `report_ref`

必要なら追加:

- `duckdb_ref`
  - 例: `duckdb:///srv/quant/research/duckdb/research.duckdb`
  - 例: `duckdb:///srv/quant/live/duckdb/live.duckdb`
- `query_export_ref`
  - 例: `/srv/quant/research/runs/<run_id>/exports/regime-comparison.parquet`

重要:

- `duckdb_ref` は human-facing URL でなく内部 ref として扱う
- `Paperclip` は DB 自体を読む必要はなく、Hermes が確認できれば十分である

---

## 8. LLM-WIKI との接続

`LLM-WIKI` に戻す前に、Hermes は少なくとも以下を確認する。

- `summary.json`
- `validation_report` または `shadow_report`
- 必要な `DuckDB` query 結果

wiki に戻す最小単位:

- hypothesis outcome
- validation lesson
- metric interpretation
- next experiment proposal

wiki に戻すときの ref:

- `run_id`
- `issue_id`
- `revision`
- `artifact_ref`

---

## 9. 初期判断

- `pi-research` と `pi-live` の両方に `DuckDB` を追加するのは妥当
- truth は `parquet` ベースに置く
- `DuckDB` は Hermes の確認用 query layer に寄せる
- live の実 tick / spread も `parquet` へ落としておく価値が高い
- `LLM-WIKI` は知識化された結果だけを持つ

---

## 10. 未決事項

- `metrics.parquet` / `ticks.parquet` / `spreads.parquet` の列標準を本当に固定対象にするか
- `DuckDB` view を strategy ごとに持つか、run / instance 横断 view に寄せるかを事前決定すべきか
- `pi-live` の query export を `pi-research` 側へどこまで同期するか
