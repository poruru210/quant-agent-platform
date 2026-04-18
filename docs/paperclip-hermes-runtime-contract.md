# Paperclip ↔ Hermes Runtime Contract

## 1. 目的

本書は、`Paperclip` 上の agent / issue / heartbeat と、`Hermes` 実行系の接続契約を定義する。

狙い:

- 役職設計を `Paperclip` の実 IF に落とす
- `Hermes profile` の差分を `adapterConfig` と runtime policy に落とす
- issue / comment / document / wake reason が Hermes 実行へどう伝わるかを固定する

本書は「理想的な agent 組織図」ではなく、実装可能な runtime contract を対象とする。

---

## 2. 契約レイヤ

`Paperclip -> Hermes` の契約は、以下の 3 層に分かれる。

### 2.1 Layer A: Paperclip 共通契約

Paperclip が保証するもの:

- agent record:
  - `adapterType`
  - `adapterConfig`
  - `role`
  - `title`
  - `reportsTo`
- heartbeat:
  - wakeup queue
  - `queued / running / succeeded / failed / timed_out / cancelled`
- agent runtime env:
  - `PAPERCLIP_*`
- REST API:
  - issues
  - comments
  - documents
  - attachments
  - approvals

### 2.2 Layer B: `hermes_local` adapter type

Paperclip 側で見える Hermes 固有契約:

- `adapterType = "hermes_local"`
- adapter は `execute / testEnvironment / detectModel / listSkills / syncSkills / sessionCodec` を提供する
- session persistence は `sessionId` ベース

### 2.3 Layer C: Project-specific policy

このプロジェクトで追加定義するもの:

- role/title と Hermes 実行 policy の対応
- issue document key 命名
- 各 role の `cwd / toolsets / timeout / wake policy`
- live ノード、research ノード、state/control ノードへの配置
- compose と skill/transport の責務境界

重要:

- Layer A/B は既存 IF
- Layer C はこのプロジェクトの運用規約
- Layer C でも固定しすぎない

補足:

- このプロジェクトでは、Hermes / Paperclip が自律的に決められる事項を事前仕様にしすぎない
- 先に固定するのは hard IF、security boundary、node routing、document key などの接着面だけに留める
- 分析手順、DuckDB query、artifact の派生形式などは runtime で改善される余地を残す

---

## 3. Source Of Truth

### 3.1 短期オペレーション

短期の作業状態は `Paperclip` を source of truth とする。

対象:

- assignment
- checkout 状態
- status
- completion comment
- review / blocked / incident への遷移
- 現在 heartbeat で扱っている work item

### 3.2 長期知識

長期知識は 2 系統に分ける。

- `LLM-WIKI`
  - strategy 案
  - research / synthesis
  - planning
  - 再利用知識
- `Hindsight`
  - Hermes run history
  - 実行結果の振り返り
  - 自己改善用観測履歴

ただし runtime contract 上、Hermes がまず読むべき一次情報は `Paperclip issue + documents + comments` である。

### 3.3 Git revision

コード・戦略 revision の source of truth は Git commit SHA とする。

---

## 4. Agent Record Contract

### 4.1 Paperclip agent record の必須要素

各 Hermes-backed agent は少なくとも以下を持つ。

- `name`
- `role`
- `title`
- `reportsTo`
- `adapterType = "hermes_local"`
- `adapterConfig`

### 4.2 `role` と `title` の使い分け

推奨:

- `title` は人間に見せる役職名
- `role` は機械判定しやすい短い識別子

初期案:

- `strategy_lead`
- `quant_research_engineer`
- `shadow_trading_operator`
- `incident_analyst`

`title` は既存設計に合わせる。

- `Strategy Lead`
- `Quant Research Engineer`
- `Shadow Trading Operator`
- `Incident Analyst`

### 4.3 `adapterConfig` が担う責務

`adapterConfig` は少なくとも以下を持つ。

- 実行文脈:
  - `cwd`
- 実行制約:
  - `timeoutSec`
  - `graceSec`
  - `maxTurnsPerRun`
- capability:
  - `toolsets`
  - `env`
  - `extraArgs`
- 会話持続:
  - `persistSession`
- prompt:
  - `promptTemplate`

重要:

- `runtimeHost` は upstream の `hermes_local` 実 IF として確認済みの field ではない
- node 配置は `adapterConfig` の必須 key ではなく、project policy として別に管理する
- したがって runtime host の決定は本書の `Node Routing Contract` 側で扱う

---

## 5. Role → Runtime Policy 対応

### 5.1 `Strategy Lead`

Paperclip:

- `role = strategy_lead`
- `title = Strategy Lead`
- `adapterType = hermes_local`

Runtime policy:

- Hermes runtime host:
  - `pi-ctl`
- 論理的な主文脈:
  - state / control
- 主 `cwd`:
  - strategy docs / wiki workspace
- 推奨 `toolsets`:
  - `file,web,productivity`
- 禁止したいもの:
  - `terminal` の強い権限
  - live / deploy 系 skill
- `timeoutSec`:
  - 短め
- `maxTurnsPerRun`:
  - 中程度
- `persistSession`:
  - 有効

用途:

- brief 作成
- issue / incident / wiki 読解
- acceptance criteria 起案

### 5.2 `Quant Research Engineer`

Paperclip:

- `role = quant_research_engineer`
- `title = Quant Research Engineer`
- `adapterType = hermes_local`

Runtime policy:

- Hermes runtime host:
  - `pi-ctl`
- 論理的な主文脈:
  - research
- 主 `cwd`:
  - repo worktree
- 推奨 `toolsets`:
  - `terminal,file,web,code_execution`
- 許可する skill:
  - nautilus backtest
  - nautilus replay
  - feature generation
  - simulation
- `timeoutSec`:
  - 長め
- `maxTurnsPerRun`:
  - 高め
- `persistSession`:
  - 有効

用途:

- code write
- validation
- report 作成

### 5.3 `Shadow Trading Operator`

Paperclip:

- `role = shadow_trading_operator`
- `title = Shadow Trading Operator`
- `adapterType = hermes_local`

Runtime policy:

- Hermes runtime host:
  - `pi-ctl`
- remote target:
  - `pi-live`
- 論理的な主文脈:
  - live validation
- 主 `cwd`:
  - live deploy control workspace
- 推奨 `toolsets`:
  - `terminal,file`
- 許可する skill:
  - shadow start
  - shadow stop
  - shadow status
  - collect report
  - rollback
- 禁止:
  - research backtest
  - strategy code write
- `timeoutSec`:
  - 中程度
- `maxTurnsPerRun`:
  - 低めから中程度
- `persistSession`:
  - 有効

用途:

- paper / shadow 実行
- status / report / rollback

### 5.4 `Incident Analyst`

Paperclip:

- `role = incident_analyst`
- `title = Incident Analyst`
- `adapterType = hermes_local`

Runtime policy:

- Hermes runtime host:
  - `pi-ctl`
- 論理的な主文脈:
  - state / control
- 主 `cwd`:
  - incident docs / wiki workspace
- 推奨 `toolsets`:
  - `file,web,productivity`
- 必要なら限定 `terminal`
- 禁止:
  - deploy
  - code write
- `timeoutSec`:
  - 中程度
- `maxTurnsPerRun`:
  - 中程度
- `persistSession`:
  - 有効

用途:

- postmortem
- divergence 分析
- next iteration draft

---

## 6. Baseline `adapterConfig` 規約

### 6.1 共通必須項目

全 Hermes-backed agent に必須:

```json
{
  "model": "provider/model",
  "cwd": "/abs/path",
  "timeoutSec": 900,
  "graceSec": 10,
  "persistSession": true,
  "promptTemplate": "...",
  "toolsets": "file,web,productivity"
}
```

### 6.2 共通推奨項目

- `maxTurnsPerRun`
- `env`
- `extraArgs`
- `quiet`
- `paperclipApiUrl`

### 6.3 `toolsets` で分離すべきこと

role 差分の第一手段は `toolsets` にする。

原則:

- `Strategy Lead`
  - `file,web,productivity`
- `Incident Analyst`
  - `file,web,productivity`
- `Quant Research Engineer`
  - `terminal,file,web,code_execution`
- `Shadow Trading Operator`
  - `terminal,file`

### 6.4 `cwd` で分離すべきこと

`cwd` は node 配置と一致させる。

- strategy / incident:
  - state/control 側の docs / wiki workspace
- research:
  - repo worktree
- shadow:
  - live runtime workspace

重要:

- `live validation` 用 `cwd` と research 用 `cwd` を混在させない
- `shadow_operator` に repo write 権限を持たせない構成が望ましい

---

## 7. Wake Reason Contract

### 7.1 `assignment`

意味:

- 新規に自分へ割り当てられた issue

期待動作:

1. `GET /api/agents/me`
2. assignment 取得
3. 対象 issue checkout
4. issue / comments / documents 読み込み
5. 作業開始

### 7.2 `comment`

意味:

- comment mention による wake

期待動作:

1. comment thread を先に読む
2. 既存作業への補足か、新規 subtask かを判定
3. 必要なら返信 comment
4. 既存 issue documents の更新

### 7.3 `timer`

意味:

- 定期 heartbeat

期待動作:

1. `in_progress` を優先
2. なければ `todo`
3. truly idle なら軽い progress / check コメントのみ

### 7.4 `approval`

意味:

- approval 解決後の follow-up

期待動作:

- linked issues を見て close / continue / blocked を判定する

### 7.5 Project policy

本プロジェクトでは以下を採る。

- `Strategy Lead`
  - `assignment`, `comment`, `on_demand`
- `Quant Research Engineer`
  - `assignment`, `comment`, `on_demand`, 必要なら `timer`
- `Shadow Trading Operator`
  - `assignment`, `comment`, `on_demand`
- `Incident Analyst`
  - `assignment`, `comment`, `on_demand`

補足:

- live 系 agent に高頻度 timer を安易に入れない

---

## 8. Issue Artifact Contract

### 8.1 原則

role 間 baton pass は `issue` の title/body だけで完結させず、`documents/{key}` を使う。

理由:

- revision 管理できる
- completion comment と分離できる
- 長文を安定して保持できる

### 8.2 標準 document key

初期推奨:

- `brief`
  - strategy brief
- `acceptance`
  - acceptance criteria
- `implementation_note`
  - 実装方針
- `execution_request`
  - 実行依頼
- `execution_result`
  - skill 実行結果の最小要約
- `validation_report`
  - backtest / replay summary
- `shadow_status`
  - live validation の現在状態
- `shadow_report`
  - shadow deploy / stop / rollback summary
- `postmortem`
  - incident analysis
- `next_iteration`
  - 次サイクルへの論点

### 8.3 comment の役割

comment は以下に限定する。

- 進捗速報
- blocker 通知
- 完了要約
- mention による wake

長文レポート本体は document に置く。

### 8.4 attachment の役割

attachment は以下に使う。

- 生ログ
- 画像
- CSV
- 重い report artifact

document から attachment ref を指す形にする。

---

## 9. Prompt Contract

### 9.1 原則

prompt は:

- Paperclip heartbeat protocol を守らせる
- role 固有責務を強調する
- 禁止事項を明記する
- document key の読み書き先を固定する

ために使う。

### 9.2 Prompt が担うべき最低限

各 role の prompt には最低限以下を含める。

- 自分の role / title
- 許可された node / workspace
- 作業前に読むべきもの:
  - issue
  - comments
  - documents
- 更新すべきもの:
  - document key
  - completion comment
- 禁止事項:
  - deploy 禁止
  - code write 禁止
  - live 直接操作禁止
  - etc.

### 9.3 Prompt で持たせすぎないもの

以下は prompt に埋め込みすぎない。

- 大きな strategy knowledge
- incident の蓄積全文
- 過去全 run history

これらは issue documents / LLM-WIKI / attachments に逃がす。

---

## 10. Session Contract

### 10.1 基本

Hermes adapter は `sessionId` を保持し、次回 `--resume` する。

### 10.2 Session を維持してよい role

初期方針:

- `Strategy Lead`
  - 維持してよい
- `Quant Research Engineer`
  - 維持してよい
- `Shadow Trading Operator`
  - 維持してよいが drift に注意
- `Incident Analyst`
  - 維持してよい

### 10.3 Reset 条件

少なくとも以下で reset 候補:

- prompt policy の大変更
- role の責務変更
- 別戦略 ID へ跨る long-lived drift
- live operator が古い deploy 前提を引きずる
- repeated failure loop

### 10.4 Session key の扱い

session は agent 単位の持続であり、task 単位の完全分離ではない可能性がある。  
したがって cross-task contamination を避ける運用が必要である。

project policy:

- strategy ごと、もしくは incident cluster ごとの session reset ルールを後で追加する

---

## 11. Skills Contract

### 11.1 事実

Hermes adapter は skill snapshot を持つが、Hermes native skill は `~/.hermes/skills/` から読み込まれる。

### 11.2 実装上の含意

- Paperclip-managed skills だけでは閉じない
- `syncSkills` は Hermes 側では no-op 相当
- desired skill と installed native skill は分けて認識する必要がある

### 11.3 Project policy

skill 定義は 2 系統に分ける。

- company-managed:
  - このプロジェクトが明示的に使わせたい skill
- host-native:
  - 各ノードの `~/.hermes/skills/` に置かれる skill

初期ルール:

- role ごとの required skill 一覧を別紙で定義
- live ノードには最小 skill のみ
- research ノードには重い analysis / coding skill を許可

---

## 12. Node Routing Contract

### 12.1 原則

agent は role ごとに「Hermes runtime をどこへ置くか」と「どのノードを操作対象にするか」を分けて固定すべきである。

### 12.2 初期対応

- `Strategy Lead`
  - Hermes runtime host は `pi-ctl`
  - 主文脈は state / control
- `Quant Research Engineer`
  - Hermes runtime host は `pi-ctl`
  - `pi-research` は remote execution target
- `Shadow Trading Operator`
  - Hermes runtime host は `pi-ctl`
  - `pi-live` は remote execution target
- `Incident Analyst`
  - Hermes runtime host は `pi-ctl`
  - 主文脈は state / control

### 12.3 実装方法

runtime contract 上は、少なくとも以下のいずれかで実現する。

1. Paperclip server と同一ホストで local CLI 実行
2. Paperclip 側 adapter が remote Hermes runtime を起動または到達する
3. ノードごとに Paperclip runtime host を分ける

### 12.4 Skill / Transport / Isolation の分離

このプロジェクトでは、外から見える IF を以下の 3 層に分ける。

1. `Hermes skill`
2. `transport`
3. `remote isolation`

定義:

- skill:
  - Hermes が呼ぶ論理機能
  - 例: `nautilus-backtest`, `nautilus-replay`, `shadow-start`, `shadow-stop`
- transport:
  - Hermes 実行ノードから remote ノードへ到達する方法
  - 初期方針は `SSH`
- remote isolation:
  - remote ノード上で job / instance を隔離する方法
  - 初期方針は `docker`
  - 将来的に `microvm` へ差し替え可能

重要:

- user / Paperclip / 上位 role が意識する IF は `skill` だけにする
- `ssh` や `docker run` は skill の内部実装に留める
- 直接スクリプトを叩く運用を主 IF にしない

推奨:

- `state / control` の常駐系だけ compose で管理する
- Hermes runtime 自体は `pi-ctl` に置く
- `research` skill は `SSH transport` で `pi-research` に到達し、run 単位 container を起動する
- `live validation` skill は `SSH transport` で `pi-live` に到達し、instance 単位 container を起動する
- 初期 SSH 認証は LAN 内の password authentication でよい
- run result の truth は `pi-research` の `parquet` artifact とし、`DuckDB` は query layer に寄せる

理由:

- `NautilusTrader` の主語は長寿命 service より `nautilus-backtest` / `nautilus-replay` / `shadow-start` / `shadow-stop` に近い
- `hermes_local` の実 IF と素直に整合する
- `research` は複数同時 run が前提になりやすく、Hermes host と execution target を分けても artifact 回収で十分扱える
- `live validation` は strategy instance 単位の lifecycle が必要
- `pi-live` は Hermes host ではなく execution target に留めたほうが軽い

したがって contract としては「role ごとに node を跨がせない」に加え、「Nautilus 実行面は compose service として固定しない」「運用 IF は direct script ではなく Hermes skill とする」を先に固定する。

---

## 13. Security Boundary Contract

### 13.1 `Quant Research Engineer`

- live ノードへ触れない
- deploy skill を持たない

### 13.2 `Shadow Trading Operator`

- repo code write しない
- research backtest をしない
- `pi-live` への remote execution のみ許可する

### 13.3 `Strategy Lead`

- code write しない
- deploy しない

### 13.4 `Incident Analyst`

- deploy しない
- code write しない

重要:

- この境界は prompt だけでなく `toolsets / cwd / env / skill availability` で担保する

---

## 14. 既定の実装順

この contract を実装へ落とす順番は以下を推奨する。

1. `Strategy Lead` と `Quant Research Engineer` の 2 role だけ先に作る
2. issue document key を導入する
3. role 別 prompt template を導入する
4. node 別 `cwd` と skill / transport を分ける
5. `Shadow Trading Operator`
6. `Incident Analyst`

理由:

- 研究ループが先に動く
- live 系の前に control / report / validation の流れを固められる

---

## 15. 未決事項

- Paperclip 上で role ごとに別 host 実行する具体方式
- `cwd` と remote workspace の provisioning 方式
- strategy 単位の session reset 規約
- issue document key の最終命名
- approval を runtime contract にどこまで組み込むか
- research / live skill contract と SSH transport の厳密仕様

---

## 16. 参照

- [paperclip-hermes-llm-wiki-if.md](D:/GitHub/quant-agent-platform/docs/paperclip-hermes-llm-wiki-if.md)
- [paperclip-hermes-nautilus-design.md](D:/GitHub/quant-agent-platform/docs/paperclip-hermes-nautilus-design.md)
- [llm-wiki-operational-contract.md](D:/GitHub/quant-agent-platform/docs/llm-wiki-operational-contract.md)
- [research-result-storage-contract.md](D:/GitHub/quant-agent-platform/docs/research-result-storage-contract.md)
