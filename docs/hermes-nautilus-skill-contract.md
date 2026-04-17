# Hermes ↔ Nautilus Skill Contract

## 1. 目的

本書は、`NautilusTrader` を Hermes から利用する際の公開 IF を `skill` として定義する。

このプロジェクトで固定したいのは以下である。

- 上位の公開 IF は `Hermes skill`
- remote ノードへの到達手段は `SSH transport`
- remote 実行の隔離は `docker` を初期値とする
- run result の truth は execution node 上の `parquet` artifact とし、`DuckDB` は node-local query layer として使う

重要:

- `ssh` は公開 IF ではない
- `docker run` も公開 IF ではない
- 直接スクリプトを叩くことを運用契約にしない

## 2. 層の分離

```text
Paperclip issue / document
  -> Hermes profile on pi-ctl
    -> Hermes skill
      -> SSH transport to pi-research
      -> or SSH transport to pi-live
        -> remote isolation (docker / microvm)
          -> NautilusTrader job / instance
```

役割:

- `Hermes skill`
  - role が使ってよい論理機能
- `SSH transport`
  - `pi-ctl` から `pi-research` / `pi-live` に命令を届ける
- `remote isolation`
  - 並列実行と安全境界を担保する

## 3. 初期採用方針

### 3.1 transport

- research 系 / live 系のどちらも `SSH`
- 接続元は `pi-ctl` 上の Hermes runtime
- 接続先は `pi-research` または `pi-live`

理由:

- `hermes_local` の実 IF と素直に整合する
- queue や常駐 API を増やさずに始められる
- research / live の実行先が対称になる
- execution node を Hermes host と分離できる

### 3.2 remote isolation

- 初期は `docker`
- 将来的に必要なら `microvm` へ差し替える

理由:

- backtest / replay の並列実行を run 単位で分離できる
- live validation を instance 単位で分離できる
- host 汚染を抑えやすい

## 4. Skill 一覧

### 4.1 Research 系

- `nautilus-backtest`
- `nautilus-replay`
- `feature-generation`
- `simulation-run`

利用 role:

- `Quant Research Engineer`

### 4.2 Live 系

- `shadow-start`
- `shadow-stop`
- `shadow-status`
- `shadow-tail-logs`
- `shadow-collect-report`
- `shadow-rollback`

利用 role:

- `Shadow Trading Operator`

## 5. Skill Contract

### 5.1 `nautilus-backtest`

入力:

- `strategy_id`
- `revision`
- `config_ref`
- `run_id`

出力:

- `run_id`
- `container_name`
- `artifact_ref`
- `report_ref`
- `exit_status`

期待動作:

- remote の `pi-research` へ `SSH transport` で到達する
- run ID ごとに独立した isolation を作る
- 複数同時実行を許可する
- 完了後は artifact を `Paperclip issue documents` に返せる形へ正規化する

### 5.2 `nautilus-replay`

入力:

- `strategy_id`
- `revision`
- `config_ref`
- `run_id`

出力:

- `run_id`
- `container_name`
- `artifact_ref`
- `report_ref`
- `exit_status`

### 5.3 `shadow-start`

入力:

- `strategy_id`
- `revision`
- `config_ref`
- `instance_id`

出力:

- `instance_id`
- `container_name`
- `spool_ref`
- `status`

期待動作:

- remote の `pi-live` へ `SSH transport` で到達する
- instance ID ごとに独立した isolation を作る
- 同名 instance の二重起動を避ける

### 5.4 `shadow-stop`

入力:

- `instance_id`

出力:

- `instance_id`
- `status`
- `report_ref`

## 6. Security Boundary

- `Quant Research Engineer` は live 系 skill を持たない
- `Shadow Trading Operator` は research 系 skill を持たない
- 境界は prompt ではなく skill 配布と SSH 到達先で担保する

## 7. 並列実行

この設計では、並列性は skill 自体ではなく remote isolation で確保する。

- `nautilus-backtest`
  - run ID ごとに別 container
- `nautilus-replay`
  - run ID ごとに別 container
- `shadow-start`
  - strategy / instance ごとに別 container

したがって、Hermes は同じ skill を複数回呼べるが、衝突回避は remote 側の命名規則と lock が担う。

## 8. 初期提案

### 8.1 SSH 認証方針

初期提案:

- `pi-research` と `pi-live` の `sshd` は LAN 内のみ到達可能にする
- 外部公開はしない
- 初期は `password authentication` を許可する
- Hermes 実行主体は `pi-ctl` から `pi-research` / `pi-live` へ接続する
- 接続先ユーザーは専用の運用ユーザーを分ける
  - 例:
    - `pi-research`: `hermes-research`
    - `pi-live`: `hermes-live`

この前提で十分と判断する理由:

- 外部入口は `Cloudflare Access / Tunnel` 側で閉じる
- LAN 側は `UniFi IDS/IPS` で守る
- SSH はインターネットに露出しない
- 初期フェーズでは、鍵管理や host allowlist より運用単純性のほうが価値が高い

初期ルール:

- `PasswordAuthentication yes`
- root login は禁止
- `hermes-research` は `pi-research` にのみ作成
- `hermes-live` は `pi-live` にのみ作成
- `sudo` は必要最小限に限定する
- shell の利用者は Hermes skill 実装のみを想定する

後段で強化するもの:

- 鍵認証への移行
- `Match User` ベースの追加制御
- command allowlist
- host allowlist

### 8.2 `Paperclip issue documents` の正規化

初期提案:

- skill ごとにバラバラな自由文を返さない
- `Paperclip issue documents` には固定 key と薄い JSON を返す
- 詳細ログや大きい成果物は artifact path / attachment ref に逃がす

固定したい document key:

- `execution_request`
- `execution_result`
- `validation_report`
- `shadow_status`
- `shadow_report`

### 8.3 result storage の前提

- `pi-research` / `pi-live` の result は `parquet` と `json` を truth にする
- `DuckDB` は複数 run / instance 比較の query layer とする
- `DuckDB` は node-local utility container として扱ってよい
- `LLM-WIKI` には Hermes が抽象化した結果だけを戻す
- raw `parquet` や event log 全文は wiki に直接入れない

詳細は [research-result-storage-contract.md](D:/GitHub/quant-agent-platform/docs/research-result-storage-contract.md) を参照。

### 8.4 `execution_request` schema

用途:

- `Strategy Lead` または上位 role から `Quant Research Engineer` / `Shadow Trading Operator` へ渡す実行依頼

初期 schema:

```json
{
  "kind": "execution_request",
  "skill": "nautilus-backtest",
  "strategy_id": "mean-reversion-001",
  "revision": "gitsha-or-tag",
  "config_ref": "docs://config/backtest/base-v1",
  "run_id": "bt-20260418-001",
  "requested_by": "strategy_lead",
  "issue_id": "pcp-123",
  "notes": "focus on drawdown after 2024-01 regime shift"
}
```

### 8.5 `execution_result` schema

用途:

- skill 実行の最小結果を返す

初期 schema:

```json
{
  "kind": "execution_result",
  "skill": "nautilus-backtest",
  "strategy_id": "mean-reversion-001",
  "revision": "gitsha-or-tag",
  "run_id": "bt-20260418-001",
  "node": "pi-research",
  "container_name": "bt-mean-reversion-001-bt-20260418-001",
  "status": "succeeded",
  "started_at": "2026-04-18T10:30:00+09:00",
  "finished_at": "2026-04-18T10:42:11+09:00",
  "artifact_ref": "/srv/quant/research/runs/bt-20260418-001",
  "report_ref": "document://validation_report/bt-20260418-001",
  "summary": "sharpe improved, turnover increased"
}
```

### 8.6 `validation_report` schema

用途:

- backtest / replay の要約結果

初期 schema:

```json
{
  "kind": "validation_report",
  "run_id": "bt-20260418-001",
  "strategy_id": "mean-reversion-001",
  "mode": "backtest",
  "window": {
    "from": "2023-01-01",
    "to": "2026-03-31"
  },
  "headline_metrics": {
    "sharpe": 1.42,
    "max_drawdown": -0.083,
    "turnover": 5.8
  },
  "decision_hint": "promising_but_needs_turnover_review",
  "artifact_ref": "/srv/quant/research/runs/bt-20260418-001"
}
```

### 8.7 `shadow_status` schema

用途:

- live validation instance の現在状態

初期 schema:

```json
{
  "kind": "shadow_status",
  "instance_id": "shadow-20260418-001",
  "strategy_id": "mean-reversion-001",
  "revision": "gitsha-or-tag",
  "node": "pi-live",
  "status": "running",
  "container_name": "shadow-mean-reversion-001-shadow-20260418-001",
  "spool_ref": "/srv/quant/live/spool/shadow-20260418-001",
  "updated_at": "2026-04-18T11:03:05+09:00"
}
```

### 8.8 `shadow_report` schema

用途:

- stop / rollback / incident 時の結果要約

初期 schema:

```json
{
  "kind": "shadow_report",
  "instance_id": "shadow-20260418-001",
  "strategy_id": "mean-reversion-001",
  "status": "stopped",
  "reason": "manual_stop_after_validation_window",
  "events_ref": "/srv/quant/live/reports/shadow-20260418-001",
  "summary": "no critical risk trigger, latency stable"
}
```

## 9. 未決事項

- `docker` から `microvm` へ差し替える条件
- `Hindsight` に返す summary の標準 schema
