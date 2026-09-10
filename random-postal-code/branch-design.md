# Zipnami Git ブランチ設計書

## 1. 目的

この文書は [Zipnami MVP 設計書](./design.md) と [tracker Issue #22](https://github.com/kishimin/random-postal-code/issues/22) を、レビュー可能なGitブランチへ分割するための正本である。

## 2. ブランチ運用契約

- `main` は共有・安定ブランチとし、直接コミットまたは直接pushしない。
- すべての変更は最新の `main` から短命な作業ブランチを作り、Pull Request 経由で反映する。
- 1ブランチは1目的、1Issue、1Pull Requestを原則とする。
- ブランチ名は `{type}/{issue-number}-{short-description}` とする。
- 利用者向け機能は `feature/`、不具合修正は `fix/`、振る舞いを変えない改善は `refactor/`、文書のみは `docs/`、テストのみは `test/`、保守は `chore/` を使う。
- `feat/` はブランチ名に使用せず、機能コミットの `feat:` と区別する。
- 後続ブランチは、表に記載した前提Issueが `main` へマージされた後、更新済み `main` から作成する。
- 作業ブランチに `origin/main` をupstreamとして設定しない。
- push、PR作成、merge、rebase、ブランチ削除はそれぞれ明示的な依頼または専用ワークフローで行う。

## 3. コミット設計

コード変更は1振る舞いずつ次のコミットを積む。

1. `test:` — 観測可能な要求を失敗するテストで固定する。
2. `feat:` または `fix:` — そのテストを通す最小実装を行う。
3. `refactor:` — 必要な場合だけ、Greenを保って構造を改善する。

文書のみは `docs:`、CIや依存設定など利用者の振る舞いを直接変更しない保守作業は `chore:` を使う。各コミットは英語の命令形Summary、空行、具体的なWhyを示すBodyを持つ。生成物、秘密情報、無関係な変更を混在させない。

## 4. 実装順序

Issue #22 はトラッカーであり、専用の実装ブランチを持たない。次の順序は依存関係を解消する安全な直列マージ順である。並行作業する場合も、各行の「前提Issue」を満たす `main` から分岐する。

| 順序 | Issue | ブランチ | 前提Issue | 所有スコープ |
| ---: | ---: | --- | --- | --- |
| 1 | [#1](https://github.com/kishimin/random-postal-code/issues/1) | `chore/1-bootstrap-monorepo` | なし | Bun workspace、共通設定、既存アセット移設、ローカル品質コマンド |
| 2 | [#2](https://github.com/kishimin/random-postal-code/issues/2) | `feature/2-shared-contracts` | #1 | `Address`、`PostalCode`、API成功・失敗契約 |
| 3 | [#3](https://github.com/kishimin/random-postal-code/issues/3) | `feature/3-normalize-postal-data` | #1, #2 | 日本郵便データ取得、正規化、決定的生成物 |
| 4 | [#4](https://github.com/kishimin/random-postal-code/issues/4) | `feature/4-random-postal-api` | #2, #3 | Hono Worker、`GET /api/random`、一様抽選 |
| 5 | [#5](https://github.com/kishimin/random-postal-code/issues/5) | `feature/5-web-frontend-foundation` | #1, #2 | React/Vite SPA、Pages設定、Frontend構成 |
| 6 | [#6](https://github.com/kishimin/random-postal-code/issues/6) | `feature/6-web-random-experience` | #4, #5 | Webの生成、表示、コピー、再生成、主要状態 |
| 7 | [#7](https://github.com/kishimin/random-postal-code/issues/7) | `feature/7-web-history` | #6 | ブラウザ履歴、20件制限、重複保持 |
| 8 | [#8](https://github.com/kishimin/random-postal-code/issues/8) | `feature/8-web-google-maps` | #6 | Maps Embed、住所選択、外部地図リンク |
| 9 | [#9](https://github.com/kishimin/random-postal-code/issues/9) | `feature/9-web-adsense-consent` | #6 | AdSense、Consent、主要機能との分離 |
| 10 | [#10](https://github.com/kishimin/random-postal-code/issues/10) | `feature/10-privacy-attribution` | #5 | `/privacy`、情報画面、出典とデータ処理開示 |
| 11 | [#11](https://github.com/kishimin/random-postal-code/issues/11) | `feature/11-api-cors-secrets` | #4, #5 | Origin完全一致CORS、環境検証、秘密情報境界 |
| 12 | [#12](https://github.com/kishimin/random-postal-code/issues/12) | `feature/12-expo-android-foundation` | #1, #2 | Expo Router、Android基盤、環境別設定 |
| 13 | [#13](https://github.com/kishimin/random-postal-code/issues/13) | `feature/13-android-random-experience` | #4, #12 | AndroidのAPI取得、表示、再生成、主要状態 |
| 14 | [#14](https://github.com/kishimin/random-postal-code/issues/14) | `feature/14-android-history-maps` | #13 | 端末履歴、20件制限、外部地図連携 |
| 15 | [#15](https://github.com/kishimin/random-postal-code/issues/15) | `feature/15-android-admob-consent` | #10, #13 | AdMobバナー、UMP、テスト広告、失敗分離 |
| 16 | [#16](https://github.com/kishimin/random-postal-code/issues/16) | `feature/16-accessibility-responsive` | #7, #8, #9, #14, #15 | Web/Androidの横断アクセシビリティとレスポンシブ確認 |
| 17 | [#17](https://github.com/kishimin/random-postal-code/issues/17) | `feature/17-failure-isolation` | #8, #9, #11, #14, #15 | API、Maps、広告、Consent、外部アプリの失敗境界 |
| 18 | [#18](https://github.com/kishimin/random-postal-code/issues/18) | `chore/18-ci-quality-gates` | #16, #17 | 全workspaceの整形、型、Lint、テスト、カバレッジ、ビルドCI |
| 19 | [#19](https://github.com/kishimin/random-postal-code/issues/19) | `chore/19-deploy-pages-workers` | #10, #11, #18 | Pages/Workers独立デプロイ、設定、rollback、smoke check |
| 20 | [#20](https://github.com/kishimin/random-postal-code/issues/20) | `chore/20-android-play-release` | #15, #16, #17, #18 | AAB、ストア素材、Data Safety、テストトラック、権限確認 |
| 21 | [#21](https://github.com/kishimin/random-postal-code/issues/21) | `test/21-mvp-acceptance` | #19, #20 | Web/Androidの最終受入試験と再現可能な証跡 |

## 5. 各ブランチの開始手順

ブランチ開始時に次を満たす。

1. 対象Issueと [design.md](./design.md) を全文確認する。
2. 関連ADRとSkillを確認する。
3. `main` が前提Issueのマージを含むことを確認する。
4. working treeに持ち越し対象でない変更がないことを確認する。
5. 最新の安定した `main` から表の正確なブランチ名を作成する。
6. 対象Issueの受入条件からテストリストを作る。

ブランチ作成コマンドや検証コマンドは、その時点でリポジトリに定義されたものだけを使用する。計画段階でパッケージスクリプト名を推測しない。

## 6. ブランチ完了条件

各ブランチは次をすべて満たした場合だけ完了とする。

- 対象Issueの受入条件を満たし、スコープ外の機能を追加していない。
- 公開契約、実装、テスト、文書が一致している。
- TDDのテストリストが空で、各振る舞いのRed/Green/Refactor履歴を説明できる。
- リポジトリで利用可能な整形が成功する。
- 利用可能な型チェックまたはコンパイルが成功する。
- 利用可能なLint・静的解析が成功する。
- 対象テストと回帰テストが成功する。
- カバレッジ全体サマリーの全指標が80%以上である。
- 利用可能なproduction buildが成功する。
- Skillによるコードレビューで未解決の重大な指摘がない。
- 変更ファイル、diff、ステージ対象、秘密情報の不在を確認する。
- 英語のWhyを含む規約準拠コミットだけで構成されている。
- Pull Requestが対象Issueを参照し、前提と検証結果を記載している。

コマンドがまだ定義されていない検証項目は推測して実行せず、Pull Requestへスキップ理由を記録する。ただし、利用可能な必須検証に失敗した状態で完了としてはならない。

## 7. 変更管理

設計変更が既存Issueの受入条件または複数ブランチへ影響する場合は、実装ブランチへ埋め込まず、設計文書とIssueを先に同期する。新しい独立スコープは新Issue・新ブランチとし、既存ブランチの目的を膨張させない。

マージ済みブランチへの追加修正は、新しい `fix/` または適切な種別のブランチで行う。共有済み履歴をamend、rebase、force pushで書き換えない。
