# Hermes 初期セットアップ実務の対応表

## 目的

本書は、「Hermes Agent を使い始めた後にまず試す10件」というタイプの実践的なセットアップ指針を、このリポジトリのアーキテクチャに対応づけるためのメモである。

意図的に、以下の 2 つに分けて整理する。

- `人間が先に固定するもの`
  - hard IF、安全境界、長寿命ポリシーとして、先に決めておくべきもの
- `Hermes / Paperclip に委ねるもの`
  - 事前に設計しすぎず、改善ループの中で育てるべきもの

この考え方は [SOUL.md](/D:/GitHub/quant-agent-platform/SOUL.md:1) に従う。

## 前提

- `pi-ctl`
  - control plane
  - `Paperclip native`
  - `Hermes native`
  - `PostgreSQL container`
- `pi-research`
  - SSH 経由で実行する research execution target
- `pi-live`
  - SSH 経由で実行する live execution target
- `LLM-WIKI`
  - strategy / research knowledge
- `Hindsight`
  - Hermes の自己改善履歴

## 対応表

| 投稿の論点 | 意味 | このリポジトリで人間が先に固定するもの | このリポジトリで Hermes / Paperclip に委ねるもの |
|---|---|---|---|
| 反BOTブラウザ環境 | 自動化のための実ブラウザに近い実行環境 | そもそもブラウザ自動化を許可するか、どのノードで動かせるか、外部資格情報を許可するか | 個別のワークフローで `Camofox`、`Browserbase`、その他の手段のどれを使うか |
| `SOUL.md` | 恒久的な人格と運用原則 | はい。これは先に設計すべき主要ポリシー面である | 具体的な改善案は Hermes から出せるが、正本は人間の方針 |
| `auxiliary` | 安い副モデルへのサブタスク分配 | コスト・安全性の大枠ポリシーのみ | どのタスクをどのモデルに回すか、provider の選択、routing の細部 |
| built-in / plugin system | Hermes の拡張面 | どの種類の拡張を許可するか、ローカル拡張の配置境界 | 具体的にどの拡張を組み合わせるか |
| `web_search` | ネイティブ検索・取得能力 | 検索を許可するか、許可 provider、外部アクセス可否、どの profile で使えるか | 各タスクでどの provider を使うか、fallback 順序、実際の使い方 |
| `Hooks` | イベント駆動の自動化・監査 | どの hook を許可するか、何を必ず記録するか、どの hook はレビュー対象か | 具体的な hook 実装とその改善 |
| `sandbox` | より安全なコマンド実行隔離 | sandbox を必須にするか、許容 backend、どの種類のタスクだけ bypass 可とするか | profile ごとの sandbox 選択、細かな構成、段階的 hardening |
| `multi-agent` | 並列実行や role 分離 | role 境界、approval 境界、node 境界、source of truth | どのタイミングで multi-agent に分割するか、どう協調させるか |
| backup / persistence | 作業物と状態の保全 | はい。保存先、バックアップ方針、秘密情報の扱い、保持期間は先に決める | 実際のバックアップ script や schedule、ファイル配置の改善 |
| `skill` | 再利用可能な手続き能力 | 命名規則、配置場所、権限の強い skill のレビュー境界、許可 skill family | どの skill を作るか、どう磨くか、いつ廃止するか |

## 先に設計すべきもの

### 1. `SOUL.md`

これは人間が与える主要な control surface として扱うべきである。

最低限、以下を定義する。

- hard IF と autonomy boundary
- role ごとの安全境界
- node ごとの責務
- source of truth の優先順位
- approval 境界
- secret handling の期待値
- live 側のリスク原則

このリポジトリでは、`SOUL.md` は単なる補助ではなく、アーキテクチャの一部である。

### 2. Search Policy

provider の使い分けを事前に固定する必要はないが、以下は決めておくべきである。

- Hermes に web search を許可するか
- 許容する provider は何か
- どの種類のタスクで citation や verification を必須にするか
- すべての profile で検索可能にするか、一部だけにするか

これにより、`exa`、`Tavily`、`firecrawl` などを Hermes が選べても、ポリシー制御は保てる。

### 3. Hook Policy

hook の実装詳細ではなく、カテゴリと制約を先に決める。

少なくとも以下を決める。

- 何のイベントを必ず記録するか
- terminal audit を必須にするか
- hook 出力は state を変更してよいか、それとも audit-only か
- live 側の hook に human approval を要求するか

その上で、具体的な hook 実装は Hermes が進化させればよい。

### 4. Sandbox Policy

すべての sandbox profile を人間が先に作り込む必要はないが、以下は固定したほうがよい。

- local 直接実行を許可するか
- Docker ベース sandbox をいつ必須にするか
- `pi-research` と `pi-live` の実行を isolated container や microVM に閉じ込めるか
- どの種類のタスクだけ unsandboxed 実行を許可するか

これは、すべての wrapper command を先に決めるより、今の設計に合っている。

### 5. Backup / Persistence Policy

ここも人間が先に持つべき領域である。

以下は先に決める。

- backup root
- local work product に `git` を必須にするか
- `.env` などの secret exclusion
- どの node がどの raw data を持つか
- research / live artifact の最低保持期間

Hermes が script や schedule を後から改善するのはよいが、安全 baseline は先に明文化したほうがよい。

### 6. Skill Governance

以下は先に決めておく価値が高い。

- project skill の配置場所
- 命名規則
- privileged skill のレビュー期待値
- Hermes bundled skill と project-specific skill の区別

ただし、skill 本体の詳細まで事前固定する必要はない。

## 事前に決めすぎないほうがよいもの

この投稿が有益なのは、Hermes が「正しい方向づけ」を受ければ、自分でかなりのセットアップを進められる前提に立っているからである。

このリポジトリでは、以下を過剰に固定しないほうがよい。

- DuckDB schema の詳細
- Parquet 列定義の詳細
- hook 実装の詳細
- auxiliary routing map の詳細
- タスクごとの search provider の固定
- 初日からの完全な skill inventory

これらは Hermes / Paperclip の運用の中で育て、安定したものだけ durable skill や policy に昇格させればよい。

## このリポジトリへの具体的な含意

### Paperclip

Paperclip は以下の control plane であり続けるべきである。

- role assignment
- issue state
- approvals
- 短期の execution coordination

すべての挙動詳細を人間が手で埋め込む場所にしてはいけない。

### Hermes

Hermes は以下を担うべきである。

- `pi-ctl` 上のローカル実行 orchestration
- `pi-research` と `pi-live` への SSH ベース delegation
- skill の成長
- hook の成長
- ポリシー境界内での運用改善

### `LLM-WIKI`

この投稿は、恒久的な guidance と working knowledge が重要だと補強している。

このリポジトリでは以下を分離して保つべきである。

- `SOUL.md`
  - 恒久的な運用原則
- `LLM-WIKI`
  - strategy / research knowledge
- `Hindsight`
  - execution / self-improvement memory

## 推奨順序

この投稿を今のアーキテクチャに当てると、次の順で進めるのがよい。

1. `SOUL.md` を canonical な human policy surface として強化する
2. search policy と許容 provider を決める
3. hook policy と audit expectation を決める
4. sandbox policy を境界レベルで決める
5. backup / persistence policy を決める
6. その上で具体的な skill、hook、運用 routine は Hermes に育てさせる

## 結論

この投稿は、あなたがすでに述べている方向性とかなり整合している。

このリポジトリに対する最も強い示唆は次の 3 点である。

- `SOUL.md` は確実に先に設計すべきである
- search、hooks、sandbox、backup は policy レベルで先に設計すべきである
- 具体的な workflow、skill 本体、query layout などの多くは Hermes / Paperclip に進化させるべきである
