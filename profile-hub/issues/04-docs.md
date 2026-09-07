# 要件定義・仕様ドキュメントを profile-hub 用に刷新する

| 項目 | 値 |
| ---- | -- |
| Issue | [#4](https://github.com/kishimin-ai-create/profile-hub/issues/4) |
| 英語タイトル | Rewrite the requirements and specification documents for profile-hub |
| マイルストーン | v1 |
| ラベル | documentation |

---

## 解決したい課題

`docs/v1/` 配下の要件定義・仕様が前身プロジェクト Daybook (日記アプリ) のもので、profile-hub の実態と一致しない。この状態で実装を進めると、仕様書を読んだ人が誤った前提で設計・レビューを行う。

具体的な不一致:

| ドキュメントの記述 | profile-hub の実態 |
| ------------------ | ------------------ |
| 日記の閲覧・日付検索・CRUD | 自己紹介・趣味記事・技術記事の閲覧と管理 |
| `diaries` テーブル | `Article` (type: hobby/engineering) ほか |
| PostgreSQL | MySQL |
| Next.js フロントエンド | Vite + React (public-web / admin-web) |
| サービス名 Daybook | profile-hub |
| 公開サイトと管理画面が同一アプリ | 3 アプリに分離 |

また i18n、SEO、広告表示、公開/下書き管理の要件が一切記載されていない。

## 提案する解決策

`docs/v1/` を profile-hub の要件で書き直す。既存のファイル構成 (requirements / specification) は維持する。

**requirements**

- `functional-requirements.md` — 公開閲覧、記事管理、プロフィール管理、認証・認可、公開/下書き、i18n、広告
- `non-functional-requirements.md` — 性能、可用性、セキュリティ、保守性、SEO、データ整合性 (MySQL)、運用 (さくらのクラウド AppRun)
- `data-requirements.md` — User / Profile / Career / Skill / Article / Category / Tag / Media / Translation / Setting
- `screen-requirements.md` — public と admin の画面要件
- `api-requirements.md` — 公開API と 管理API の分離
- `usecase.md` — 訪問者 / 管理者のユースケース

**specification**

- `api-specification.md`、`db-specification.md`、`ui-specification.md` を上記に合わせて更新する

既存ファイルのタイポ (`functinal-requirements.md`, `non-functionnal-requirements.md`, `screen_requirements.md`) も併せて修正する。

## 代替案

- **Daybook のドキュメントを残したまま profile-hub 用を追記する** — どちらが有効な仕様か判別できなくなる。置き換えを採用して不採用。
- **ドキュメントを廃止して Issue のみを仕様とする** — 横断的な仕様 (データモデル、API一覧) を追跡しづらい。ドキュメント維持を採用して不採用。

## 受入条件

- [ ] `docs/v1/` に Daybook・日記・PostgreSQL・Next.js を前提とした記述が残っていない
- [ ] データ要件が MySQL と `Article.type` / `Article.status` の設計を反映している
- [ ] i18n (ja/en)、SEO、広告表示、公開/下書き管理の要件が記載されている
- [ ] 画面要件が public / admin の 2 系統に分かれている
- [ ] ファイル名のタイポが修正されている

## 対象外

- ADR の作成 (設計判断が確定した時点で別途記録する)
- README の更新

## 補足

- 全体方針は Epic Issue を参照する。


---

## デザインレビューによる更新 (2026-09-04)

デザインレビュー (`review/figma-ui-design-20260904.md`) が以下の未決事項を確定させたため、ドキュメント刷新時に取り込む。

- 記事本文の形式は **Markdown**
- Contact は **フォームを持たない** (SNS・メールへの導線のみ)
- 記事の多言語編集は **言語タブ方式**
- 記事一覧のフィルターは **type + status**
- Admin ダッシュボードの指標は **全記事 / 公開中 / 下書き**
