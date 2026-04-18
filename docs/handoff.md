# Quant Agent Platform Handoff

本書は、このリポジトリを `Codex cloud` や別セッションへ引き継ぐための短い handoff である。  
詳細は各設計書を参照しつつ、まずは現状の前提と次アクションを素早く掴むことを目的とする。

---

## 1. 目的

このプロジェクトは、戦略立案、研究、実装、検証、shadow/live 実行、事後分析を、`Paperclip`、`Hermes`、`NautilusTrader`、`LLM-WIKI`、`Hindsight` を組み合わせて運用するための quant platform を設計している。

現在の基本方針は次の通り。

- `Paperclip`
  - control plane
  - task / issue / document / assignment の source of truth
- `Hermes`
  - 実行主体
  - `Paperclip` から起動され、skill を使って仕事を進める
- `NautilusTrader`
  - backtest / replay / paper / shadow の execution engine
- `LLM-WIKI`
  - strategy research / planning / long-lived knowledge
- `Hindsight`
  - Hermes の自己改善と実行履歴
- `DuckDB + parquet`
  - research / live result の query / analysis layer

---

## 2. 現在のノード構成

- `pi-ctl`
  - `Paperclip native`
  - `Hermes native`
  - `PostgreSQL container`
  - `cloudflared`
  - 必要なら `Hindsight native`
- `pi-research`
  - `NautilusTrader` の backtest / replay / simulation 実行先
  - run artifact
  - `DuckDB`
- `pi-live`
  - `NautilusTrader` の paper / shadow 実行先
  - live artifact
  - `DuckDB`

公開面:

- Cloudflare Tunnel で外に出すのは `Paperclip / Hindsight` の Web UI のみ
- `PostgreSQL`, `pi-research`, `pi-live`, SSH, Nautilus 実行面は外部公開しない

---

## 3. 実IFの整理

### 3.1 `Paperclip -> Hermes`

重要な実装理解:

- `hermes_local` は local の Hermes CLI を起動する adapter
- したがって、`Paperclip` と `Hermes` は同じホスト `pi-ctl` に置くのが実IFに素直

### 3.2 `Hermes -> pi-research / pi-live`

- `Hermes` は skill の内側で `SSH` を使って remote 実行する
- `pi-research`
  - backtest / replay / simulation
- `pi-live`
  - paper / shadow / live validation

重要:

- 直接スクリプトを公開 IF にしない
- 公開 IF はあくまで `Hermes skill`
- `SSH`, `docker`, 将来の `microvm` は skill の内部実装

### 3.3 `NautilusTrader`

- compose 常駐サービスとしては置かない
- per-job / per-instance 実行の execution target として扱う

---

## 4. 設計原則

このプロジェクトでは、通常の開発のように全項目を事前に固定しない。  
先に固定するのは次だけである。

- hard IF
- node 境界
- 公開面
- 安全境界
- source of truth
- human approval boundary

逆に、次のようなものは Hermes / Paperclip の改善ループに委ねる。

- DuckDB query の細部
- parquet 列設計の細部
- 派生 export の粒度
- 実務フローの細かな洗練
- 補助 skill の増殖

この方針は [SOUL.md](/D:/GitHub/quant-agent-platform/SOUL.md:1) にも反映済み。

---

## 5. Hermes と Paperclip の役割分担

### 5.1 Hermes

- global / durable identity は `SOUL.md`
- repo / project context は `AGENTS.md`
- memory は `MEMORY.md` / `USER.md`
- skill は `~/.hermes/skills/`

### 5.2 Paperclip

Paperclip には `Hermes` の `SOUL.md` と同型の単一ファイルはない。  
Agent の振る舞いは主に次の 3 層で決まる。

- `agent record`
  - `name`, `role`, `title`, `reportsTo`, `capabilities`, `adapterType`, `adapterConfig`, `runtimeConfig`
- `adapterConfig.promptTemplate`
  - legacy prompt
- `instructions bundle`
  - managed instructions
  - default entry file は `AGENTS.md`

設計上の読み方:

- `Hermes SOUL.md`
  - Hermes 全体の恒久方針
- `Paperclip capabilities`
  - role の短い説明
- `Paperclip instructions bundle`
  - role ごとの実務指示
- `adapterConfig.promptTemplate`
  - adapter 実行時の補助 prompt

---

## 6. compose の現状方針

compose は `pi-ctl` 上の `PostgreSQL` に限定する方針。

理由:

- `Paperclip native + Hermes native` の方が `hermes_local` の実IFに合う
- `NautilusTrader` は固定常駐 compose より per-job 実行が自然
- Raspberry Pi 上で組み込み DB は不安定だったため、`PostgreSQL container` を使う

したがって、現在の基本線は:

- `Paperclip native`
- `Hermes native`
- `PostgreSQL container`

である。

---

## 7. 既知の重要論点

### 7.1 人間が先に決めるべきもの

- `SOUL.md`
- Paperclip の role taxonomy
- role ごとの `instructions bundle` の骨組み
- 公開面と安全境界
- human approval boundary

### 7.2 まだ未確定だが、次に詰める価値が高いもの

- `pi-ctl -> pi-research / pi-live` の SSH bootstrap 方針
  - 初回は password auth
  - その後は Hermes の bootstrap skill で鍵認証へ移行する案が有力
- project-specific Hermes skills
  - `nautilus-backtest`
  - `nautilus-replay`
  - `feature-generation`
  - `simulation-run`
  - `shadow-start`
  - `shadow-stop`
  - `shadow-status`

### 7.3 あえて今は固定しないもの

- DuckDB schema の細部
- parquet 列定義
- レポート export の詳細
- skill 実装の細かな最適化

---

## 8. 最初に読むべき文書

- [architecture-overview.md](/D:/GitHub/quant-agent-platform/docs/architecture-overview.md:1)
- [SOUL.md](/D:/GitHub/quant-agent-platform/SOUL.md:1)
- [paperclip-hermes-nautilus-design.md](/D:/GitHub/quant-agent-platform/docs/paperclip-hermes-nautilus-design.md:1)
- [paperclip-hermes-runtime-contract.md](/D:/GitHub/quant-agent-platform/docs/paperclip-hermes-runtime-contract.md:1)
- [hermes-nautilus-skill-contract.md](/D:/GitHub/quant-agent-platform/docs/hermes-nautilus-skill-contract.md:1)
- [paperclip-agent-instructions-model.md](/D:/GitHub/quant-agent-platform/docs/paperclip-agent-instructions-model.md:1)
- [hermes-paperclip-config-surfaces.md](/D:/GitHub/quant-agent-platform/docs/hermes-paperclip-config-surfaces.md:1)
- [research-result-storage-contract.md](/D:/GitHub/quant-agent-platform/docs/research-result-storage-contract.md:1)

---

## 9. 次アクション候補

### Option A

`Paperclip` role ごとの `instructions bundle / AGENTS.md` テンプレートを作る。

### Option B

`Hermes` 用の bootstrap skill 設計を詰める。

- SSH 初期接続
- 鍵配布
- known_hosts 登録
- 鍵認証移行

### Option C

`pi-ctl` の native service 設計を詰める。

- `Paperclip`
- `Hermes`
- `cloudflared`
- `PostgreSQL container`

