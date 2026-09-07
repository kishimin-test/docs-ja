# MySQL のデータモデルを確定し Drizzle スキーマを実装する

| 項目 | 値 |
| ---- | -- |
| Issue | [#6](https://github.com/kishimin-ai-create/profile-hub/issues/6) |
| 英語タイトル | Settle the MySQL data model and implement the Drizzle schema |
| マイルストーン | v1 |
| ラベル | enhancement,area:api |

---

## 解決したい課題

profile-hub のデータモデルが未確定で、API・管理画面・公開サイトのどの実装も開始できない。既存の `docs/v1/requirements/data-requirements.md` は Daybook の `users` / `diaries` 2 テーブル構成かつ PostgreSQL 前提であり、そのまま使えない。

加えて、初期案のエンティティ一覧には設計上の判断が必要な点が残っている。

- `Hobby` テーブルと `Article.type = hobby` が重複する。`Article` へ統合すべきか
- 主キーに UUID を使う場合、MySQL では `CHAR(36)` と `BINARY(16)` のどちらにするか (インデックスサイズと可読性のトレードオフ)
- 多言語データを `Translation` テーブルへ縦持ちするか、各テーブルに `_ja` / `_en` 列を持たせるか

## 提案する解決策

MySQL 上のデータモデルを確定し、Drizzle ORM のスキーマとマイグレーションとして実装する。

### エンティティ

| エンティティ | 役割 |
| ------------ | ---- |
| `User` | 管理者アカウント |
| `Profile` | 自己紹介 (名前、肩書き、紹介文、アバター) |
| `Career` | 経歴 |
| `Skill` | スキルと習熟度 |
| `Article` | 趣味記事・技術記事 |
| `Category` | 記事カテゴリ |
| `Tag` | 記事タグ |
| `ArticleTag` | 記事とタグの多対多 |
| `SocialLink` | SNSリンク |
| `Announcement` | お知らせ |
| `Media` | 画像などのメディア (v2 で利用) |
| `Translation` | 多言語テキスト |
| `Setting` | サイト設定 |

### Article

```
type    : hobby | engineering
status  : draft | published
```

`type` の追加のみで将来カテゴリ (旅行・登壇・制作実績など) を拡張できることを設計上の必須条件とする。

### 決めること

- `Hobby` を独立テーブルにせず `Article.type` へ統合する方針の可否
- 主キーの型 (UUID の格納方式、または AUTO_INCREMENT)
- タイムスタンプの型と保存タイムゾーン (UTC 保存を前提とする)
- 記事の削除方式 (物理削除 / 論理削除)
- 公開日時 (`published_at`) を `status` と別に持つか
- 一意制約とインデックス (記事スラッグ、`status` + `type` + 公開日時、`User.email`)

## 代替案

- **各テーブルに `_ja` / `_en` 列を持たせる** — 実装は単純だが言語追加のたびにスキーマ変更が必要になる。`Translation` の縦持ちと比較して決定する。
- **趣味記事と技術記事を別テーブルにする** — カテゴリ追加のたびにテーブル・API・画面が増えるため不採用。

## 受入条件

- [ ] 上記「決めること」がすべて決定され、根拠とともに `docs/v1/specification/db-specification.md` に記載されている
- [ ] Drizzle スキーマが実装され、MySQL に対してマイグレーションが適用できる
- [ ] マイグレーションを空の DB に適用して全テーブルと制約が作成される
- [ ] `Article.type` に新しい値を追加してもテーブル追加が不要であることが、スキーマ上確認できる
- [ ] 開発用のシードデータを投入できる

## 対象外

- API エンドポイントの実装 — 別Issue
- 本番 DB のプロビジョニング — 別Issue

## 補足

- ORM: Drizzle ORM。既存の `backend/drizzle.config.ts` を移行元とする
- 既存 `docs/v1/requirements/data-requirements.md` は Daybook のもので置き換え対象


---

## デザインレビューによる更新 (2026-09-04)

デザイン (`review/figma-ui-design-20260904.md` の指摘 H-6) がこのデータモデルと一致していない。

| 項目 | このモデル | デザイン |
| ---- | ---------- | -------- |
| `Skill` | 名前 + カテゴリ + 習熟度 | `#2:85` はタグの羅列のみ。カテゴリも習熟度も無し |
| `Career` | 期間 + 組織 + 役割 + 説明 | `#2:76` は `2026 — Frontend Engineering / Product development` の 1 文字列 |

どちらを正とするかを決める。デザインを正とするなら `Skill` と `Career` を簡素化でき、モデルを正とするならデザインに項目を追加する。いずれにせよ決定を記録する。

受入条件に以下を追加する。

- [ ] `Skill` と `Career` のデザインとの不一致が解消され、決定が記録されている
