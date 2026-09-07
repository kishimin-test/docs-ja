# admin-web のアプリ基盤と認証ガードを構築する

| 項目 | 値 |
| ---- | -- |
| Issue | [#12](https://github.com/kishimin-ai-create/profile-hub/issues/12) |
| 英語タイトル | Build the admin-web application foundation and auth guard |
| マイルストーン | v1 |
| ラベル | enhancement,area:admin-web |

---

## 解決したい課題

管理画面アプリの土台がなく、ログイン画面も記事管理画面も実装を開始できない。土台なしに画面を実装すると、認証ガード・API 呼び出し・エラー表示が画面ごとにばらつき、未認証で管理画面が表示される事故も起きやすい。

## 提案する解決策

`apps/admin-web` を Vite + React で構築する。

- Vite + React + TypeScript の SPA として構成する
- TanStack Router によるルーティングと、認証ガードによる保護
  - 未認証状態で保護ルートへアクセスするとログイン画面へリダイレクトする
  - ガードはルート定義側で一括して適用し、画面ごとの付け忘れが起きない構造にする
- TanStack Query による API 状態管理 (取得・更新・キャッシュ・再取得)
- orval による API クライアント生成 (API の OpenAPI スキーマを入力とする)
- 401 応答を検知した際にログイン画面へ遷移する共通処理
- 共通レイアウト — ヘッダー、サイドナビゲーション、ログアウト
- 共通のローディング表示、エラー表示、トースト通知
- フォーム基盤 (React Hook Form + スキーマ検証) と、API 側の検証エラーをフィールドへ反映する仕組み
- ディレクトリ構成を `src/{app,features,components,hooks,lib,api,models,providers,schemas,types,utils,tests}` とし、`boundaries` の依存方向制約を満たす
- Storybook と Vitest / Testing Library のセットアップ
- 管理画面は検索エンジンにインデックスさせない

## 代替案

- **public-web と同一アプリ内のルートとして管理画面を実装する** — 実装量は減るが、公開側に管理用の依存とルートが含まれる。分離を採用して不採用。
- **API クライアントを手書きする** — API 変更時の追従漏れが起きる。orval による生成を採用して不採用。

## 受入条件

- [ ] `bun run dev` で管理画面が起動する
- [ ] 未認証状態で保護ルートへ直接 URL アクセスするとログイン画面へリダイレクトされる
- [ ] 認証済み状態で保護ルートを表示でき、リロードしても認証状態が維持される
- [ ] API が 401 を返したときにログイン画面へ遷移する
- [ ] ログアウトすると保護ルートへアクセスできなくなる
- [ ] orval で API クライアントが生成され、型が解決される
- [ ] `bun run lint` `bun run typecheck` `bun run test` が成功する
- [ ] `boundaries` の依存方向制約に違反していない
- [ ] 管理画面がクローラーにインデックスされない設定になっている

## 対象外

- 個別画面 (ログイン、記事管理、プロフィール管理) の実装 — 別Issue
- 認証APIの実装 — 別Issue

## 補足

- 依存: モノレポ再編Issue、Lint基盤Issue、API 基盤Issue、認証Issue
- 技術構成は mojica のフロントエンドに揃える


---

## デザインレビューによる更新 (2026-09-04)

サイドバーのデザイン (`#3:18`) は Dashboard / Articles / Profile / Media / Settings を並べている。Media は v2 (#26) のため、v1 では非表示にするか、無効状態であることを明示する。
