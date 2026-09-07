# フロントエンドの Lint 基盤を mojica と同一ルールで整備する

| 項目 | 値 |
| ---- | -- |
| Issue | [#3](https://github.com/kishimin-ai-create/profile-hub/issues/3) |
| 英語タイトル | Set up the frontend lint baseline with the same rules as mojica |
| マイルストーン | v1 |
| ラベル | enhancement,area:tooling |

---

## 解決したい課題

profile-hub のフロントエンドは mojica (https://github.com/kishimin/mojica) と同じLintルールで運用したいが、現在のリポジトリにはその設定が存在しない。アプリを 2 つ (public-web / admin-web) 作るため、設定を各アプリへコピーすると必ず乖離する。

## 提案する解決策

mojica の `frontend/eslint.config.mjs` と同等のルールセットを `packages/config` に集約し、`apps/public-web` と `apps/admin-web` から共有して適用する。

### 移植対象

**独自ルール (`eslint-rules/`)**

- `prefer-object-derived-union`
- `prefer-generated-image-msw-handler`
- `prefer-named-exports-in-utils`
- `limit-props-keys`
- `require-blank-line-between-form-fields`

**構文制約 (`no-restricted-syntax`)**

- 関数宣言・関数式を禁止しアロー関数を使う
- `React.` 名前空間経由の参照を禁止し API を直接 import する
- JSX の文字列属性・テキストを波括弧で包む
- `let` を禁止し `const` を使う

**その他の主要ルール**

- `max-params: 5`、`utils/` 配下は `max-lines-per-function: 30`
- `@typescript-eslint/no-explicit-any: error`、`recommendedTypeChecked` 有効
- `import/order` のアルファベット順、`unused-imports/no-unused-imports`
- `@eslint-community/eslint-comments/require-description` (Lint抑制には理由を必須にする)
- `eslint-plugin-boundaries` による bulletproof-react 型の依存方向制約
  - `shared/` は `features/` `app/` に依存しない
  - `features/<feature>` は他 feature を import しない
  - `app/` は誰からも import されない
- `features/*/components/**` から `@/models/*` (生成されたAPIモデル) の import を禁止する
- Vitest / Testing Library / Storybook / JSDoc ルール
- `prettier` を最後に適用し、`eslint-plugin-oxlint` で oxlint と重複するルールを無効化する

**併用ツール**

- `oxlint` (`.oxlintrc.json`) — `lint` は `oxlint && eslint .` の順で実行する
- `markuplint` (`.markuplintrc.json`) — `src/**/*.tsx` のマークアップ検証
- 独自ルールの回帰テスト (`eslint.config.rule-tests.mjs` 相当)

## 代替案

- **各アプリに設定をコピーする** — 初期は速いが 2 アプリ間で必ず乖離する。共有パッケージ化して不採用。
- **mojica の設定を npm パッケージとして公開し依存する** — 再利用性は高いが、公開・バージョニングの運用コストが発生する。リポジトリ内共有で足りるため現時点では不採用。

## 受入条件

- [ ] `apps/public-web` `apps/admin-web` で `bun run lint` が `oxlint` と `eslint` の順に実行され、成功する
- [ ] 上記 5 種の独自ルールが両アプリで有効になっている
- [ ] `let`、関数宣言、`React.` 名前空間参照、波括弧なしJSX文字列属性が Lint エラーになる
- [ ] `shared/` から `features/` を import すると `boundaries/dependencies` エラーになる
- [ ] `features/a` から `features/b` を import すると Lint エラーになる
- [ ] 独自ルールの回帰テストが実行でき、成功する
- [ ] `bun run lint:markup` が実行できる

## 対象外

- API (`apps/api`) の Lint 設定 — 既存 `backend/eslint.config.mts` を引き継ぐ
- Prettier のフォーマット規則の変更

## 補足

- 参照元: https://github.com/kishimin/mojica の `frontend/eslint.config.mjs`, `frontend/eslint-rules/`, `frontend/.oxlintrc.json`, `frontend/.markuplintrc.json`
- boundaries のディレクトリ規約に合わせ、両アプリとも `src/{app,features,components,hooks,lib,api,models,providers,schemas,types,utils,tests}` 構成にする必要がある。
