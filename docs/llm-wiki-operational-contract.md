# LLM-WIKI Operational Contract

## 1. 目的

本書は、このプロジェクトで `LLM-WIKI` をどう使うかを定義する。

重要:

- `LLM-WIKI` は command と file layout を中心とした research / synthesis system である
- `Paperclip issue documents` の代替ではない
- ただし **strategy 案のタイミングから使う**
- 短期タスク管理は `Paperclip`
- 調査、仮説形成、計画立案、長期知識化は `LLM-WIKI`
- Hermes の自己改善主系は `Hindsight` であり、`LLM-WIKI` はその主保存先ではない

---

## 2. 基本原則

### 2.1 Source of truth の分離

`Paperclip` が持つもの:

- 今回の task
- 今回の brief
- 今回の validation report
- 今回の deploy note
- 今回の postmortem

`LLM-WIKI` が持つもの:

- strategy 案の初期調査
- hypothesis 形成の材料
- implementation plan の地盤
- 再利用価値のある strategy rationale
- incident から抽象化された lessons learned
- venue / market / execution に関する継続知識
- 実験から得た pattern
- 検証済みの作法 / guardrail / checklist

`Hindsight` が持つもの:

- Hermes run history
- 実行の振り返り
- 自己改善用の観測履歴
- prompt / behavior 改善の材料

### 2.2 strategy phase で先に wiki を使う

このプロジェクトでは `LLM-WIKI` を strategy 案のタイミングから使う。

原則:

1. `Strategy Lead` は新規戦略や改善テーマを受けたら、まず wiki で調査する
2. `research` / `query` / `plan` で仮説と実装・検証方針を整理する
3. その結果を execution 用に `Paperclip issue documents` へ切り出す
4. 実行後の lesson や incident も再び wiki に戻す

つまり:

- 仮説形成前後は wiki が先
- 実行管理は Paperclip が先
- Hermes 自己改善は Hindsight が先
- 知見蓄積は wiki へ戻す

### 2.3 `nvk/llm-wiki` 由来の運用上の意味

元の `nvk/llm-wiki` では、以下が明確である。

- `/wiki:research`
  - 調査そのものを行う
- `/wiki:plan`
  - wiki research を前提に implementation plan を作る
- question mode
  - 質問を playbook や thesis 候補へ落とす
- `--new-topic`
  - 新規テーマをその場で wiki 化する

したがって、本プロジェクトでも `LLM-WIKI` を「後で整理する保管庫」としてだけ扱わない。

### 2.4 API 前提にしない

現時点では `LLM-WIKI` を REST API で叩く前提にしない。  
まずは以下の IF を採る。

- `/wiki init`
- `/wiki:research`
- `/wiki:ingest`
- `/wiki:compile`
- `/wiki:query`
- `/wiki:plan`
- `/wiki:output`
- `/wiki:lint`

---

## 3. 採用する wiki topology

### 3.1 初期方針

初期は **project-local wiki** を主に採る。

理由:

- strategy 案のタイミングから repo 文脈の近くで調査したい
- `nvk/llm-wiki` 自体が `.wiki/` を first-class に扱っている
- strategy repo と一緒に管理しやすい
- `Strategy Lead` と `Incident Analyst` が同じ知識空間を見やすい
- node 配置上、state / control ノードで完結しやすい

### 3.2 初期構造

リポジトリ直下または運用用 workspace に `.wiki/` を持つ想定とする。

例:

```text
.wiki/
├─ inbox/
├─ raw/
│  ├─ articles/
│  ├─ papers/
│  ├─ repos/
│  ├─ notes/
│  └─ data/
├─ wiki/
│  ├─ concepts/
│  ├─ topics/
│  ├─ references/
│  └─ theses/
├─ output/
├─ _index.md
├─ config.md
└─ log.md
```

補足:

- これは `nvk/llm-wiki` の local wiki 構造に寄せる
- 以前の `strategies/` `incidents/` などの project-specific 分類は、まず article のタグや topic 側で表現し、必要になってから増やす

### 3.3 将来拡張

将来的には topic wiki の分離を検討する。

候補:

- `market-microstructure`
- `execution-engineering`
- `exchange-ops`
- `strategy-library`

ただし初期段階では分けすぎない。

---

## 4. 標準コマンドの使い分け

### 4.1 `/wiki init <name> --local`

使う場面:

- repo 単位で strategy / incident / validation 知識を育てたいとき

### 4.2 `/wiki:research`

使う場面:

- 新規 strategy 案の初期調査
- 仮説候補の探索
- 市場 / venue / prior art の調査

重要:

- 新規テーマは `--new-topic`
- local wiki では `--local`
- 質問形式でも使える

### 4.3 `/wiki:query`

使う場面:

- 既存知識の読み出し
- strategy brief 前の確認
- incident pattern の照会

### 4.4 `/wiki:plan`

使う場面:

- strategy 実装計画
- validation plan
- operator playbook 草案

重要:

- 元の repo では `wiki-grounded implementation plan` を作る command として定義されている
- したがって `Strategy Lead` が strategy brief を起こす前後で使う前提にする

### 4.5 `/wiki:ingest`

使う場面:

- URL / file / text を raw source として取り込む
- Paperclip 由来の report 要約を raw/notes に落とす

### 4.6 `/wiki:compile`

使う場面:

- raw sources を article に昇格するとき
- research の後に article を整備するとき

### 4.7 `/wiki:output`

使う場面:

- report
- checklist
- summary
- comparison

### 4.8 `/wiki:lint`

使う場面:

- structure を整える
- compile 後の崩れを直す

---

## 5. Role ごとの責務

### 5.1 `Strategy Lead`

やること:

- strategy 案のタイミングで wiki を初期化 / 参照する
- `research` で仮説候補と周辺知識を集める
- `query` で既存知識を読み出す
- `plan` で実装・検証計画の叩き台を作る
- その結果を `Paperclip brief` に切り出す
- 後で strategy rationale を再び wiki に戻す

主コマンド:

- `/wiki init <name> --local`
- `/wiki:research "<topic or question>" --local`
- `/wiki:query "<strategy or incident question>" --local`
- `/wiki:plan "<new experiment or improvement>" --local`
- `/wiki:ingest <derived note or source> --local`
- `/wiki:compile --local`

### 5.2 `Quant Research Engineer`

やること:

- 基本は Paperclip issue documents 側が主
- ただし再利用価値の高い validation lesson を wiki 候補として切り出す

主コマンド:

- 直接 wiki を触るのは最小限
- 必要なら `/wiki:ingest --local`

原則:

- research engineer は wiki の主編集者ではない
- validation report 本体はまず Paperclip に置く

### 5.3 `Shadow Trading Operator`

やること:

- 直接 wiki を編集しない
- 運用ノウハウ化したいものは deploy note / incident note として Paperclip に返す

### 5.4 `Incident Analyst`

やること:

- postmortem を読み、再利用知識へ昇格する
- incident pattern を compile する
- 必要なら operator artifact を output する

主コマンド:

- `/wiki:ingest --local`
- `/wiki:compile --local`
- `/wiki:query "<have we seen this pattern before?>" --local`
- `/wiki:output summary --local`

---

## 6. strategy phase からの標準ワークフロー

### 6.1 新規 strategy 案

1. `Strategy Lead` が `.wiki/` を確認する
2. 未作成なら `/wiki init <name> --local`
3. `/wiki:research "<topic or question>" --local` で初期調査する
4. `/wiki:query` で既存知識を整理する
5. `/wiki:plan "<experiment or implementation goal>" --local` で計画叩き台を作る
6. その結果を `Paperclip issue documents/brief` に切り出す

### 6.2 Strategy brief 後

1. `Strategy Lead` が `brief` と整合するよう wiki note を更新する
2. `/wiki:compile --local`

### 6.3 Validation 後

1. `Quant Research Engineer` が `validation_report` を issue document に保存する
2. `Strategy Lead` または `Incident Analyst` が再利用価値のある lesson を抽出する
3. 必要なら `/wiki:ingest --local`
4. `/wiki:compile --local`

### 6.4 Incident 後

1. `Incident Analyst` が `postmortem` document を作る
2. 抽象化できる結論を wiki へ戻す
3. `/wiki:compile --local`
4. 必要なら `/wiki:output checklist --local`

---

## 7. 何を最初から wiki に入れるか

### 7.1 strategy phase で入れるもの

以下は strategy 段階から wiki に入れてよい。むしろ入れる前提でよい。

- 新規戦略テーマの初期 research
- 仮説候補の整理
- venue / market / microstructure の下調べ
- 競合 / 類似手法 / prior art
- 実験設計の前提知識

### 7.2 execution 後に戻すもの

以下は execution 後に wiki へ戻す。

- validation から得た再利用 lesson
- incident の抽象化結果
- operator checklist
- strategy rationale の更新版
- DuckDB で確認した run 間比較の要約

### 7.3 原則 wiki にしないもの

- 一時的な task 進捗
- 単発の completion comment
- task 個別の雑多なログ全文
- まだ検証不足の主張
- raw `parquet` artifact 本体
- DuckDB の生クエリ結果全文

---

## 8. Paperclip との接続規約

### 8.1 Paperclip へ渡す単位

wiki 調査結果は、そのまま runtime task にしない。  
execution に渡すときは `Paperclip issue documents` に切り出す。

対象:

- `brief`
- `acceptance`
- `validation_report`
- `postmortem`

### 8.2 wiki へ戻す単位

issue 全文を丸ごと入れず、以下のどれかにする。

- 要約ノート
- 抽象化済み lesson
- raw source snapshot
- DuckDB 比較結果の人間向け要約

ただし strategy phase の exploratory note や external source は、最初から wiki に入れてよい。

### 8.3 ref の残し方

wiki に戻す知識は、最低限以下を残す。

- originating issue ID
- document key
- commit SHA
- report ref

### 8.4 comment を直接 wiki 化しない

comment はノイズが多いので、直接 wiki article にしない。  
必要なら要約して `inbox/` や `raw/notes/` に落とす。

---

## 9. Node 配置との関係

### 9.1 初期配置

wiki の編集主体は state / control ノードに寄せる。

理由:

- `Strategy Lead`
- `Incident Analyst`

が主編集者だからである。

### 9.2 research ノードとの関係

research ノードは:

- issue document を書く
- `parquet` artifact と `DuckDB` query layer を保持する
- 必要なら wiki 候補メモを出す

までに留める。

詳細は [research-result-storage-contract.md](D:/GitHub/quant-agent-platform/docs/research-result-storage-contract.md) を参照。

### 9.3 live validation ノードとの関係

live validation ノードは wiki を直接更新しない。  
運用上の知見は Paperclip 経由で state / control 側へ戻す。

---

## 10. 運用ルール

### 10.1 compile の責任者

初期責任者:

- `Strategy Lead`
- `Incident Analyst`

### 10.2 compile の頻度

初期方針:

- strategy 案の初期調査では research / plan に続いて compile してよい
- issue 完了ごとに必ず compile しなくてもよい
- 週次または incident / strategy milestone ごとに compile してよい

理由:

- wiki を event log にせず、かつ research workspace としては十分早く更新するため

### 10.3 lint

wiki structure の崩れを避けるため、定期的に以下を行う。

- `/wiki:lint --local`

必要なら:

- `/wiki:lint --fix --local`

---

## 11. 実装順

1. project-local `.wiki/` を作る
2. `Strategy Lead` が新規 strategy 案で `/wiki:research` と `/wiki:plan` を使う前提を入れる
3. minimal structure を `nvk/llm-wiki` 互換で作る
4. `Strategy Lead` と `Incident Analyst` だけに wiki 編集責務を持たせる
5. `Paperclip issue documents -> wiki` の接続手順を決める
6. compile / query / output のテンプレを固める

---

## 12. 未決事項

- `.wiki/` を repo 内に置くか、別 workspace に置くか
- `source_refs` frontmatter の最終スキーマ
- local wiki の article 分類をどこまで project-specific に寄せるか
- compile を人手主導にするか、限定自動化するか
- wiki を Git 管理するかどうか

---

## 13. 参照

- [paperclip-hermes-llm-wiki-if.md](D:/GitHub/quant-agent-platform/docs/paperclip-hermes-llm-wiki-if.md)
- [paperclip-hermes-runtime-contract.md](D:/GitHub/quant-agent-platform/docs/paperclip-hermes-runtime-contract.md)
- [paperclip-hermes-nautilus-design.md](D:/GitHub/quant-agent-platform/docs/paperclip-hermes-nautilus-design.md)
- https://github.com/nvk/llm-wiki/blob/master/README.md
- https://github.com/nvk/llm-wiki/blob/master/AGENTS.md
- https://github.com/nvk/llm-wiki/blob/master/claude-plugin/commands/plan.md
- https://github.com/nvk/llm-wiki/blob/master/claude-plugin/commands/wiki.md
