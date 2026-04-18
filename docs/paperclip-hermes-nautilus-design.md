# Paperclip × Hermes × Nautilus 運用設計書（暫定）

## 1. 目的

本設計書は、Paperclip の company / employee モデルを前提に、Hermes profile をどのように Paperclip 上の役職へ割り当て、どのような baton pass で戦略立案・実装・研究検証・shadow/live 運用・incident 分析を回すかを定義する。

本設計では、Nautilus は **execution engine / tool** として扱う。  
主設計対象は以下とする。

- Paperclip 上の役職名
- Hermes profile との紐づけ
- role 間の呼び出し IF
- Hermes profile ごとの使用 skill
- 実装と検証の責務境界
- CEO / CTO による全体統括

---

## 2. 前提

### 2.1 実装前提

現行の Hermes adapter は、Paperclip から渡された task / comment / assignment を受け、Hermes を CLI で起動して実行するモデルである。  
したがって、role 間 IF の実体はまず **Paperclip issue / comment / assignment** である。

補足:

- 恒久方針は [SOUL.md](D:/GitHub/quant-agent-platform/SOUL.md) を参照
- `Paperclip -> Hermes` と `LLM-WIKI` の実 IF 調査結果は [paperclip-hermes-llm-wiki-if.md](D:/GitHub/quant-agent-platform/docs/paperclip-hermes-llm-wiki-if.md) を参照
- 本設計書は role / baton pass / node 配置を主対象とし、adapterConfig や wiki command sequence の詳細は別紙で詰める
- runtime 側の具体契約は [paperclip-hermes-runtime-contract.md](D:/GitHub/quant-agent-platform/docs/paperclip-hermes-runtime-contract.md) を参照
- wiki 側の具体運用は [llm-wiki-operational-contract.md](D:/GitHub/quant-agent-platform/docs/llm-wiki-operational-contract.md) を参照
- research result の保存 / `DuckDB` query layer は [research-result-storage-contract.md](D:/GitHub/quant-agent-platform/docs/research-result-storage-contract.md) を参照

### 2.2 設計原則

1. **Paperclip の role は役職名として定義する**  
   工程名ではなく、会社組織として自然な肩書きを使う。

2. **Hermes profile は role の実行主体である**  
   Paperclip が役職と指示を持ち、Hermes はそれを実行する employee である。

3. **Nautilus は主設計対象ではない**  
   Nautilus は Hermes が使う tool / engine であり、主たる IF は role↔profile↔handoff にある。

4. **実装と研究検証は同一 Hermes profile に持たせる**  
   Hermes のフィードバックループを生かすため、strategy implementation と research validation は分離しない。

5. **promotion と shadow/live 運用は別系統にする**  
   実装・研究ループと、本番寄りの deploy / rollback / monitoring は分離する。

6. **CEO / CTO が全体 control plane を担う**  
   task の切り出し、優先順位、昇格判断、incident escalation は CEO / CTO role が担う。

7. **戦略知と自己改善知は分ける**  
   strategy 案や hypothesis 形成は `LLM-WIKI` を主系とし、Hermes の自己改善は `Hindsight` を主系とする。

8. **compose は常駐系だけに使う**
   `Paperclip` や `PostgreSQL` のような state / control 常駐系には compose を使うが、`NautilusTrader` の research / live 実行は Hermes skill 経由の job / instance 起動を基本とする。

9. **直接スクリプトを運用 IF にしない**
   `ssh` や `docker run` は skill の内部実装としてのみ使い、上位の IF は `Hermes skill` として定義する。

10. **Hermes / Paperclip が自律的に決められることは事前固定しない**
   固定するのは hard IF、node 境界、安全境界、公開面、source of truth までに留める。分析粒度、DuckDB query、parquet 列設計、派生 export などは通常の開発のように先に細かく決めず、Hermes / Paperclip の改善ループの中で必要時に決めさせる。

---

## 3. 組織図

```text
CEO / CTO
├─ Strategy Lead
│   └─ Hermes profile: strategist
├─ Quant Research Engineer
│   └─ Hermes profile: research_engineer
├─ Shadow Trading Operator
│   └─ Hermes profile: shadow_operator
└─ Incident Analyst
    └─ Hermes profile: postmortem_analyst
```

補足:

- `CEO / CTO` は Paperclip 上の最上位統括 role
- Hermes profile は、Paperclip 上では agent employee として表現される
- `CEO / CTO` は human-first でもよい
- 必ずしも全 role が Hermes-backed である必要はない

---

## 4. Paperclip role/title と Hermes profile の紐づけ

### 4.1 CEO / CTO

- Paperclip role/title: `CEO` / `CTO`
- Hermes profile: なし（初期）
- 主責務:
  - 全体方針
  - 優先順位決定
  - 各 role への task 切り出し
  - promotion 可否判断
  - incident escalation

### 4.2 Strategy Lead

- Paperclip role/title: `Strategy Lead`
- Hermes profile: `strategist`
- 主責務:
  - strategy phase 初期の research / synthesis
  - 仮説定義
  - strategy brief 作成
  - 改善方向の整理
  - WIKI 更新

### 4.3 Quant Research Engineer

- Paperclip role/title: `Quant Research Engineer`
- Hermes profile: `research_engineer`
- 主責務:
  - strategy 実装
  - research 用 backtest / replay
  - 結果確認
  - 修正
  - 再実行
  - commit 作成
- 重要:
  - **実装と研究検証は同じ profile に持たせる**
  - Hermes の feedback loop を最大限に活用するため

### 4.4 Shadow Trading Operator

- Paperclip role/title: `Shadow Trading Operator`
- Hermes profile: `shadow_operator`
- 主責務:
  - approved revision の deploy
  - status / logs / report 回収
  - rollback

### 4.5 Incident Analyst

- Paperclip role/title: `Incident Analyst`
- Hermes profile: `postmortem_analyst`
- 主責務:
  - divergence / incident 分析
  - 次の改善案作成
  - strategist へのフィードバック

---

## 5. 責務境界

### 5.1 CEO / CTO

**やること**

- role 間の baton pass
- task の優先順位決定
- promotion 可否判断
- incident escalation
- shadow/live 継続判断

**やらないこと**

- strategy 実装
- Nautilus 実行
- live ノード直接操作

### 5.2 Strategy Lead / strategist

**やること**

- 仮説形成
- strategy brief 作成
- incident / WIKI の読み込み
- acceptance criteria 草案
- 改善方向の整理

**やらないこと**

- code write
- Nautilus 実行
- shadow/live deploy

### 5.3 Quant Research Engineer / research_engineer

**やること**

- code write
- Git worktree 上での strategy 実装
- research ノードでの backtest / replay
- 結果確認
- 修正と再実行
- commit SHA 作成
- report 返却

**やらないこと**

- promotion 最終判断
- shadow/live deploy
- incident 統括

### 5.4 Shadow Trading Operator / shadow_operator

**やること**

- approved revision の deploy
- status / report / rollback
- live/shadow ノードの限定操作

**やらないこと**

- strategy code write
- research backtest
- 仮説変更

### 5.5 Incident Analyst / postmortem_analyst

**やること**

- divergence / incident 読解
- postmortem 作成
- 次の改善案作成
- strategist への論点返却

**やらないこと**

- code write
- deploy
- promotion 判断

---

## 6. role 間の呼び出し IF

### 6.1 基本方針

role 間の呼び出しは、まず **Paperclip issue / comment / assignment** により行う。

共通フォーマット:

- issue title
- issue body
- assignee
- expected output
- refs
- completion comment

---

### 6.2 CEO / CTO → Strategy Lead

#### IF 名

`strategy_brief_issue`

#### 目的

新規戦略または既存戦略改善の方針を Strategy Lead に渡す。

#### 入力

- initiative / objective
- 優先度
- 制約
- 関連 WIKI refs
- 関連 incident refs
- 期待成果物

#### 出力

- strategy brief
- hypothesis
- acceptance criteria 草案
- 次に Quant Research Engineer へ渡すべき内容

---

### 6.3 Strategy Lead → Quant Research Engineer

#### IF 名

`research_execution_issue`

#### 目的

仮説を実装・研究検証タスクに落とし込む。

#### 入力

- strategy_id
- strategy brief
- 仮説
- 改善対象
- acceptance criteria 草案
- WIKI refs
- incident refs

#### 出力

- commit SHA
- module/class
- backtest/replay summary
- report refs
- self-assessment
- promotion 候補か否か

---

### 6.4 Quant Research Engineer → CEO / CTO

#### IF 名

`research_result_review`

#### 目的

研究結果を統括者へ返し、次アクションを決める。

#### 入力

- commit SHA
- 実装概要
- validation summary
- report refs
- リスク
- 推奨アクション

#### 出力

- revise
- approve_for_shadow
- reject
- assign_to_incident_review

---

### 6.5 CEO / CTO → Shadow Trading Operator

#### IF 名

`shadow_deploy_issue`

#### 目的

approved revision を shadow/live に投入する。

#### 入力

- approved commit SHA
- strategy identifier
- deploy mode
- params ref
- rollback 条件
- 監視ポイント

#### 出力

- deploy 完了
- current status
- report ref
- incident 有無

---

### 6.6 CEO / CTO → Incident Analyst

#### IF 名

`incident_review_issue`

#### 目的

divergence / incident の読解を依頼する。

#### 入力

- incident summary
- divergence note
- result refs
- 関連 strategy / commit
- 見てほしい論点

#### 出力

- postmortem
- next experiment suggestions
- strategist に渡すべき要点

---

### 6.7 Incident Analyst → Strategy Lead

#### IF 名

`next_iteration_brief`

#### 目的

incident 分析結果を次の戦略仮説へ接続する。

#### 入力

- incident conclusions
- 改善候補
- 追加仮説
- WIKI update suggestions

#### 出力

- updated strategy brief
- 新規 `research_execution_issue` 草案

---

## 7. Skill を誰が呼ぶか

### 7.1 Strategy Lead

- 呼ばない

### 7.2 Quant Research Engineer

- 呼ぶ
- 例:
  - `nautilus-backtest`
  - `nautilus-replay`
- research ノードのみ操作

### 7.3 Shadow Trading Operator

- 呼ぶ
- 例:
  - `deploy_shadow`
  - `status_shadow`
  - `collect_shadow_report`
  - `rollback_shadow`
- live/shadow ノードのみ操作

### 7.4 Incident Analyst

- 原則呼ばない
- 必要なら report 収集系のみ限定

### 7.5 CEO / CTO

- 原則呼ばない
- workflow control に専念

---

## 8. Hermes profile ごとの使用 SKILL

### 8.1 strategist

#### 目的

仮説形成、strategy brief 作成、WIKI 更新

#### 使用 SKILL

- `llm-wiki-read`
- `llm-wiki-write`
- `strategy-brief-template`
- `incident-context-read`
- `artifact-ref-read`
- `paperclip-issue-reply`

#### 主な toolset

- file
- web
- productivity
- terminal（軽め）

#### 禁止・制限

- code write しない
- SSH しない
- Nautilus 実行しない

---

### 8.2 research_engineer

#### 目的

strategy 実装、research 検証、修正ループ

#### 使用 SKILL

- `research-runbook`
- `strategy-code-edit`
- `git-worktree-flow`
- `ssh-research-node`
- `nautilus-backtest`
- `nautilus-replay`
- `parse-validation-report`
- `paperclip-result-report`

#### 主な toolset

- terminal
- file
- git
- code_execution

#### 禁止・制限

- live/shadow ノードへは触れない
- promotion 判定しない

---

### 8.3 shadow_operator

#### 目的

approved revision の deploy と監視

#### 使用 SKILL

- `deploy-shadow-runbook`
- `shadow-start`
- `shadow-stop`
- `shadow-status`
- `collect-shadow-report`
- `status-shadow-runbook`
- `rollback-runbook`
- `paperclip-deploy-report`

#### 主な toolset

- terminal
- file

#### 禁止・制限

- strategy code write しない
- research backtest をしない

---

### 8.4 postmortem_analyst

#### 目的

incident / divergence の分析

#### 使用 SKILL

- `incident-analysis-template`
- `llm-wiki-read`
- `compare-validation-vs-shadow`
- `next-experiment-draft`
- `paperclip-incident-report`

#### 主な toolset

- file
- web
- terminal（限定）

#### 禁止・制限

- code write しない
- deploy しない

---

## 9. 運用サイクル

### 9.1 新規戦略・改善サイクル

1. CEO / CTO が `strategy_brief_issue` を Strategy Lead に渡す
2. Strategy Lead が仮説と brief を作成する
3. CEO / CTO が `research_execution_issue` を Quant Research Engineer に渡す
4. Quant Research Engineer が実装 + research 検証を回す
5. 結果を CEO / CTO に返す
6. CEO / CTO が以下を判断する
   - 差し戻し
   - shadow へ進める
   - incident review に回す

### 9.2 Shadow / Live サイクル

1. CEO / CTO が `shadow_deploy_issue` を Shadow Trading Operator に渡す
2. Shadow Trading Operator が deploy を行う
3. status / report を返す
4. CEO / CTO が継続 or rollback を判断する

### 9.3 Incident サイクル

1. CEO / CTO が `incident_review_issue` を Incident Analyst に渡す
2. Incident Analyst が postmortem を返す
3. Strategy Lead が次の仮説へ反映する

---

## 10. WIKI / Hindsight / Git / Nautilus の位置づけ

### 10.1 WIKI

WIKI は以下のために使う。

- strategy 案の初期調査
- strategy rationale
- lessons learned
- hypothesis history
- decision context

補足:

- `LLM-WIKI` は strategy / research / planning の知識面を担う
- Hermes の自己改善ログの主系ではない

### 10.2 Hindsight

Hindsight は以下のために使う。

- Hermes run history
- 実行結果の振り返り
- 自己改善用の観測履歴
- operational hindsight

補足:

- Hermes の自己改善は `Hindsight` を主系にする
- strategy 案の初期調査は `LLM-WIKI` を主系にする

### 10.3 Git

Git は以下のために使う。

- strategy code
- commit SHA による revision 固定
- worktree ベースの Hermes 実装
- diff / rollback / reproducibility

### 10.4 Nautilus

Nautilus は以下のために使う。

- backtest
- replay
- shadow/live execution

重要:

- Nautilus は tool / engine である
- 主要 IF は Nautilus ではなく role↔profile 間にある

---

## 11. 設計原則（要約）

1. Paperclip の role は **役職名**で表現する
2. Hermes profile はその role の **実行者**である
3. 実装と研究検証は `Quant Research Engineer` にまとめる
4. `Strategy Lead` は仮説担当であり、実装しない
5. `Shadow Trading Operator` は approved revision のみ扱う
6. `CEO / CTO` が全体の control plane となる
7. role 間 IF はまず Paperclip issue / comment で定義する
8. Nautilus は execution engine であり、主設計対象は role↔profile↔handoff である

---

## 12. 最小構成

補足:

- strategy 知識の主系は `LLM-WIKI`
- Hermes の自己改善知の主系は `Hindsight`

### Paperclip role/title

- `CEO` / `CTO`
- `Strategy Lead`
- `Quant Research Engineer`
- `Shadow Trading Operator`
- `Incident Analyst`

### Hermes profile

- `strategist`
- `research_engineer`
- `shadow_operator`
- `postmortem_analyst`

---

## 13. 配置・公開構成

### 13.1 外部公開経路

Paperclip の人間向け UI は、以下の経路で公開する。

```text
[Browser]
   |
   v
https://paperclip.example.com
   |
   v
[Cloudflare Access]
   |
   v
[Cloudflare Tunnel]
   |
   v
[Raspberry Pi]
  ├─ Paperclip Web UI   (127.0.0.1:3000 など)
  ├─ Paperclip API      (必要に応じて同一ホスト内)
  └─ PostgreSQL         (外部公開しない)
```

重要:

- 外部から直接到達できる入口は `Cloudflare Access` のみとする
- `Paperclip Web UI` と `Paperclip API` は原則 `127.0.0.1` bind とし、`cloudflared` からローカル転送する
- `PostgreSQL` はインターネット非公開とし、少なくとも public bind しない
- 家庭内ルータや拠点側 FW での inbound port 開放を前提にしない

### 13.2 公開面の分離方針

`Paperclip Web UI` と `Paperclip API` は論理的に分離して扱う。

推奨:

- 人間向け UI: `paperclip.example.com`
- API / automation 向け: `api.paperclip.example.com`

補足:

- 初期は同一ホスト・同一 Pi 上でよい
- ただし Access policy は UI と API で分ける
- UI は human login 前提、API は service token / machine identity 前提が扱いやすい

### 13.3 制御系と実行系の分離

本設計では、Pi 上の役割を **state / control plane**、**research execution plane**、**live validation plane** に分ける。

- state / control plane:
  - Paperclip native Web UI
  - Paperclip native API
  - PostgreSQL container
  - Hindsight / 履歴管理
  - メタデータ保存
  - 結果索引
  - 監視基盤
  - Hermes native runtime
  - Hermes adapter の制御入口
  - Cloudflare Tunnel
- research execution plane:
  - `research_engineer` による実装 / backtest / replay
  - NautilusTrader
  - alpha ツール群
  - 特徴量生成
  - シミュレーション
  - 再計算ジョブ
  - run artifact
  - DuckDB query layer
- live validation plane:
  - `shadow_operator` が `pi-ctl` から操作する deploy / status / rollback target
  - NautilusTrader の live engine
  - 市場データ受信
  - paper / shadow 実行
  - リアルタイム signal validation
  - 注文 / シグナル / イベントのローカルスプール
  - 軽量なリスクガード

設計意図:

- 外部公開が必要なのは state / control plane のみ
- research と live を別ノードへ分離し、相互に負荷影響させない
- live validation は 8GB 機でも、他の重い処理と混在させなければ十分に現実的である
- role 間 IF は引き続き Paperclip issue / comment / assignment を中心に扱う
- compose は `pi-ctl` の DB に限定し、Paperclip / Hermes は native 配置する
- `pi-research` / `pi-live` はどちらも SSH 経由の execution target にする
- research result の truth は `pi-research` の `parquet` artifact とし、`DuckDB` は query layer に寄せる

### 13.4 機材割り当て（初期推奨）

利用可能機材:

- Raspberry Pi 5 / RAM 16GB / SSD 1TB x 2
- Raspberry Pi 5 / RAM 8GB / SSD 1TB x 1

初期推奨の割り当て:

#### `pi-research`: Research ノード（16GB）

- `research_engineer`
- Hermes の研究 / レビュー / 検証ワークフロー
- NautilusTrader による `backtest` / `replay`
- alpha ツール群
- 特徴量生成
- シミュレーション
- 再計算ジョブ
- Git worktree
- validation report 生成
- run artifact 保存
- DuckDB query layer

理由:

- CPU / RAM / SSD I/O を強く使う重い研究処理をここへ隔離する
- このノードは止まっても live validation に直接影響させない
- 実装と研究検証を同一 profile にまとめる設計と相性がよい

#### `pi-ctl`: State / Control ノード（16GB）

- `Paperclip Web UI`
- `Paperclip API`
- `PostgreSQL` container
- `Hindsight`
- 履歴管理
- メタデータ保存
- 結果索引
- 監視基盤
- `cloudflared`
- Hermes native runtime
- Hermes adapter の制御入口

理由:

- ここは system の「頭脳と台帳」であり、研究ノードや live ノードの負荷から切り離す
- stateful な DB と管理 UI 群を余裕のある 16GB ノードに集約できる
- Cloudflare Tunnel の着地点を 1 台に固定できる

#### `pi-live`: Live Validation ノード（8GB）

- `shadow_operator`
- NautilusTrader の live engine
- 市場データ受信
- paper / shadow 実行
- 実 tick / 実 spread 収集
- DuckDB query layer
- リアルタイム signal validation
- 注文 / シグナル / イベントのローカルスプール
- 軽量なリスクガード

理由:

- 8GB でも、銘柄数や venue 数が極端でなければ live validation 専用ノードとして十分現実的である
- 大事なのはメモリ量より、他の重い処理と混在させないことである
- paper / shadow 実行を研究処理から分離し、安定性を優先できる

### 13.5 Role と配置の対応

- `CEO / CTO`
  - Browser から `Cloudflare Access` 経由で Paperclip を操作する
- `Strategy Lead`
  - 主に Paperclip 上で brief / hypothesis を作る
  - source of truth は state / control であり、Hermes-backed にする場合の runtime host も `pi-ctl`
- `Quant Research Engineer`
  - Hermes runtime host は `pi-ctl`
  - `pi-research` が backtest / replay / simulation の execution target
- `Shadow Trading Operator`
  - Hermes runtime host は `pi-ctl`
  - `pi-live` は approved revision の paper / shadow 実行 target
- `Incident Analyst`
  - source of truth は `pi-ctl`
  - Hermes-backed にする場合の runtime host も `pi-ctl`
  - 履歴、結果索引、report artifact を参照して incident 分析を担う

### 13.6 この構成で明確になること

- 外部公開対象は `Paperclip` を含む state / control plane のみ
- `PostgreSQL` は非公開の内部 state store として扱い、`pi-ctl` 上では container で動かす
- Hermes profile は役職に対応する execution actor だが、同一ホストで動かす必要はない
- 初期構成では Paperclip / Hermes は `pi-ctl` に native 配置する
- Nautilus は research plane と live validation plane の双方で使う tool / engine である
- live validation ノードは「軽くて安定した専用機」として扱う
- `NautilusTrader` は compose service として固定せず、run / instance 単位の lifecycle を持たせる
- `ssh` や `docker run` は見える IF ではなく skill の内側へ隠蔽する
- `pi-research` / `pi-live` では raw result を `parquet` に残し、Hermes が `DuckDB` query を確認してから `LLM-WIKI` へ知見を戻す

### 13.7 初期構成の弱点と補強ポイント

弱点:

- `pi-ctl` 障害時は `Paperclip UI / API / PostgreSQL / 履歴管理` を同時に失う
- state / control plane が単一障害点になる
- `pi-live` は live 専用にするぶん、障害時の影響が paper / shadow 実行へ直結する

初期から入れておきたい補強:

- PostgreSQL の定期バックアップを `pi-research` または外部バックアップ先へ退避する
- Paperclip の設定 / artifact metadata / WIKI のバックアップ方針を決める
- `cloudflared` 設定と Access policy を IaC 的に管理できる形へ寄せる
- API 用の認証方式を UI 用と分離する
- `pi-live` のローカルスプールが `pi-ctl` 側へ非同期転送されるようにし、state / control 側の瞬断で live を止めない
- `pi-live` 上のリスクガードは軽量かつ fail-closed に設計する

将来拡張:

- `pi-research` を PostgreSQL warm standby / replica 候補にする
- API と UI の host 名を完全分離する
- shadow と live を別ノードへさらに分ける

---

## 14. 未決事項

以下は今後、実コードと運用に合わせて詰める。

- Paperclip issue template 本文
- 各 Hermes profile の skill 実装詳細
- research / live skill contract と SSH transport の具体仕様
- WIKI の実ソフトウェア機能に応じた metadata 表現方法
- CEO / CTO を human-first にするか、限定的 executive profile を置くか
- `paperclip.example.com` と API 用 host 名の切り分け
- Cloudflare Access policy と service token policy の具体設計
- `pi-research` 障害時の復旧手順と PostgreSQL バックアップ / replica 方針
