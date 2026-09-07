# モノレポ構成 (apps / packages / infra) へ再編する

| 項目 | 値 |
| ---- | -- |
| Issue | [#2](https://github.com/kishimin-ai-create/profile-hub/issues/2) |
| 英語タイトル | Restructure the repository into a monorepo (apps / packages / infra) |
| マイルストーン | v1 |
| ラベル | enhancement,area:tooling |

---

## 解決したい課題

現在のリポジトリはルート直下に `backend/` `frontend/` が並ぶ 2 アプリ構成で、公開サイトと管理画面を分離できない。また共有したい型・UI・Lint設定を置く場所がなく、アプリを増やすと設定ファイルが重複する。

- `frontend/` は Next.js 前提の設定 (`next.config.ts`) が残っており、採用する Vite + React 構成と一致しない。
- `backend/` `frontend/` ともに実装ソースが削除済みで、いま再編すれば移行コストが最小で済む。

## 提案する解決策

Bun ワークスペースによるモノレポへ再編する。

```
repo
├── apps
│   ├── public-web
│   ├── admin-web
│   └── api
├── packages
│   ├── ui
│   ├── types
│   ├── config
│   └── utils
└── infra
```

- ルート `package.json` に `workspaces` を定義し、`bun install` を 1 回で完結させる。
- 既存 `backend/` の Drizzle・Hono 関連設定を `apps/api` へ移す。
- 既存 `frontend/` の Next.js 固有設定は破棄し、`apps/public-web` `apps/admin-web` を Vite + React で新規に作る。
- `packages/config` に共有の ESLint / TypeScript / Prettier 設定を置き、各アプリから参照する。
- 各アプリで `dev` `build` `typecheck` `lint` `test` のスクリプト名を統一する。

## 代替案

- **既存の `backend/` `frontend/` を維持して admin だけ追加する** — 共有設定の重複が残り、`frontend/` の Next.js 前提設定と採用構成の不一致も残る。実装ソースが存在しない今のうちに再編するほうが安価なため不採用。
- **Turborepo / Nx を導入する** — タスクキャッシュの恩恵はあるが、アプリ 3 個の規模では設定コストが上回る。将来必要になった時点で追加できるため、まずは Bun ワークスペースのみとする。

## 受入条件

- [ ] リポジトリルートで `bun install` が成功し、全ワークスペースの依存が解決される
- [ ] `apps/public-web` `apps/admin-web` `apps/api` `packages/*` が上記の構成で存在する
- [ ] 各アプリで `bun run typecheck` `bun run lint` `bun run test` `bun run build` が実行できる
- [ ] `packages/*` を `apps/*` から import でき、型が解決される
- [ ] ルート直下に旧 `backend/` `frontend/` が残っていない

## 対象外

- 各アプリの画面・API の実装
- CI ワークフローの更新 (別Issue)
- デプロイ構成 (別Issue)

## 補足

- 移行元: 既存 `backend/` (Bun + Hono + Drizzle 設定)、`frontend/` (Next.js 設定は破棄)
- `frontend/storybook-static/` はビルド成果物のため移行対象外。`.gitignore` へ追加する。
