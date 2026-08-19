MVP向けにCloudflareの役割を最小化し、**「Cloudflare Pagesによるフロント配信」と「画像生成APIへのレート制限」**に整理した最新版です。高度なWAF・Turnstile・Bot対策・オリジン直接アクセス制限はMVP後に回します。

# mojica MVP リリース計画書

## 1. 概要

mojica MVPをWebアプリとして公開する。

MVPでは、フロントエンド、mojica API、Glyph Forge APIをそれぞれ独立してデプロイする。

フロントエンドはCloudflare Pages、mojica APIおよびGlyph Forge APIはさくらのクラウド上でDockerコンテナとして稼働させる。

Cloudflareは、フロントエンドの配信に加えて、画像生成APIへの大量アクセスをバックエンド到達前に制限するために利用する。

MVPでは広告を導入せず、サービスのリリースと利用状況の確認を優先する。

---

# 2. リリース対象

MVPで以下をリリースする。

| システム        | 役割                                | 実行環境                  |
| --------------- | ----------------------------------- | ------------------------- |
| mojica Web      | ユーザー向けWeb UI                  | Cloudflare Pages          |
| mojica API      | WebとGlyph Forge間のバックエンドAPI | さくらのクラウド / Docker |
| Glyph Forge API | 文字アート画像生成                  | さくらのクラウド / Docker |

モバイルアプリはMVPのリリース対象外とする。

---

# 3. インフラ構成

```text
                    ┌─────────────────────┐
                    │       Browser       │
                    └──────────┬──────────┘
                               │
                    Webページ取得
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Cloudflare Pages   │
                    │     Frontend        │
                    └─────────────────────┘

Browser
   │
   │ POST /images
   ▼
Cloudflare
   │
   │ Rate Limit
   ▼
mojica API
ASP.NET Core
さくらのクラウド
Docker
   │
   ▼
Glyph Forge API
さくらのクラウド
Docker
   │
   ▼
Image Generation
```

Cloudflare Pagesで配信されたSPAからのAPIリクエストは、ユーザーのブラウザからmojica APIへ送信される。

その際、mojica APIの公開ドメインをCloudflare経由とし、Cloudflareでレート制限を行った上で、さくらのクラウド上のmojica APIへ転送する。

---

# 4. Cloudflare

## 導入目的

MVPにおけるCloudflare導入の主な目的は以下とする。

1. Cloudflare Pagesによるフロントエンドの配信
2. 画像生成APIへの大量アクセスの制限
3. 大量アクセスによるバックエンド負荷および想定外の課金高騰の抑制

MVPではCloudflareの機能を必要最小限に利用する。

---

# 5. Cloudflare Pages

mojica WebをCloudflare Pagesへデプロイする。

GitHubリポジトリとCloudflare Pagesを連携し、対象ブランチへの変更を契機としてビルド・デプロイできる構成とする。

Cloudflare Pagesではビルド後の静的ファイルを配信する。

MVPでは以下を担当する。

- HTML / CSS / JavaScriptの配信
- SPAの配信
- HTTPS
- 独自ドメイン
- GitHub連携によるデプロイ

Cloudflare Pagesはmojica APIのプロキシとして使用しない。

APIリクエストはCloudflare Pagesからではなく、ユーザーのブラウザからmojica APIへ送信する。

---

# 6. CloudflareによるAPIレート制限

画像生成処理は通常の静的コンテンツ配信よりサーバー負荷が高いため、`POST /images` をCloudflareによるレート制限の対象とする。

```text
Browser
   │
   │ POST /images
   ▼
Cloudflare
   │
   ├── 制限以内
   │       │
   │       ▼
   │   mojica API
   │
   └── 制限超過
           │
           ▼
       リクエスト拒否
```

一定時間内のリクエスト数が設定した上限を超えた場合、Cloudflare側でリクエストを拒否する。

制限されたリクエストは、さくらのクラウド上のmojica APIまで到達させない。

これにより、画像生成リクエストの急増による以下のリスクを抑制する。

- mojica APIの負荷増加
- Glyph Forge APIの負荷増加
- サーバーリソースの過剰消費
- 想定外のインフラ利用量増加
- インフラ費用の高騰

具体的なリクエスト回数および制限時間については、Glyph Forge APIの処理能力、さくらのクラウドの料金体系、通常利用時の画像生成回数を考慮して決定する。

---

# 7. mojica API

mojica APIはASP.NET Coreで実装する。

さくらのクラウド上でDockerコンテナとして稼働させる。

主な責務：

- Webからの画像生成リクエスト受付
- バリデーション
- i18n
- レート制限
- HEXからRGBへの変換
- Glyph Forge APIの呼び出し
- エラーハンドリング
- 生成画像の返却

mojica APIは、Cloudflareを経由する公開APIとして利用する。

---

# 8. mojica APIによるレート制限

Cloudflareによるレート制限に加えて、mojica API自身でもレート制限を行う。

```text
Browser
   ↓
Cloudflare Rate Limit
   ↓
mojica API Rate Limit
   ↓
Glyph Forge API Rate Limit
```

Cloudflareはバックエンドへ到達する前の制限を担当する。

mojica APIのレート制限は、API自身およびGlyph Forge APIを保護するためのバックエンド側の制限として利用する。

レート制限を超過した場合は `429 Too Many Requests` を返却する。

可能な場合は `Retry-After` ヘッダーを返却する。

---

# 9. Glyph Forge API

Glyph Forge APIはさくらのクラウド上でDockerコンテナとして稼働させる。

画像生成処理を担当する。

mojica WebからGlyph Forge APIを直接呼び出さない。

通信経路は以下とする。

```text
mojica Web
    ↓
mojica API
    ↓
Glyph Forge API
```

フロントエンドはGlyph Forge APIの仕様やエンドポイントを意識しない。

Glyph Forge API自身のレート制限も維持する。

---

# 10. Docker

mojica APIとGlyph Forge APIはDockerイメージとして管理する。

それぞれ独立したコンテナとして実行する。

Dockerを利用することで、ローカル開発環境と本番環境の差異を小さくし、再現可能なデプロイ環境を構築する。

---

# 11. 通信

ユーザーと公開サービス間の通信はHTTPSを使用する。

フロントエンド：

```text
Browser
   │
   │ HTTPS
   ▼
Cloudflare Pages
```

API：

```text
Browser
   │
   │ HTTPS
   ▼
Cloudflare
   │
   ▼
mojica API
```

バックエンド：

```text
mojica API
   │
   ▼
Glyph Forge API
```

mojica APIとGlyph Forge API間の通信方式は、さくらのクラウド上のネットワーク構成に合わせて決定する。

---

# 12. CORS

mojica APIではCORSを設定する。

本番環境では、原則としてmojica WebのOriginのみを許可する。

開発環境ではローカル開発用Originを別途許可する。

すべてのOriginを無条件に許可する設定は本番環境では使用しない。

CORSはブラウザからのアクセスを制御するための仕組みであり、大量アクセス対策やセキュリティ対策としてCloudflareのレート制限を代替するものではない。

---

# 13. 環境変数

環境によって変更される値はソースコードへ直接記述せず、環境変数または設定ファイルとして管理する。

対象例：

- Glyph Forge API URL
- CORS許可Origin
- APIタイムアウト
- レート制限設定
- 実行環境

秘密情報が必要になった場合はGitリポジトリへコミットしない。

---

# 14. エラー・ログ

mojica APIおよびGlyph Forge APIでは、運用上必要なログを記録する。

主な対象：

- APIエラー
- Glyph Forge API呼び出し失敗
- タイムアウト
- レート制限
- 予期しない例外

個人情報や不要なリクエスト内容をログへ記録しない。

内部の例外情報やスタックトレースはユーザーへ返却しない。

---

# 15. 監視

MVPでは最低限、以下を確認できる状態を目標とする。

- Webがアクセス可能か
- mojica APIが稼働しているか
- Glyph Forge APIが稼働しているか
- APIエラーが継続的に発生していないか
- サーバーリソースが逼迫していないか
- Cloudflareのレート制限が大量に発生していないか

高度な監視基盤の構築はMVPでは必須としない。

---

# 16. 広告

MVPでは広告を導入しない。

MVPの目的は、サービスを公開して実際の利用状況を確認することであり、広告による収益化は優先しない。

広告を導入すると以下の追加対応が発生する可能性がある。

- 広告表示領域のUI設計
- 広告サービスとの連携
- プライバシーポリシー
- Cookie・トラッキングへの対応
- 同意管理
- パフォーマンスへの影響確認

これらは画像生成というmojicaの主要価値とは直接関係しないため、MVPでは対象外とする。

---

# 17. MVP後の広告検討

サービスの利用状況が確認できた段階で広告導入を再検討する。

判断材料として以下を確認する。

- アクセス数
- 画像生成回数
- 継続利用状況
- サーバー運用コスト
- Glyph Forge APIの処理負荷

広告を導入する場合は、画像生成操作を妨げない場所への配置を基本とする。

画像生成ボタン付近や画像生成フローの途中に広告を挿入することは避ける。

---

# 18. MVPにおけるCloudflare対象外

MVPではCloudflareの高度なセキュリティ機能を必要以上に導入しない。

以下はMVPの必須要件としない。

- Cloudflare Turnstile
- 高度なBot対策
- 複雑なWAFルール
- 地域ベースのアクセス制限
- 高度なDDoS対策設定
- Cloudflare Access
- オリジンサーバーへの直接アクセスを完全に遮断するための追加構成

これらは実際のアクセス状況や問題を確認した後、必要に応じて導入する。

---

# 19. MVP後のセキュリティ強化

大量アクセス、不正Bot、Cloudflareを迂回したアクセスなどが実際に問題となった場合は、段階的に対策を追加する。

候補：

1. オリジンサーバーへの直接アクセス制限
2. Cloudflare Turnstile
3. WAFルールの追加
4. Bot対策
5. より詳細なアクセス制御

MVPでは先回りしてすべてを実装せず、必要になった対策から追加する。

---

# 20. リリース前確認

リリース前に以下を確認する。

- mojica Webの本番ビルド
- Cloudflare Pagesへのデプロイ
- 独自ドメイン
- HTTPS
- SPAルーティング
- mojica APIのDockerイメージ
- Glyph Forge APIのDockerイメージ
- mojica APIへのHTTPSアクセス
- Cloudflare経由でのmojica APIアクセス
- Cloudflareの `POST /images` レート制限
- mojica API自身のレート制限
- mojica API → Glyph Forge API通信
- Glyph Forge APIのレート制限
- CORS
- 環境変数
- バリデーション
- 日本語・英語切り替え
- APIエラーハンドリング
- 画像生成
- PNG画像の自動ダウンロード
- PC表示
- スマートフォン表示
- 基本的なアクセシビリティ

---

# 21. リリース方針

MVPでは、セキュリティ機能や運用機能を必要以上に作り込むことよりも、実際に利用可能な状態でサービスを公開することを優先する。

MVPにおける基本構成は以下とする。

```text
Browser
   │
   ├── Web
   │     ↓
   │   Cloudflare Pages
   │
   └── POST /images
          ↓
       Cloudflare
       Rate Limit
          ↓
       mojica API
       Rate Limit
          ↓
       Glyph Forge API
       Rate Limit
```

リリース時点では、

```text
Web
↓
画像生成
↓
自動ダウンロード
```

というmojicaの主要フローを安定して利用できる状態を完成条件とする。

広告、モバイルアプリ、画像プレビュー、ログイン、保存、SNS共有、高度なCloudflareセキュリティ対策などはMVPのリリース条件に含めない。
