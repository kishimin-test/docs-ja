# public-web のアプリ基盤と共通レイアウトを構築する

| 項目 | 値 |
| ---- | -- |
| Issue | [#16](https://github.com/kishimin-ai-create/profile-hub/issues/16) |
| 英語タイトル | Build the public-web application foundation and shared layout |
| マイルストーン | v1 |
| ラベル | enhancement,area:public-web |

---

## 解決したい課題

公開サイトアプリの土台がなく、各ページの実装を開始できない。土台なしにページを実装すると、レイアウト・データ取得・エラー表示がページごとにばらつく。

また、公開サイトは SEO・i18n・広告表示という横断要件を持つため、これらを後から差し込める構造を最初に決めておかないと、全ページの作り直しになる。

## 提案する解決策

`apps/public-web` を Vite + React で構築する。

- Vite + React + TypeScript の SPA として構成する
- TanStack Router によるルーティング (`/`, `/about`, `/hobby`, `/hobby/:slug`, `/engineering`, `/engineering/:slug`, `/contact`)
- TanStack Query による公開APIからのデータ取得
- orval による API クライアント生成
- 共通レイアウト — ヘッダー (ロゴ、サイト名、ナビゲーション、言語切替の設置場所)、フッター
- 404 ページと、データ取得失敗時のエラー表示
- ローディング状態の表示
- ページ単位でメタ情報 (title / description / OGP) を差し込める仕組み。SEO Issue で利用する
- ロケール付きルーティングの方式を決定する。i18n Issue で利用する
- 広告枠の配置場所を確保する。広告 Issue で利用する
- レスポンシブ対応。デスクトップとモバイルで閲覧できる
- ディレクトリ構成を `src/{app,features,components,hooks,lib,api,models,providers,schemas,types,utils,tests}` とし、`boundaries` の依存方向制約を満たす
- Storybook と Vitest / Testing Library のセットアップ
- 認証に関する依存とコードを含めない

## 代替案

- **admin-web とルーティング・レイアウトを共有する** — 実装は減るが、公開側に管理用の依存が入る。分離を採用して不採用。
- **共通 UI を最初から `packages/ui` に置く** — 2 アプリで実際に共有する必要が生じたコンポーネントから移す。先に抽象化しない。

## 受入条件

- [ ] `bun run dev` で公開サイトが起動する
- [ ] 上記のルートがすべて表示でき、直接 URL アクセスとリロードでも動作する
- [ ] ヘッダーとフッターが全ページで一貫して表示される
- [ ] 存在しないパスで 404 ページが表示される
- [ ] API 取得失敗時にエラー表示が出て、画面が白紙にならない
- [ ] デスクトップとモバイルの画面幅で崩れずに表示される
- [ ] ページ単位で title / description を設定できる
- [ ] 公開サイトのバンドルに認証関連のコードが含まれない
- [ ] `bun run lint` `bun run typecheck` `bun run test` が成功する
- [ ] `boundaries` の依存方向制約に違反していない

## 対象外

- 各ページの内容の実装 — 別Issue
- SEO、i18n、広告の実装 — 別Issue

## 補足

- 依存: モノレポ再編Issue、Lint基盤Issue、API 基盤Issue
- 技術構成は mojica のフロントエンドに揃える


---

## デザインレビューによる更新 (2026-09-04)

デザインレビュー (`review/figma-ui-design-20260904.md`) が共通レイアウトに関わる 3 点を検出した。

- **V-3** — public の 5 画面すべてで、フッターに著作権表記しか描画されず `GitHub / X / Contact` のリンク (`#2:49` ほか) が現れない。実装前にフッターの意図した内容を確認する
- **M-4** — ヘッダー・フッターは 64px でインセットされるが、本文はページごとに 64 / 100 / 180 / 240 / 260px と異なり、ロゴと本文の左端が揃わない。コンテナ幅を少数 (例: 標準 / 記事) に整理する
- **V-10** — public の全フレームでフッターの下に 160〜200px の未使用領域がある。フッターが下端に固定されていないため。短いページでもフッターが最下部に来るようにする (sticky footer)

受入条件に以下を追加する。

- [ ] 短いページでもフッターがビューポート下端に位置する
- [ ] ヘッダーと本文の左端が揃っている
