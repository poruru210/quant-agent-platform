# Paperclip / Hermes / LLM-WIKI IF 調査メモ

## 1. 目的

この文書は、運用設計を理論先行で広げるのではなく、既存ソフトウェアが実際に持っている IF に設計を合わせるための確認メモである。

前提:

- `Paperclip` は control plane
- `Hermes` は Paperclip から起動される agent runtime
- `LLM-WIKI` は strategy / incident / lessons learned を蓄積する候補
- `Hindsight` は Hermes の自己改善に関わる主系候補

重要:

- まず乗るべきなのは `Paperclip` の adapter / heartbeat / REST API 契約である
- `Hermes` については `Paperclip` 本体と `hermes-paperclip-adapter` の責務を分けて考える
- `LLM-WIKI` については、存在しない専用 API を仮定しない
- `Hindsight` については、自己改善ループの主系である前提を置くが、IF は未調査なので別紙で確認が必要

---

## 2. 結論サマリ

### 2.1 Paperclip → Hermes の IF は存在するか

存在する。  
ただし、1 枚岩の単一 IF ではなく、以下の 3 層に分かれている。

1. `Paperclip` の agent / adapter / heartbeat 契約
2. `hermes_local` adapter type の登録と runtime 呼び出し
3. `NousResearch/hermes-paperclip-adapter` が定義する `adapterConfig` / CLI 引数 / session / skill sync 契約

したがって、設計は `Paperclip` 一般 IF と `Hermes` 固有 IF を分離して持つべきである。

### 2.2 LLM-WIKI の IF は存在するか

存在する。  
ただし、主 IF は Web API ではなく **command + file layout** である。

つまり、`LLM-WIKI にノウハウを貯める` とは:

- `/wiki:ingest`
- `/wiki:compile`
- `/wiki:query`
- `/wiki:plan`
- `/wiki:output`

のようなコマンドを使うか、もしくはその前提となる wiki ディレクトリ構造に沿ってファイルを置くことを意味する。

`LLM-WIKI metadata API` のようなものを前提にした設計は、現時点では避けるべきである。

補足:

- 元の `nvk/llm-wiki` の README / AGENTS / command 定義を見ると、`LLM-WIKI` は後段の保管庫というより **research / synthesis / planning の初期フェーズから使う system** として設計されている
- 特に `/wiki:research` と `/wiki:plan` は、strategy 案のタイミングで使うのが自然である

---

## 3. Paperclip 側で確認できた IF

### 3.1 Adapter の基本契約

Paperclip の heartbeat 実行時、server は agent の `adapterType` と `adapterConfig` を見て adapter を起動する。

確認できたこと:

- agent は `adapterType` と `adapterConfig` を持つ
- heartbeat 時に adapter の `execute()` が呼ばれる
- adapter は agent runtime を spawn / call する
- adapter は stdout / usage / cost / session state を返す

意味:

- `Paperclip -> Hermes` は「issue を直接 Hermes に投げる」のではなく、まず adapter 実行契約に従って起動される
- よって設計の第一級 IF は `role 間 baton pass` ではなく、Paperclip の adapter / heartbeat 契約である

### 3.2 Agent runtime の heartbeat 契約

Paperclip agent は heartbeat ごとに REST API を叩いて仕事を進める。

確認できたこと:

- agent は `GET /api/agents/me` で自分の identity を得る
- `GET /api/companies/{companyId}/issues?...` で assignment を取得する
- `POST /api/issues/{issueId}/checkout` で atomic checkout する
- `GET /api/issues/{issueId}` と `GET /api/issues/{issueId}/comments` で文脈を読む
- `PATCH /api/issues/{issueId}` で status 更新と comment 追加を行う
- `POST /api/companies/{companyId}/issues` で subtask を委譲できる

意味:

- 実際の work protocol は、現行設計書にある通り `Paperclip issue / comment / assignment` が中核でよい
- ただしこれは「業務上の IF」であり、実装上は heartbeat REST protocol に落ちる

### 3.3 Agent process に inject される環境変数

Paperclip は agent runtime に以下を inject する。

- `PAPERCLIP_AGENT_ID`
- `PAPERCLIP_COMPANY_ID`
- `PAPERCLIP_API_URL`
- `PAPERCLIP_API_KEY`
- `PAPERCLIP_RUN_ID`
- `PAPERCLIP_TASK_ID`
- `PAPERCLIP_WAKE_REASON`
- `PAPERCLIP_WAKE_COMMENT_ID`
- `PAPERCLIP_APPROVAL_ID`
- `PAPERCLIP_APPROVAL_STATUS`
- `PAPERCLIP_LINKED_ISSUE_IDS`

意味:

- Hermes 側で task / comment / approval を判定する根拠は、issue body を自由解釈することではなく、まずこの injected runtime context に置くべき

### 3.4 Issue documents / comments / attachments

Paperclip issue は単なる task ではなく、文書と添付を持てる。

確認できたこと:

- keyed text documents を持てる
- `PUT /api/issues/{issueId}/documents/{key}` で revisioned text artifact を保存できる
- comments の `@mention` は agent wake を起こせる
- attachments をアップロードできる

意味:

- `strategy brief`
- `validation report`
- `postmortem`
- `deploy note`

のような中間成果物は、まず `Paperclip issue documents` に載せるのが自然
- `LLM-WIKI` は長期知識の蓄積先として分けたほうが整理しやすい

### 3.5 `hermes_local` は実在するか

実在する。  
確認できたこと:

- docs 上で built-in adapter として列挙されている
- `server/src/adapters/builtin-adapter-types.ts` に `hermes_local` が含まれている
- `ui/src/adapters/hermes-local/index.ts` が `hermes-paperclip-adapter/ui` を参照している

意味:

- `Paperclip が Hermes を built-in adapter type として扱う` 前提は妥当
- ただし実装のかなりの部分は `hermes-paperclip-adapter` パッケージ側に委譲されている

---

## 4. Hermes adapter 側で確認できた IF

調査対象:

- `NousResearch/hermes-paperclip-adapter`

### 4.1 Adapter の実体

この adapter は `ServerAdapterModule` を実装し、少なくとも以下を提供する。

- `execute`
- `testEnvironment`
- `detectModel`
- `listSkills`
- `syncSkills`
- `sessionCodec`

意味:

- `Paperclip -> Hermes` の固有 IF は「CLI を起動するだけ」ではない
- model detection、skill snapshot、session codec も IF の一部である

### 4.2 `adapterConfig` と CLI 引数

README と `src/server/execute.ts` から、少なくとも以下の config 項目が確認できた。

- `model`
- `provider`
- `timeoutSec`
- `graceSec`
- `toolsets`
- `persistSession`
- `worktreeMode`
- `checkpoints`
- `hermesCommand`
- `verbose`
- `quiet`
- `extraArgs`
- `env`
- `promptTemplate`
- `paperclipApiUrl`
- `maxTurnsPerRun`

また、CLI 呼び出しは概ね以下に対応している。

- `hermes chat -q <prompt>`
- `-Q`
- `-m`
- `--provider`
- `-t`
- `--max-turns`
- `-w`
- `--checkpoints`
- `-v`
- `--source tool`
- `--yolo`
- `--resume <sessionId>`

意味:

- `Hermes profile` 設計は、この `adapterConfig` と CLI フラグに落ちる形で定義すべき
- `research_engineer` と `shadow_operator` の差は、role 名だけでなく `toolsets` / `cwd` / `extraArgs` / `env` の違いとして実装されるべき

### 4.3 Prompt template の実装

`hermes-paperclip-adapter` は prompt template を持ち、`taskId` や `commentId` などを埋め込む。

確認できたこと:

- `{{agentId}}`
- `{{agentName}}`
- `{{companyId}}`
- `{{runId}}`
- `{{taskId}}`
- `{{taskTitle}}`
- `{{taskBody}}`
- `{{commentId}}`
- `{{wakeReason}}`
- `{{projectName}}`
- `{{paperclipApiUrl}}`

のような変数が使える

意味:

- 設計書で定義している `strategy_brief_issue` などは、最終的には issue title/body と prompt template 展開で Hermes に渡る
- したがって「role 間 IF」を考えるときは、issue schema と prompt template の両方を一緒に設計する必要がある

### 4.4 Session persistence

`sessionCodec` により、Hermes 側の session は `sessionId` として持続化される。

確認できたこと:

- `sessionCodec` は `sessionId` / `session_id` を serialize / deserialize する
- 実行時、前回 session があれば `--resume <sessionId>` を付ける

意味:

- Hermes profile を「毎回 stateless に起動する」前提は誤り
- role ごとの memory drift や session reset 方針を設計に入れる必要がある

### 4.5 Skills IF

adapter は skill snapshot API を持つ。

確認できたこと:

- `listSkills`
- `syncSkills`
- `resolveDesiredSkillNames`

があり、Paperclip-managed skills と `~/.hermes/skills/` をマージして見せる

重要:

- Hermes-native skills は `~/.hermes/skills/` を走査する
- `syncSkills` は Hermes 側では実質 no-op で、snapshot を返す
- つまり skill の「有効化/管理」は Paperclip 側 desired skill と Hermes 側常設 skill が混ざる

意味:

- 設計書にある `Hermes profile ごとの使用 SKILL` は概念としてはよい
- ただし実装面では「Paperclip が skill を完全支配する」とは言えない
- `~/.hermes/skills/` にある native skill の存在も IF に含める必要がある

---

## 5. LLM-WIKI 側で確認できた IF

調査対象:

- `llm-wiki.net`
- `nvk/llm-wiki`

### 5.1 提供形態

LLM-WIKI は主に以下で提供される。

- Claude Code plugin
- portable `AGENTS.md`

意味:

- `LLM-WIKI service` という常駐 API 前提ではない
- agent のコンテキストやプロジェクトディレクトリに入る command protocol として扱うべき

### 5.2 主コマンド

確認できた主コマンド:

- `/wiki`
- `/wiki init`
- `/wiki:ingest`
- `/wiki:compile`
- `/wiki:query`
- `/wiki:plan`
- `/wiki:research`
- `/wiki:lint`
- `/wiki:output`
- `/wiki:assess`

意味:

- `LLM-WIKI に蓄積する` は、多くの場合 `ingest -> compile -> query/plan/output` の流れになる
- 設計書でいう `WIKI 更新` は、抽象語のままではなくこの command sequence に落とす必要がある

### 5.3 ファイル構造

確認できた基本構造:

- `~/wiki/`
- `topics/<name>/`
- `inbox/`
- `raw/`
- `wiki/`
- `output/`
- `_index.md`
- `log.md`

重要:

- `raw/` は immutable
- `wiki/` は compiled knowledge
- `output/` は生成物

意味:

- 戦略ノウハウを保存するなら、「生ログ」「中間メモ」「コンパイル済み知識」を分けるべき
- `Paperclip issue documents` と `LLM-WIKI raw/wiki/output` の責務を混ぜないほうがよい

### 5.4 この設計への含意

`Strategy Lead が WIKI 更新` という設計は、そのままだと粗い。  
より正確には、`Strategy Lead` は strategy phase の早い段階で wiki を **更新しつつ調査と計画を進める**。

より実装寄りには以下のどちらかになる。

1. project-local wiki を持ち、strategy 案の調査時点から `/wiki:research` `/wiki:query` `/wiki:plan` を使う
2. その後、issue から得た validation / incident の知見を `ingest` / `compile` で戻す
3. topic wiki を別で持つ場合は、strategy / incident / validation をテーマ別に compile する

つまり、`WIKI` は:

- REST API 経由で structured metadata を書くものではなく
- command と markdown file layout に従って知識を編纂するもの
- かつ、strategy phase の research / planning interface でもある

として扱うべきである

---

## 6. この調査から設計に入れるべき制約

### 6.1 Paperclip の source of truth

短期の work coordination は `Paperclip issue / comment / document` に置く。

理由:

- heartbeat protocol がそれを前提にしている
- checkout / status / comments / wakeup がそこに統合されている

### 6.2 Hermes profile の差分は adapterConfig で持つ

`strategist` / `research_engineer` / `shadow_operator` / `postmortem_analyst` の違いは、少なくとも以下で表すべきである。

- `cwd`
- `toolsets`
- `timeoutSec`
- `maxTurnsPerRun`
- `promptTemplate`
- `env`
- 必要なら `extraArgs`

role 名だけで責務分離したつもりになるのは危ない。

### 6.3 LLM-WIKI は長期知識の蓄積先に寄せる

推奨の住み分け:

- `Paperclip issue documents`
  - 今回の brief
  - 今回の report
  - 今回の postmortem
- `LLM-WIKI`
  - 再利用したい strategy rationale
  - incident lessons learned
  - venue / market / execution rule の蓄積知識
  - 実験から得た抽象化済みパターン

### 6.4 まだ仮定でしかない部分

以下は未確定、または設計で勝手に決めてはいけない。

- Paperclip 上での `Hermes profile` と agent role/title の自動解決規則
- `Strategy Lead -> WIKI 更新` の具体 command sequence
- `research_result_review` などの issue body template 本文
- `pi-research` / `pi-ctl` / `pi-live` への remote execution routing の実装方法
- live ノードに対する Hermes / Nautilus 実行 skill / transport の仕様

---

## 7. 次に必要な設計文書

この調査の次に必要なのは、理想論の role 図ではなく、以下の実装寄り文書である。

### 7.1 Paperclip ↔ Hermes Runtime Contract

最低限含めるべきもの:

- agent role/title と `adapterConfig` の対応
- 各 profile の `cwd` / `toolsets` / `timeoutSec` / `promptTemplate`
- 使用する issue document keys
- wake reason ごとの振る舞い
- session reset 条件

### 7.2 LLM-WIKI Operational Contract

最低限含めるべきもの:

- どの role がどの command を使うか
- `ingest / compile / query / plan / output` の使い分け
- `Paperclip issue documents` から何を wiki に昇格させるか
- topic wiki と local wiki のどちらを使うか

補足:

- 本リポジトリでは [paperclip-hermes-runtime-contract.md](D:/GitHub/quant-agent-platform/docs/paperclip-hermes-runtime-contract.md) と [llm-wiki-operational-contract.md](D:/GitHub/quant-agent-platform/docs/llm-wiki-operational-contract.md) を初版として追加済み

---

## 8. 確認元

Paperclip:

- https://docs.paperclip.ing/adapters/overview
- https://docs.paperclip.ing/adapters/creating-an-adapter
- https://docs.paperclip.ing/start/architecture
- https://docs.paperclip.ing/guides/agent-developer/heartbeat-protocol
- https://docs.paperclip.ing/api/issues
- https://docs.paperclip.ing/api/agents
- https://docs.paperclip.ing/api/authentication
- https://docs.paperclip.ing/api/overview
- https://docs.paperclip.ing/deploy/environment-variables
- https://github.com/paperclipai/paperclip/blob/master/server/src/adapters/builtin-adapter-types.ts
- https://github.com/paperclipai/paperclip/blob/master/ui/src/adapters/hermes-local/index.ts
- https://github.com/paperclipai/paperclip/blob/master/docs/agents-runtime.md

Hermes adapter:

- https://github.com/NousResearch/hermes-paperclip-adapter/blob/main/README.md
- https://github.com/NousResearch/hermes-paperclip-adapter/blob/main/src/index.ts
- https://github.com/NousResearch/hermes-paperclip-adapter/blob/main/src/server/index.ts
- https://github.com/NousResearch/hermes-paperclip-adapter/blob/main/src/server/execute.ts
- https://github.com/NousResearch/hermes-paperclip-adapter/blob/main/src/server/skills.ts
- https://github.com/NousResearch/hermes-paperclip-adapter/blob/main/src/ui/build-config.ts

LLM-WIKI:

- https://llm-wiki.net/
