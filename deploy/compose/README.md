# Compose Layout

このディレクトリには、**`pi-ctl` の DB 専用 compose** を置く。

現在の前提はこれで固定する。

- `Paperclip` は `pi-ctl` にネイティブ配置する
- `Hermes runtime` も `pi-ctl` にネイティブ配置する
- `PostgreSQL` は Raspberry Pi 上での安定性のため **container で動かす**
- `cloudflared` も `Paperclip native` に寄せて、初期はネイティブ service を推奨する
- `NautilusTrader` の research / live 実行は compose 管理しない

要するに、**compose は DB のためだけに使う**。

## ファイル

- `state-control.compose.yml`
  - `pi-ctl` 向け
  - `PostgreSQL` のみ
- `state-control.env.example`
  - `PostgreSQL` 用 env サンプル

## なぜこの形か

実 IF を優先すると、`hermes_local` は Paperclip 側で **ローカルの Hermes CLI を起動する** 前提である。  
したがって `Paperclip` と `Hermes runtime` は同じ `pi-ctl` 上でネイティブに動かすのが最も自然である。

一方で、Raspberry Pi 環境では組み込み DB が安定しなかったため、DB だけは container に分離する。

この形のメリット:

- `Paperclip native` と `hermes_local` の実 IF が素直に噛み合う
- `~/.hermes/skills/`、SSH 設定、workspace を自然に扱える
- DB だけを container 化して運用を単純化できる
- research / live 実行面を compose の固定 service にしなくてよい

## 使い方

`pi-ctl` 上で `.env` を作って起動する。

```bash
cp state-control.env.example .env.state-control
docker compose --env-file .env.state-control -f state-control.compose.yml up -d
```

その後、`Paperclip native` と `Hermes native` は host 側で起動する。

## `pi-ctl` の想定プロセス配置

- compose:
  - `postgres`
- native:
  - `paperclip`
  - `hermes`
  - `cloudflared`
  - 必要なら `hindsight`

補足:

- `Hindsight` は初期段階では native 配置を推奨する
- もし後で `Hindsight` を container 化してもよいが、base compose には含めない
- `Paperclip` / `Hindsight` が同一 Postgres instance を使う場合でも、接続 URL はアプリ側で明示する

## いま未確定な点

### 1. `Paperclip native` の起動方式

- `systemd` か
- 専用 launcher か
- update 手順をどうするか

### 2. `Hindsight` の配置

- native に置くか
- 別 container にするか
- Paperclip と同じ Postgres instance をどう使い分けるか

### 3. `cloudflared` の配置

- 初期は native service を推奨
- ただし IaC や更新方式をどうそろえるかは別途決める

## 設計との対応

- `pi-ctl`
  - `Paperclip native`
  - `Hermes native`
  - `PostgreSQL container`
- `pi-research`
  - compose ではなく Hermes skill + SSH + isolated container
- `pi-live`
  - compose ではなく Hermes skill + SSH + isolated container

関連文書:

- [architecture-overview.md](D:/GitHub/quant-agent-platform/docs/architecture-overview.md)
- [paperclip-hermes-nautilus-design.md](D:/GitHub/quant-agent-platform/docs/paperclip-hermes-nautilus-design.md)
- [paperclip-hermes-runtime-contract.md](D:/GitHub/quant-agent-platform/docs/paperclip-hermes-runtime-contract.md)
- [hermes-nautilus-skill-contract.md](D:/GitHub/quant-agent-platform/docs/hermes-nautilus-skill-contract.md)
- [../runtime/README.md](D:/GitHub/quant-agent-platform/deploy/runtime/README.md)
