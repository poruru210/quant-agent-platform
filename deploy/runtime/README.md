# Nautilus Runtime Model

このディレクトリは、`pi-ctl` 上の Hermes runtime から、`pi-research` / `pi-live` で `NautilusTrader` を実行するモデルを定義する。

重要:

- `NautilusTrader` は compose の常駐 service として固定しない
- Hermes runtime は `pi-ctl` に置く
- research 系 skill も live 系 skill も `SSH transport` で remote node に到達する
- remote ノード上では `docker` などで run または instance 単位に隔離する
- 実 IF の主語は `docker compose up` や直接スクリプトではなく `nautilus-backtest` / `nautilus-replay` / `shadow-start` / `shadow-stop` である
- run result の truth は execution node 上の `parquet` とし、`DuckDB` は node-local query layer として使う

## 1. なぜ job model か

`research` 側では以下が起きる。

- strategy ごとに設定が異なる
- 同時に複数 backtest / replay を走らせる
- 終了後に artifact を回収して消したい
- 失敗 run だけを再実行したい

`live validation` 側では以下が起きる。

- strategy instance 単位で起動 / 停止 / rollback したい
- spool と report を instance ごとに隔離したい
- research 系の重い処理と混ぜたくない

このため、compose より `skill + SSH + per-container lifecycle` のほうが IF に合う。

## 2. ノードごとの運用単位

### 2.1 `pi-research`: research execution

Hermes `research_engineer` が `pi-ctl` から呼ぶ skill:

- `nautilus-backtest`
- `nautilus-replay`
- `run_feature_job`
- `run_simulation`

実行単位:

- 1 skill call = 1 remote isolated job
- container は `--rm` で終了後に消す
- artifact は run ID 単位で host に残す

### 2.2 `pi-live`: live validation

Hermes `shadow_operator` が `pi-ctl` から呼ぶ skill:

- `shadow-start`
- `shadow-stop`
- `shadow-status`
- `shadow-tail-logs`

実行単位:

- 1 strategy instance = 1 named container
- container は long-lived でもよいが、compose ではなく skill 実装が lifecycle を持つ
- spool / report / logs は instance ID 単位で host に残す

## 3. ホストディレクトリ

初期案:

- `pi-research`
  - `/srv/quant/research/workspaces`
  - `/srv/quant/research/runs`
  - `/srv/quant/research/cache`
  - `/srv/quant/research/duckdb`
- `pi-live`
  - `/srv/quant/live/instances`
  - `/srv/quant/live/spool`
  - `/srv/quant/live/reports`
  - `/srv/quant/live/logs`
  - `/srv/quant/live/duckdb`

原則:

- code / config / artifacts / spool を分ける
- run ID または instance ID でディレクトリを切る
- live spool は `pi-ctl` へ非同期転送できる前提で置く
- raw result は execution node の `parquet` に残す
- `DuckDB` は `pi-research` / `pi-live` の両方で query layer として使う
- live の実 tick / 実 spread / event も `parquet` へ落とす

## 4. Skill Contract

### 4.1 `nautilus-backtest`

最低限の入力:

- `strategy_id`
- `revision`
- `config_path`
- `run_id`

最低限の出力:

- `container_name`
- `artifact_dir`
- `stdout_log`
- `exit_code`

内部実装イメージ:

```bash
ssh pi-research "docker run --rm \
  --name bt-${STRATEGY_ID}-${RUN_ID} \
  -v /srv/quant/research/workspaces/${STRATEGY_ID}:/workspace:ro \
  -v /srv/quant/research/runs/${RUN_ID}:/artifacts \
  ${NAUTILUS_RESEARCH_IMAGE} \
  nautilus-backtest --config '${CONFIG_PATH}'"
```

注意:

- 上の `ssh` と remote command は skill の内側の実装例であり、運用 IF ではない
- `python /path/to/script.py` のような直接スクリプト呼び出しを主契約にしない

### 4.2 `nautilus-replay`

最低限の入力:

- `strategy_id`
- `revision`
- `config_path`
- `run_id`

最低限の出力:

- `container_name`
- `artifact_dir`
- `stdout_log`
- `exit_code`

### 4.3 `shadow-start`

最低限の入力:

- `strategy_id`
- `instance_id`
- `revision`
- `config_path`

最低限の出力:

- `container_name`
- `spool_dir`
- `report_dir`
- `pid` または container status

内部実装イメージ:

```bash
ssh pi-live "docker run -d \
  --name shadow-${STRATEGY_ID}-${INSTANCE_ID} \
  -v /srv/quant/live/instances/${INSTANCE_ID}:/runtime \
  -v /srv/quant/live/spool/${INSTANCE_ID}:/spool \
  -v /srv/quant/live/reports/${INSTANCE_ID}:/reports \
  ${NAUTILUS_LIVE_IMAGE} \
  shadow-start --config '${CONFIG_PATH}'"
```

### 4.4 `shadow-stop`

最低限の入力:

- `instance_id`

最低限の出力:

- `container_name`
- `stop_result`

## 5. Hermes との接続

`Paperclip` から直接 Nautilus を呼ばない。  
呼び出しの流れは以下とする。

1. `Paperclip` が `hermes_local` adapter を起動する
2. Hermes profile が issue / documents / comments を読む
3. role に許可された Hermes skill だけを呼ぶ
4. research 系なら `SSH transport` で `pi-research` に到達し、live 系なら `SSH transport` で `pi-live` に到達する
5. remote isolation として container を起動する
6. artifact / report / summary を `Paperclip issue documents` と comment に返す
7. 必要なら `DuckDB` query で research / live の結果を確認してから `LLM-WIKI` に知見を戻す

重要:

- `Quant Research Engineer` は live skill を持たない
- `Shadow Trading Operator` は research skill を持たない
- security boundary は prompt ではなく skill availability と remote isolation で担保する

## 6. Result Storage

初期方針:

- run ごとの truth は `/srv/quant/research/runs/<run_id>/`
- 最低限 `summary.json` と `metrics.parquet` を残す
- `orders.parquet` / `fills.parquet` / `positions.parquet` は raw analysis 用に残す
- `DuckDB` は `/srv/quant/research/duckdb/` と `/srv/quant/live/duckdb/` に置き、query layer とする
- `pi-live` では `ticks.parquet` / `spreads.parquet` / `events.parquet` を残す
- `LLM-WIKI` には raw ではなく Hermes が要約した lesson / rationale / next step を戻す
- `DuckDB` は常駐 service ではなく node-local utility container として扱ってよい

詳細は [research-result-storage-contract.md](D:/GitHub/quant-agent-platform/docs/research-result-storage-contract.md) を参照。

## 7. Env サンプル

- [research-job.env.example](D:/GitHub/quant-agent-platform/deploy/runtime/research-job.env.example)
- [live-instance.env.example](D:/GitHub/quant-agent-platform/deploy/runtime/live-instance.env.example)

## 8. 未決事項

- 初期 transport は `ssh + password authentication` でよいか
- run / instance metadata をどこまで `Paperclip issue documents` に正規化するか
- `Hindsight` に返す run summary の粒度
- live risk guard を skill 実装に含めるか、別 process にするか

## 9. 初期提案

- SSH は LAN 内のみで使い、外部公開しない
- 初期は password authentication でよい
- Hermes runtime は `pi-ctl` に置く
- `pi-research` と `pi-live` に remote 実行用の専用ユーザーを置く
- `Paperclip issue documents` には固定 key の JSON を返す
- 詳細ログは document 本文へ詰め込まず、artifact path / attachment ref に逃がす
- run result は `parquet` と `DuckDB` で確認し、wiki には抽象化結果だけを戻す

詳細は [hermes-nautilus-skill-contract.md](D:/GitHub/quant-agent-platform/docs/hermes-nautilus-skill-contract.md) を参照。
