# プロフィール関連 API (Profile / Career / Skill / SNSリンク / お知らせ) を実装する

| 項目 | 値 |
| ---- | -- |
| Issue | [#10](https://github.com/kishimin-ai-create/profile-hub/issues/10) |
| 英語タイトル | Implement the profile-related APIs (Profile / Career / Skill / SocialLink / Announcement) |
| マイルストーン | v1 |
| ラベル | enhancement,area:api |

---

## 解決したい課題

自己紹介サイトの中心である about ページ (プロフィール、経歴、スキル、SNSリンク) と、お知らせを表示・管理する API がない。これらがないと公開サイトの about ページと管理画面のプロフィール編集が実装できない。

## 提案する解決策

プロフィール関連リソースの API を、公開用と管理用に分けて実装する。

### 対象リソース

| リソース | 内容 |
| -------- | ---- |
| `Profile` | 名前、肩書き、紹介文、アバター。単一レコード |
| `Career` | 経歴。期間、組織、役割、説明 |
| `Skill` | スキル名、カテゴリ、習熟度 |
| `SocialLink` | 種別 (GitHub / X など)、URL、表示順 |
| `Announcement` | お知らせ。タイトル、本文、公開状態、公開日時 |

### 公開API (認証不要)

| メソッド | パス |
| -------- | ---- |
| `GET` | `/api/public/profile` (Profile / Career / Skill / SocialLink をまとめて返す) |
| `GET` | `/api/public/announcements` |

### 管理API (認証必須)

- `Profile` — 取得・更新
- `Career` / `Skill` / `SocialLink` / `Announcement` — 一覧・作成・更新・削除・並び替え

### 要件

- `Career` `Skill` `SocialLink` は表示順を管理者が指定でき、公開APIはその順序で返す
- `Announcement` は公開状態を持ち、公開APIは公開済みのもののみ返す
- `SocialLink` の URL は形式を検証し、`http` / `https` 以外のスキームを拒否する
- 公開APIはメールアドレスなど公開対象でない項目を返さない

## 代替案

- **プロフィール・経歴・スキル・SNSリンクを個別の公開エンドポイントに分ける** — about ページの表示に複数リクエストが必要になる。1 エンドポイントへの集約を採用して不採用。
- **プロフィールを `Setting` の JSON カラムで持つ** — スキーマ変更は不要になるが、経歴やスキルの並び替え・検証が扱いにくい。テーブル分割を採用して不採用。

## 受入条件

- [ ] 認証なしで公開APIからプロフィール、経歴、スキル、SNSリンクを取得できる
- [ ] 経歴・スキル・SNSリンクが管理者の指定した表示順で返る
- [ ] 未公開のお知らせが公開APIに現れない
- [ ] 認証済み管理者が各リソースを作成・更新・削除・並び替えできる
- [ ] 認証なしで管理APIを呼ぶと 401 が返る
- [ ] `javascript:` などのスキームを持つ SNS リンク URL が 400 で拒否される
- [ ] 公開APIの応答に管理者のメールアドレスやパスワードハッシュが含まれない

## 対象外

- アバター画像のアップロード — v2 (当面は URL 指定)
- お問い合わせフォームの送信API — 別Issue

## 補足

- 依存: データモデルIssue、API 基盤Issue、認証Issue


---

## デザインレビューによる更新 (2026-09-04)

デザイン (`review/figma-ui-design-20260904.md` の指摘 H-6) はこのモデルより狭い。

- `Skill` はカテゴリも習熟度も無いタグの羅列として描画されている (`#2:85`)
- `Career` は 1 行 1 文字列 `2026 — Frontend Engineering / Product development` として描画されている (`#2:76`)

実装前にデータモデルIssue (#6) と併せて決める。リソースをデザインに合わせて簡素化するか、デザインを拡張するかのいずれか。
