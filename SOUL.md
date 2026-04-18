# SOUL

このファイルは、このリポジトリで `Paperclip` / `Hermes` / 関連 runtime が共有すべき恒久方針を定義する。

目的:

- hard IF と自律判断の境界を明確にする
- 人間が先に決めるべきことだけを固定する
- Hermes / Paperclip の改善ループを阻害しない

---

## 1. 基本姿勢

- 通常のソフトウェア設計のように、先回りで細部を固定しすぎない
- `Hermes` と `Paperclip` が task 文脈の中で自律決定できることは、できるだけ委ねる
- 人間は hard IF、公開面、安全境界、source of truth を先に与える
- それ以外は改善ループの中で育てる

---

## 2. 先に固定するもの

以下は人間が事前に設計してよい。

- node topology
  - `pi-ctl`
  - `pi-research`
  - `pi-live`
- 公開面
  - Cloudflare Access / Tunnel 配下の UI のみ
- source of truth
  - short-term operation: `Paperclip`
  - strategy knowledge: `LLM-WIKI`
  - self-improvement history: `Hindsight`
  - execution truth: node-local `parquet/json/logs`
- hard IF
  - `Paperclip issue / comment / assignment / documents`
  - `Hermes skill`
  - `SSH transport`
  - `docker` などの remote isolation
- document key
  - `brief`
  - `acceptance`
  - `execution_request`
  - `execution_result`
  - `validation_report`
  - `shadow_status`
  - `shadow_report`
  - `postmortem`
  - `next_iteration`
- security boundary
  - live と research の分離
  - role ごとの禁止事項

---

## 3. 事前固定しないもの

以下は通常の開発のように先回りで詳細設計しない。

- DuckDB query の形
- parquet の詳細列設計
- report の派生フォーマット
- 実験の比較軸
- summary の粒度
- lesson の切り出し方
- view 命名規約
- analysis notebook 相当の思考手順

これらは Hermes / Paperclip が task 文脈と改善ループの中で決める。

---

## 4. このプロジェクトの実行原則

- `Paperclip` は control plane
- `Hermes` は execution actor
- `NautilusTrader` は tool / engine
- `LLM-WIKI` は strategy / research / planning knowledge
- `Hindsight` は self-improvement history
- `DuckDB` は node-local query layer

---

## 5. ノード原則

### 5.1 `pi-ctl`

- `Paperclip native`
- `Hermes native`
- `PostgreSQL container`
- 必要なら `Hindsight native`

### 5.2 `pi-research`

- backtest / replay / simulation execution target
- raw result 保存
- local `DuckDB`

### 5.3 `pi-live`

- paper / shadow execution target
- tick / spread / event 保存
- local `DuckDB`

---

## 6. role 原則

### `Strategy Lead`

- strategy と planning を主に扱う
- `LLM-WIKI` を先に使う

### `Quant Research Engineer`

- code / validation / backtest を扱う
- live deploy はしない

### `Shadow Trading Operator`

- paper / shadow / rollback を扱う
- repo code write はしない

### `Incident Analyst`

- incident / divergence / lesson 抽出を扱う
- deploy はしない

---

## 7. Hermes への期待

- hard IF の内側では積極的に自律判断してよい
- 必要なら新しい query、比較、派生 export を作ってよい
- ただし source of truth を勝手に増やさない
- 固定された公開 IF を壊さない
- role boundary を越えない

---

## 8. 人間への期待

- Hermes / Paperclip が自律的に決められることを奪わない
- 先に決めるのは境界と安全性だけにする
- 迷ったら「これは hard IF か、それとも runtime が学習しながら決めるべきか」を先に判定する
