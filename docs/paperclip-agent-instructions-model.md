# Paperclip Agent Instructions Model

本書は、`Paperclip` における **Agent の人格・振る舞い・役割指示がどこで定義されるか** を整理するためのメモである。  
特に、`Hermes` の `SOUL.md` と比較したときに、`Paperclip` 側に何が相当するのかを明確にすることを目的とする。

---

## 1. 結論

`Paperclip` には、`Hermes` のような **単一の `SOUL.md` 相当ファイル** は見当たらない。  
その代わり、Agent の振る舞いは主に次の 3 層で決まる。

- `agent record`
  - `name`
  - `role`
  - `title`
  - `reportsTo`
  - `capabilities`
  - `adapterType`
  - `adapterConfig`
  - `runtimeConfig`
- `adapterConfig.promptTemplate`
  - 旧来の prompt ベース設定
- `instructions bundle`
  - `AGENTS.md` を入口にする managed instructions

つまり、`Paperclip` における「人格・役割指示」の中心は、単一の `SOUL.md` ではなく、**Agent record + promptTemplate + managed instructions bundle** の組み合わせである。

---

## 2. 何がどこに入るのか

### 2.1 `agent record`

まず Agent 自体の基本属性がある。  
source 上では `Agent` 型と validator に以下の項目がある。

- `name`
- `role`
- `title`
- `reportsTo`
- `capabilities`
- `adapterType`
- `adapterConfig`
- `runtimeConfig`
- `budgetMonthlyCents`
- `metadata`

ここで重要なのは、`capabilities` は存在するが、これは **短い責務説明や期待する能力の欄** であって、長い人格定義そのものとは限らない、という点である。

---

### 2.2 `adapterConfig.promptTemplate`

`Paperclip` の hire / create-agent 周りの reference では、`adapterConfig` の中に `promptTemplate` を入れる例がある。

例:

```json
{
  "adapterType": "codex_local",
  "adapterConfig": {
    "cwd": "/absolute/path",
    "model": "claude-sonnet-4-5-20250929",
    "promptTemplate": "You are CTO..."
  }
}
```

このことから、UI 上で見える「人格っぽい入力欄」は、少なくとも一部の adapter では **`adapterConfig.promptTemplate` に入る可能性が高い**。

ただし、source では `promptTemplate` は **legacy prompt** として扱われている。  
したがって、長期的な正規形はここだけではない。

---

### 2.3 `instructions bundle`

source でいちばん重要なのはここである。  
`Paperclip` には Agent ごとの instructions bundle があり、以下のような情報を持つ。

- `mode`
  - `managed` または `external`
- `rootPath`
- `managedRootPath`
- `entryFile`
- `resolvedEntryPath`
- `files`

さらに instructions service では、以下の key が明示されている。

- `instructionsBundleMode`
- `instructionsRootPath`
- `instructionsEntryFile`
- `instructionsFilePath`
- `agentsMdPath`

そして default entry file は **`AGENTS.md`** である。

つまり、`Paperclip` の新しい instructions model では、

- prompt を 1 本の文字列で持つ
- その代わり Agent ごとに instructions root を持ち
- 入口として `AGENTS.md` を読む

という方向に寄っている。

これは `Hermes` の `SOUL.md` とはかなり性質が違う。

- `Hermes SOUL.md`
  - instance-global identity
  - durable personality
- `Paperclip instructions bundle`
  - agent-local instructions
  - role / task / execution guidance

---

## 3. UI の「人格入力欄」は何か

UI spec と component 実装を見ると、Agent 作成・編集 UI には少なくとも次の面が存在する。

- `Capabilities`
  - identity section の自由記述欄
- `Prompt Template`
  - local adapter で表示される prompt 欄
- `Instructions`
  - `AgentDetail` 上の別タブ

したがって、ユーザーが Web UI で `CEO` などを追加するときに見た「人格を入力するような欄」は、実装上は次のどれかである。

- `capabilities`
  - 短い責務説明
- `adapterConfig.promptTemplate`
  - adapter に渡す prompt
- managed instructions bundle の editor
  - `AGENTS.md` などを編集する別タブ

また、`AgentConfigForm` の実装では `Capabilities` と `Prompt Template` は別フィールドであり、`AgentDetail` では `configuration` と `instructions` が分離されている。  
したがって、現在の Paperclip では **identity summary / prompt / instructions bundle は別面** と考えるのが自然である。

したがって、Web UI に「人格っぽい欄」があったとしても、それを単純に `SOUL.md` 相当とみなすのは危険である。

---

## 4. Hermes の `SOUL.md` との対応

このプロジェクトでは、次のように整理するのが自然である。

- `Hermes SOUL.md`
  - Hermes 全体の恒久方針
  - tone / posture / safety / autonomy boundary
- `Paperclip agent record`
  - 役職、組織位置、短い責務
- `Paperclip instructions bundle`
  - その role に固有の実務指示
- `adapterConfig`
  - model / cwd / tool / adapter-specific prompt

要するに:

- **人格の核** は `Hermes SOUL.md`
- **役職ごとの運用指示** は `Paperclip instructions bundle`
- **短い説明** は `capabilities`
- **adapter 実行時の細部** は `adapterConfig`

---

## 5. 設計上の含意

この理解に立つと、事前に人間が設計すべきものは次のように分かれる。

### 5.1 人間が先に設計するもの

- `Hermes` の global `SOUL.md`
- `Paperclip` の role taxonomy
  - `CEO / CTO`
  - `Strategy Lead`
  - `Quant Research Engineer`
  - `Shadow Trading Operator`
  - `Incident Analyst`
- role ごとの `Paperclip instructions bundle` の骨組み
- `capabilities` の短い記述方針

### 5.2 runtime に委ねてよいもの

- instructions bundle の細部改善
- promptTemplate の細かな調整
- role ごとの派生手順
- skill から見た補助的な運用メモ

---

## 6. 実務上の推奨

`Paperclip` を `Hermes` と組み合わせて使うなら、次の前提で考えるのがよい。

- `SOUL.md` を `Paperclip` にコピーしない
- `Paperclip` 側では role ごとの `AGENTS.md` ベース instructions を持つ
- UI の短い人格欄だけで role の本質を表そうとしない
- `capabilities` は summary と割り切る
- 長文指示は instructions bundle に置く

これにより、

- Hermes の恒久的 identity
- Paperclip の組織的 role instructions
- adapter 実行の具体設定

を分離できる。

---

## 7. source で確認した主なポイント

- `packages/shared/src/types/agent.ts`
  - `Agent` 型と `AgentInstructionsBundle`
- `packages/shared/src/validators/agent.ts`
  - create/hire schema
  - instructions bundle update schema
- `server/src/services/agents.ts`
  - agent record の永続化対象
- `server/src/services/agent-instructions.ts`
  - `AGENTS.md` default
  - legacy `promptTemplate` fallback
  - managed instructions bundle
- `server/src/routes/agents.ts`
  - instructions bundle route
  - adapter ごとの instructions path key
- `skills/paperclip-create-agent/references/api-reference.md`
  - `adapterConfig.promptTemplate` を含む hire 例
- `skills/paperclip-create-agent/SKILL.md`
  - run prompt は adapter config に入れると明記
- `ui/src/pages/AgentDetail.tsx`
  - `configuration` と `instructions` の分離
- `ui/src/components/AgentConfigForm.tsx`
  - `Capabilities` と `Prompt Template` の別フィールド
- `docs/specs/agent-config-ui.md`
  - create dialog / config UI の field 定義
