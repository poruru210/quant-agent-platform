# Quant Agent Platform Architecture Overview

本書は、このプロジェクトの全体像を素早く把握するための概要資料である。  
詳細仕様ではなく、まず以下を一目で掴むことを目的とする。

- 全体アーキテクチャ
- ノード配置
- 呼び出しフロー
- ロールごとの責務
- 主要 IF の対応関係

詳細は末尾の参照文書へ進む。

---

## 1. 一言でいうと

このシステムは、以下の役割分担で動く。

- `Paperclip`
  - control plane
  - task / issue / document / assignment の source of truth
- `Hermes`
  - 実行主体
  - Paperclip から起動され、skill を使って仕事を進める
- `NautilusTrader`
  - backtest / replay / paper / shadow を行う execution engine
- `LLM-WIKI`
  - strategy research / planning / long-lived knowledge
- `Hindsight`
  - Hermes の自己改善と実行履歴
- `DuckDB + parquet`
  - research / live result の query / analysis layer

---

## 2. 全体アーキテクチャ

```text
Browser
  -> Cloudflare Access
    -> Cloudflare Tunnel
      -> pi-ctl
         -> Paperclip native Web UI / API
         -> Hermes native runtime
         -> PostgreSQL container
         -> optional native Hindsight
              -> SSH -> pi-research
                  -> NautilusTrader backtest / replay / simulation
                  -> parquet artifacts
                  -> DuckDB query layer
              -> SSH -> pi-live
                  -> NautilusTrader paper / shadow
                  -> parquet live artifacts
                  -> DuckDB query layer
                  -> live spool / reports

LLM-WIKI
  <- Hermes が strategy / validation / incident の知見を要約して反映
```

---

## 3. ノード配置

### 3.1 `pi-ctl`

役割:

- state / control plane
- 外部公開の着地点
- Hermes runtime の配置先

主なコンポーネント:

- `Paperclip native`
- `Hermes native`
- `PostgreSQL container`
- `cloudflared`
- 必要なら `Hindsight native`

### 3.2 `pi-research`

役割:

- research execution plane
- backtest / replay / simulation 実行先
- result storage / analysis

主なコンポーネント:

- `NautilusTrader`
- alpha / feature / simulation jobs
- run artifact
- `DuckDB`
- Git worktree / workspace

### 3.3 `pi-live`

役割:

- live validation plane
- paper / shadow 実行先

主なコンポーネント:

- `NautilusTrader live engine`
- local `DuckDB`
- live parquet artifacts
- market data receive
- spool / reports
- lightweight risk guard

---

## 4. 呼び出しフロー

### 4.1 strategy 立案

```text
CEO / CTO
  -> Strategy Lead
    -> LLM-WIKI research / query / plan
    -> Paperclip issue / brief / acceptance を作成
```

ここでの主系:

- 調査と構想は `LLM-WIKI`
- task 化と管理は `Paperclip`

### 4.2 research execution

```text
Paperclip issue assignment
  -> Hermes on pi-ctl
    -> research skill
      -> SSH to pi-research
        -> isolated Nautilus job
          -> parquet / json artifacts
          -> DuckDB query
    -> execution_result / validation_report を Paperclip に返す
    -> 必要なら LLM-WIKI に lesson を戻す
```

ここでの主系:

- orchestration は `Paperclip`
- execution は `Hermes skill`
- engine は `NautilusTrader`
- result analysis は `parquet + DuckDB`
- knowledge 化は `LLM-WIKI`

### 4.3 live validation

```text
Paperclip issue assignment
  -> Hermes on pi-ctl
    -> live skill
      -> SSH to pi-live
        -> isolated shadow instance
          -> spool / report / status
    -> shadow_status / shadow_report を Paperclip に返す
```

ここでの主系:

- live の source of truth は `Paperclip`
- 実行先は `pi-live`
- heavy research job と混在させない

### 4.4 incident / self-improvement

```text
incident or validation outcome
  -> Paperclip documents / artifacts
  -> Incident Analyst が分析
  -> Hermes の自己改善は Hindsight を参照
  -> 再利用価値の高い知見だけ LLM-WIKI に戻す
```

---

## 5. ロール一覧

### 5.1 `CEO / CTO`

責務:

- 全体統括
- 優先順位決定
- task assign

主に触るもの:

- `Paperclip`

### 5.2 `Strategy Lead`

責務:

- strategy hypothesis
- research brief
- acceptance criteria

主に触るもの:

- `LLM-WIKI`
- `Paperclip`

### 5.3 `Quant Research Engineer`

責務:

- 実装
- backtest / replay / simulation
- validation report 作成

主に触るもの:

- `Paperclip`
- `Hermes research skills`
- `pi-research`
- `DuckDB + parquet`

### 5.4 `Shadow Trading Operator`

責務:

- paper / shadow 起動停止
- status 確認
- live validation report

主に触るもの:

- `Paperclip`
- `Hermes live skills`
- `pi-live`

### 5.5 `Incident Analyst`

責務:

- postmortem
- divergence 分析
- lesson 抽出

主に触るもの:

- `Paperclip`
- `Hindsight`
- `LLM-WIKI`

---

## 6. ロールと IF のマッピング

| Role | 主な入力 IF | 主な出力 IF | 主な実行先 |
| --- | --- | --- | --- |
| `CEO / CTO` | `Paperclip issue / assignment / comment` | task assign / approval / priority | `pi-ctl` |
| `Strategy Lead` | `LLM-WIKI query/research/plan`, `Paperclip documents` | `brief`, `acceptance`, hypothesis notes | `pi-ctl` |
| `Quant Research Engineer` | `execution_request`, codebase, config refs | `execution_result`, `validation_report`, artifacts | `pi-research` |
| `Shadow Trading Operator` | approved revision, deploy request | `shadow_status`, `shadow_report` | `pi-live` |
| `Incident Analyst` | history, reports, artifacts, Hindsight | `postmortem`, next iteration notes, wiki lesson | `pi-ctl` |

---

## 7. Hermes Skill マッピング

Hermes が使う skill は 2 層ある。

- bundled / optional の既存 Hermes skill
- このプロジェクトで追加する Nautilus / runtime skill

### 7.1 既存 Hermes skill の位置づけ

このプロジェクトで関係が深い既存 skill:

| Skill | 由来 | 導入状態 | 使いどころ |
| --- | --- | --- | --- |
| `plan` | Hermes 既存 | bundled | strategy 実行前の計画整理。実行せず plan を作る |
| `code-review` | Hermes 既存 | bundled | 実装変更や validation code の review |
| `native-mcp` | Hermes 既存 | bundled | 外部 MCP server を Hermes tool として注入する場合 |
| `qmd` | Hermes 既存 | optional | ローカル知識ベース検索を使いたい場合の補助。`LLM-WIKI` とは別物 |
| `docker-management` | Hermes 既存 | optional | isolation layer を運用・調査するときの補助 |

補足:

- 上の skill は Hermes 側の既存資産であり、このプロジェクト専用 IF ではない
- 主 execution IF は後述の project skill である

### 7.2 project skill の位置づけ

このプロジェクトで公開 IF として扱う skill:

| Skill | 由来 | 導入状態 | 対象ノード | 用途 | 主な返却 |
| --- | --- | --- | --- | --- | --- |
| `nautilus-backtest` | project 新規 | 要実装 | `pi-research` | backtest 実行 | `execution_result`, `validation_report` |
| `nautilus-replay` | project 新規 | 要実装 | `pi-research` | replay 実行 | `execution_result`, `validation_report` |
| `feature-generation` | project 新規 | 要実装 | `pi-research` | feature 作成 | artifact / summary |
| `simulation-run` | project 新規 | 要実装 | `pi-research` | simulation 実行 | artifact / summary |
| `shadow-start` | project 新規 | 要実装 | `pi-live` | paper / shadow 起動 | `shadow_status` |
| `shadow-stop` | project 新規 | 要実装 | `pi-live` | instance 停止 | `shadow_report` |
| `shadow-status` | project 新規 | 要実装 | `pi-live` | 稼働状態確認 | `shadow_status` |
| `shadow-tail-logs` | project 新規 | 要実装 | `pi-live` | live log 確認 | log ref / summary |
| `shadow-collect-report` | project 新規 | 要実装 | `pi-live` | report 回収 | `shadow_report` |
| `shadow-rollback` | project 新規 | 要実装 | `pi-live` | rollback | status / report |

### 7.3 ロール別の skill 対応

| Role | Hermes 既存 skill | project 新規 skill |
| --- | --- | --- |
| `CEO / CTO` | 原則なし | 原則なし |
| `Strategy Lead` | `plan`, 必要なら `native-mcp` | 原則なし |
| `Quant Research Engineer` | `code-review`, 必要なら `plan`, `native-mcp` | `nautilus-backtest`, `nautilus-replay`, `feature-generation`, `simulation-run` |
| `Shadow Trading Operator` | 必要なら `docker-management` | `shadow-start`, `shadow-stop`, `shadow-status`, `shadow-tail-logs`, `shadow-collect-report`, `shadow-rollback` |
| `Incident Analyst` | `code-review`, 必要なら `plan`, `qmd` | 原則なし。report 参照中心 |

重要:

- 実運用の主 IF は project skill
- bundled skill は補助
- `LLM-WIKI` 自体は Hermes skill ではなく別系統の knowledge system

### 7.4 読み方

- `Hermes 既存 / bundled`
  - Hermes に最初から入っている
- `Hermes 既存 / optional`
  - Hermes 側に用意はあるが、追加導入が必要
- `project 新規 / 要実装`
  - このプロジェクトで新しく定義して実装する

---

## 8. システム間 IF のマッピング

| From | To | 公開 IF | 備考 |
| --- | --- | --- | --- |
| Browser | `Paperclip UI` | Cloudflare Access + Tunnel | 外部公開は UI 系のみ |
| `Paperclip` | `Hermes` | `hermes_local` adapter | Hermes runtime は `pi-ctl` |
| `Hermes` | bundled / optional Hermes skill | skill call | `plan`, `code-review`, `native-mcp` など |
| `Hermes` | `pi-research` | Hermes skill + SSH | backtest / replay / simulation |
| `Hermes` | `pi-live` | Hermes skill + SSH | paper / shadow |
| `NautilusTrader` | artifact store | `parquet/json/logs` | truth は file artifact |
| `Hermes` | `DuckDB` | query | research / live の複数 run 比較 |
| `Hermes` | `LLM-WIKI` | wiki command / file layout | 知見の昇格 |
| `Hermes` | `Hindsight` | run history / self-improvement loop | 自己改善主系 |
| `Hermes` | `Paperclip` | `execution_result`, `validation_report`, `shadow_status`, `shadow_report` | issue documents へ返却 |

---

## 9. データの流れ

### 8.1 strategy knowledge

```text
idea
  -> LLM-WIKI
  -> brief / acceptance
  -> Paperclip issue
```

### 8.2 execution result

```text
Nautilus run
  -> parquet / json / logs on pi-research
  -> DuckDB query
  -> Hermes summary
  -> Paperclip validation_report
  -> LLM-WIKI lesson
```

### 8.3 live validation result

```text
shadow run
  -> parquet / json / logs / spool on pi-live
  -> DuckDB query
  -> Hermes summary
  -> Paperclip shadow_report
  -> 必要なら incident / wiki lesson
```

---

## 10. 重要な設計判断

### 9.1 なぜ Hermes は `pi-ctl` か

- `hermes_local` adapter の実 IF に最も素直
- `Paperclip` と Hermes runtime の主語が揃う
- research / live を対称に SSH 実行先として扱える

### 9.2 なぜ `Paperclip native + PostgreSQL container` か

- Raspberry Pi では組み込み DB が不安定だった
- `Paperclip` 本体は `hermes_local` と近接させたい
- DB だけを container に切り出すと IF と運用の両方が安定する

### 9.3 なぜ Nautilus は compose 常駐ではないか

- backtest / replay は複数同時実行が前提
- strategy / run ごとに隔離したい
- shadow も instance 単位 lifecycle が自然

### 9.4 なぜ `DuckDB + parquet` か

- raw result を file truth として残せる
- 複数 run 比較を軽く行える
- live の実 tick / spread / event も同じ形で後から確認できる
- wiki を生データ置き場にしなくてよい

### 9.5 なぜ `LLM-WIKI` と `Hindsight` を分けるか

- `LLM-WIKI` は strategy / research / planning の知識面
- `Hindsight` は Hermes 自己改善の履歴面

---

## 11. 参照

Hermes の既存 skill catalog 一次情報:

- https://github.com/NousResearch/hermes-agent/blob/main/website/docs/reference/skills-catalog.md
- https://github.com/nousresearch/hermes-agent

---

## 12. 最初に見るべき詳細文書

- 全体設計: [paperclip-hermes-nautilus-design.md](D:/GitHub/quant-agent-platform/docs/paperclip-hermes-nautilus-design.md)
- runtime / role policy: [paperclip-hermes-runtime-contract.md](D:/GitHub/quant-agent-platform/docs/paperclip-hermes-runtime-contract.md)
- Hermes skill IF: [hermes-nautilus-skill-contract.md](D:/GitHub/quant-agent-platform/docs/hermes-nautilus-skill-contract.md)
- research result 保存: [research-result-storage-contract.md](D:/GitHub/quant-agent-platform/docs/research-result-storage-contract.md)
- wiki 運用: [llm-wiki-operational-contract.md](D:/GitHub/quant-agent-platform/docs/llm-wiki-operational-contract.md)
- runtime 運用メモ: [README.md](D:/GitHub/quant-agent-platform/deploy/runtime/README.md)
