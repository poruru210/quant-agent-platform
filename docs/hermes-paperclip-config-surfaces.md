# Hermes / Paperclip Config Surfaces

本書は、`Hermes` と `Paperclip` で **Agent / Role の挙動を左右する設定面** を棚卸しするための inventory である。  
目的は、先に固定すべき hard IF と、runtime に委ねてよい領域を切り分けることにある。

注意:

- ここでは「何が設定面として存在するか」を優先して整理する
- ここで列挙するすべてを事前固定すべき、という意味ではない
- `SOUL.md` のように人間が与えるべき恒久方針と、`AGENTS.md` のような project context を分けて考える

---

## 1. 結論

挙動を決める設定面は、大きく分けると次の 2 系統である。

### 1.1 Hermes 側

- instance-global identity
  - `SOUL.md`
- instance-global memory
  - `MEMORY.md`
  - `USER.md`
- instance-global skills
  - `~/.hermes/skills/`
- instance/global config
  - `config.yaml`
  - personality / clarification / skill directories など
- project context
  - `.hermes.md` / `HERMES.md`
  - `AGENTS.md`
  - `CLAUDE.md`
  - `.cursorrules`
  - `.cursor/rules/*.mdc`
- session overlay
  - `/personality`
  - ephemeral system prompt 系

### 1.2 Paperclip 側

- company / org structure
- agent record
  - `name`
  - `role`
  - `title`
  - `reportsTo`
  - `capabilities`
  - `budget`
  - `adapterType`
  - `adapterConfig`
- heartbeat settings
- issue / comment / document / attachment / approval
- board override / budget / pause / resume / terminate

---

## 2. Hermes の設定面

### 2.1 `SOUL.md`

用途:

- Hermes の primary identity
- tone / style / personality
- ambiguity や disagreement への default posture

現行 docs で強い記述:

- `SOUL.md` は agent identity の slot #1
- location は `HERMES_HOME/SOUL.md`
- global / instance-level の durable identity として使う

重要:

- 新しい docs 群では **global / instance-level SOUL** が主記述である
- 一方、古い personality page には **project-level `./SOUL.md`** と `config.yaml` personas の記述も残っている
- 現時点では、**global-only を canonical と見なすのが安全**で、project-specific 指示は `AGENTS.md` 側に寄せるべき

このプロジェクトでの扱い:

- `SOUL.md` は人間が与える恒久方針
- repo ごとの設計指示は入れない

### 2.2 `AGENTS.md` と project context files

用途:

- repo structure
- coding conventions
- project workflow
- ports / paths / forbidden actions

公式 docs 上の優先順位:

- `.hermes.md` / `HERMES.md`
- `AGENTS.md`
- `CLAUDE.md`
- `.cursorrules`

補足:

- startup 時は first-match-win
- subdirectory では `AGENTS.md` が progressive discovery される

このプロジェクトでの扱い:

- repo / subdirectory ごとの instructions はここ
- `SOUL.md` に project 固有情報を入れない

### 2.3 `MEMORY.md` / `USER.md`

用途:

- environment facts
- learned conventions
- user preferences

特徴:

- `~/.hermes/memories/` に保存
- session start 時に inject
- Hermes 自身が memory tool で更新できる

このプロジェクトでの扱い:

- 人間が初期値を細かく設計するより、Hermes に運用させる側

### 2.4 skills

用途:

- on-demand knowledge / workflow modules

設定面:

- `~/.hermes/skills/`
- bundled skills
- optional skills
- external skill directories

このプロジェクトで重要なこと:

- 既存 Hermes skills
- project-specific new skills
- host-native skill install policy

### 2.5 `config.yaml` / session overlays

公式 docs で見える設定面:

- clarification timeout
- context file behavior
- skills config
- built-in personalities
- `/personality` overlay

注意:

- personality docs には `config.yaml` custom personas や `agent.system_prompt` の記述が残る
- ただし current canonical docs は `SOUL.md` を primary identity と強調している
- ここは version drift があるため、実装確認前に過度に依存しないほうがよい

---

## 3. Paperclip の設定面

詳細な instructions model は [paperclip-agent-instructions-model.md](/D:/GitHub/quant-agent-platform/docs/paperclip-agent-instructions-model.md:1) を参照。

### 3.1 company / org

設定面:

- company goal
- org chart
- company budget

これは agent 単位ではなく、会社全体の統治面を決める。

### 3.2 agent record

公式 docs で確認できる主項目:

- `name`
- `role`
- `title`
- `reportsTo`
- `capabilities`
- `adapterType`
- `adapterConfig`
- `budgetMonthlyCents`
- status

source まで見ると、実際には以下も重要である。

- `runtimeConfig`
- `metadata`

補足:

- `capabilities` は存在するが、これは短い責務説明であり、`Hermes` の `SOUL.md` のような恒久人格と同一視しないほうがよい
- `adapterConfig.promptTemplate` は実在するが、source 上では legacy prompt 扱いである
- より新しい正規形は managed `instructions bundle` で、default entry は `AGENTS.md`

### 3.2.1 instructions bundle

Paperclip には、agent record とは別に instructions bundle がある。

確認できる要素:

- `mode`
  - `managed` / `external`
- `rootPath`
- `entryFile`
- `managedRootPath`
- `resolvedEntryPath`
- `files`

実装上の key:

- `instructionsBundleMode`
- `instructionsRootPath`
- `instructionsEntryFile`
- `instructionsFilePath`
- `agentsMdPath`

重要:

- `promptTemplate` だけが人格定義の場ではない
- 現行の設計は、Agent ごとの instructions root と `AGENTS.md` を軸にしている
- したがって、Paperclip の role-specific guidance は instructions bundle に寄せて考えるべき

このプロジェクトで重要なのは:

- `role`
- `title`
- `reportsTo`
- `adapterType = hermes_local`
- `adapterConfig`

### 3.3 heartbeat settings

board operator docs で見える設定面:

- interval
- cooldown
- max concurrent runs
- wake triggers

これは role の性格をかなり変えるので、人間が先に触る設定面である。

### 3.4 issue / document / attachment / approval

挙動面で重要な設定面:

- issue lifecycle
- document key 設計
- approval gate
- budget gate

このプロジェクトでは `documents/{key}` が role 間 IF の中心になる。

### 3.5 adapter

Paperclip が agent runtime に渡す大きな設定面は:

- `adapterType`
- `adapterConfig`

注意:

- `adapterConfig` の中身は adapter ごとに違う
- `process` adapter のように docs 化された schema もある
- しかし `hermes_local` の config schema は、Paperclip docs だけでは十分に明文化されていない

したがって:

- `hermes_local` の実 config 項目は adapter repo / implementation を見て詰める必要がある

---

## 4. ベストプラクティス

### 4.1 Hermes

- `SOUL.md` は stable に保つ
- project 指示は `AGENTS.md` に寄せる
- subdirectory `AGENTS.md` を活用する
- memory や query の細部は Hermes に委ねる
- bundled / optional / project skill を分けて管理する

### 4.2 Paperclip

- role / title / reportsTo は人間が明示する
- issue / document / approval を hard IF として設計する
- adapterConfig に project wish-list を混ぜすぎない
- heartbeat settings は role 単位に分ける

### 4.3 このプロジェクトで先に固定すべきもの

- `SOUL.md` の単位と責務
- `AGENTS.md` の責務
- `Paperclip` の role/title/reportsTo
- document key
- approval boundary
- skill naming

### 4.4 先に固定しすぎないもの

- DuckDB query の細部
- parquet の列設計
- runtime の分析フロー
- summary の粒度

---

## 5. リポジトリ上の分け方の示唆

最終設計では、少なくとも以下を分けるのが自然である。

- `ops/hermes/instance/`
  - `SOUL.md` 由来の恒久方針
  - global Hermes config inventory
- `ops/hermes/project/`
  - `AGENTS.md` 設計
  - subdirectory context strategy
- `ops/hermes/profiles/`
  - role ごとの personality / prompt / skill policy
- `ops/paperclip/agents/`
  - agent record templates
  - adapterConfig templates
- `ops/paperclip/governance/`
  - approval / budget / wake policies

ここではまだ最終ディレクトリ案を固定しないが、**Hermes instance-level** と **Paperclip org-level** は分離して管理したほうがよい。

---

## 6. いま次に見るべきもの

優先順位としては次が自然である。

1. `SOUL.md`
2. `AGENTS.md`
3. `Paperclip` の agent record / heartbeat / approval
4. `hermes_local` adapter の実 config 項目
5. skill install / host-native skill policy

---

## 7. Sources

Hermes official:

- https://hermes-agent.nousresearch.com/docs/guides/use-soul-with-hermes/
- https://hermes-agent.nousresearch.com/docs/user-guide/features/personality/
- https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files
- https://hermes-agent.nousresearch.com/docs/user-guide/features/memory/
- https://hermes-agent.nousresearch.com/docs/user-guide/features/skills
- https://hermes-agent.nousresearch.com/docs/reference/skills-catalog/
- https://hermes-agent.nousresearch.com/docs/user-guide/configuration/

Paperclip official:

- https://docs.paperclip.ing/start/core-concepts
- https://docs.paperclip.ing/start/architecture
- https://docs.paperclip.ing/adapters/overview
- https://docs.paperclip.ing/guides/agent-developer/heartbeat-protocol
- https://docs.paperclip.ing/guides/board-operator/managing-agents
- https://docs.paperclip.ing/api/agents
- https://docs.paperclip.ing/api/issues
- https://docs.paperclip.ing/guides/board-operator/approvals
- https://docs.paperclip.ing/api/authentication
