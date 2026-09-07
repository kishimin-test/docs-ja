# Zipnami 要件定義書 v1.0

| 項目         | 内容               |
| ------------ | ------------------ |
| プロダクト名 | Zipnami            |
| リポジトリ   | random-postal-code |
| 更新日       | 2026-09-07         |
| ステータス   | MVP 要件確定       |

---

## 1. プロダクト概要

### 1.1 名称

| 用途                     | 名称                  |
| ------------------------ | --------------------- |
| プロダクト名             | Zipnami               |
| Web 表示名               | Zipnami               |
| Android アプリ表示名     | Zipnami               |
| Google Play ストア掲載名 | Zipnami               |
| リポジトリ名             | `random-postal-code`  |
| Cloudflare Pages 名      | `zipnami`             |
| Cloudflare Worker 名     | `random-postal-code`  |
| Android パッケージ名     | `jp.kishimin.zipnami` |

`Zipnami` は `ZIP`（郵便番号）と `nami`（波）を組み合わせた造語。

ランダムな郵便番号をきっかけに、日本のどこかへ流れ着く体験を表現する。

ユーザーに見えるブランド名には `Zipnami` を使用し、内部のリポジトリ名・Worker 名には `random-postal-code` を使用する。

Android パッケージ名は `jp.kishimin.zipnami` とする。

Google Play 公開後はパッケージ名を変更できないため、製品版公開前に最終確認する。

### 1.2 コンセプト

全国の郵便番号から1件をランダムに選び、その住所と地図を表示する。

利用者はボタンを押すだけで、日本のどこかへランダムに飛ぶことができる。

「住所検索」ではなく「知らない場所との偶然の出会い」を楽しむプロダクトとする。

---

## 2. 目的

Zipnami の目的は以下とする。

- 日本全国の知らない場所と偶然出会える
- 郵便番号を起点に住所と地図を眺められる
- 1操作で結果が得られる
- Web と Android の両方で利用できる
- Zipnami 自身による不要な個人情報の収集・保存を行わない
- 第三者サービスによるデータ処理を利用者へ適切に開示する
- 小さくシンプルな構成で継続運用できる

---

## 3. MVP スコープ

### 3.1 Web

MVP では以下を提供する。

- ランダムな郵便番号の生成
- 郵便番号表示
- 住所表示
- Google Maps 埋め込み表示
- Google Maps で開く
- 生成履歴表示
- Google AdSense 広告

### 3.2 Android

MVP では以下を提供する。

- ランダムな郵便番号の生成
- 郵便番号表示
- 住所表示
- 外部地図アプリまたはブラウザで住所を開く
- 生成履歴表示
- アプリ情報表示
- Google Mobile Ads SDK / AdMob によるバナー広告

Android アプリ内への Google Maps iframe または Google Maps SDK による地図埋め込みは行わない。

### 3.3 MVP 対象外

以下は MVP では実装しない。

- ユーザー登録
- ログイン
- サーバー側履歴保存
- お気に入り
- SNS 投稿
- 都道府県指定
- 市区町村指定
- 位置情報取得
- GPS
- プッシュ通知
- 課金
- iOS アプリ
- 多言語対応
- Android アプリ内地図
- インタースティシャル広告
- リワード広告
- App Open 広告

---

## 4. Web 画面

Web は1画面構成とする。

### 4.1 初期状態

初期表示時にはまだ郵便番号を生成しない。

以下を表示する。

- Zipnami ロゴ / タイトル
- 簡単な説明
- 「ランダム生成する」ボタン
- 空の結果領域
- 広告領域

### 4.2 生成後

ランダム生成後は以下を表示する。

- 郵便番号
- 該当するすべての住所
- Google Maps
- Google Maps で開く
- 履歴
- 広告

### 4.3 再生成

結果表示後も「ランダム生成する」を使用できる。

新しい結果を生成した場合、生成結果を履歴の先頭へ追加する。

履歴は最大20件とし、21件目が追加された場合は最も古い履歴を削除する。

---

## 5. 郵便番号データ

### 5.1 データソース

日本郵便が公開する郵便番号データを使用する。

アプリケーション実行時に日本郵便へ API リクエストは行わない。

ビルド用スクリプトによって必要な形式へ変換し、アプリケーション用データとして利用する。

### 5.2 使用データ

最低限以下を保持する。

```ts
type Address = {
  prefecture: string;
  city: string;
  town: string;
};

type PostalCode = {
  postalCode: string;
  addresses: Address[];
};
```

### 5.3 正規化

ランダム抽選の単位は、日本郵便の元データのレコードではなく、**一意な郵便番号**とする。

同一郵便番号に複数の住所レコードが存在する場合、データ生成時に1つの `PostalCode` へ正規化する。

正規化ルールは以下とする。

1. 郵便番号をキーとしてレコードをグループ化する
2. 都道府県・市区町村・町域の組を1つの `Address` として保持する
3. 完全に同一の `Address` は重複を除去する
4. 重複除去後の `Address` は日本郵便データの出現順を維持する
5. 同一郵便番号に対応するすべての `Address` を `addresses` へ保持する
6. 複数住所を1つの文字列や代表住所へ統合しない
7. 正規化後は1郵便番号につき1つの `PostalCode` のみ存在させる

例:

```text
元データ

123-4567 東京都 ○○市 A町
123-4567 東京都 ○○市 B町
123-4567 東京都 ○○市 A町

↓

正規化後

123-4567
addresses:
- 東京都 ○○市 A町
- 東京都 ○○市 B町
```

同一郵便番号内で都道府県または市区町村が異なる場合も、それぞれを別の `Address` として保持する。

これにより有効な住所情報を代表値によって失わない。

### 5.4 ランダム抽選

正規化済みの一意な郵便番号一覧から1件を一様ランダムに選択する。

すべての一意な郵便番号は同一確率で選択される。

元データに複数の住所レコードを持つ郵便番号が、他の郵便番号より高い確率で選択されてはならない。

### 5.5 データ更新

郵便番号データ更新はアプリケーション実行時ではなく、データ生成処理として行う。

```text
日本郵便データ
      ↓
scripts/build-data.ts
      ↓
正規化
      ↓
アプリ用データ
      ↓
Web / Worker
```

---

## 6. Web 技術構成

### 6.1 Frontend

以下を使用する。

- React
- TypeScript
- Vite

### 6.2 Backend

以下を使用する。

- Cloudflare Workers
- Hono
- TypeScript

### 6.3 Cloudflare

React SPA は Cloudflare Pages、Hono API は Cloudflare Workers 上で提供する。

React と Hono は別の Cloudflare プロジェクトとして、独立してビルド・デプロイする。

React は Vite で静的ビルドし、生成した `dist` を Cloudflare Pages から配信する。

Hono は Cloudflare Worker としてデプロイする。

React はビルド時環境変数 `VITE_API_BASE_URL` から Hono API の公開 URL を取得する。

Hono API は許可された Cloudflare Pages の Origin に対してのみ CORS レスポンスヘッダーを返す。

Android からのアクセスはブラウザ CORS の対象外だが、React と同じ公開 API 契約を利用する。

### 6.4 Web 構成

```text
Browser
   │
   ▼
Cloudflare Pages
   │ React SPA
   │ HTTPS / CORS
   ▼
Cloudflare Workers
   │ Hono API
   └── 正規化済み郵便番号データ
```

React と Hono は別サービスとしてデプロイする。

### 6.5 API

最低限以下を提供する。

```text
GET /api/random
```

レスポンス例:

```json
{
  "postalCode": "100-0001",
  "addresses": [
    {
      "prefecture": "東京都",
      "city": "千代田区",
      "town": "千代田"
    }
  ]
}
```

### 6.6 エラー

API エラー時は適切な HTTP ステータスコードと JSON を返す。

例:

```json
{
  "error": {
    "code": "INTERNAL_ERROR",
    "message": "郵便番号の生成に失敗しました"
  }
}
```

---

## 7. Web プロジェクト構成

### 7.1 Monorepo

Web と Android を同一リポジトリで管理する。

workspace を使用した monorepo とする。

```text
random-postal-code/
├── apps/
│   ├── web/
│   │   ├── frontend/
│   │   │   ├── src/
│   │   │   │   ├── app/
│   │   │   │   │   ├── providers/
│   │   │   │   │   ├── routes/
│   │   │   │   │   ├── views/
│   │   │   │   │   └── App.tsx
│   │   │   │   ├── api/
│   │   │   │   ├── components/
│   │   │   │   ├── features/
│   │   │   │   ├── hooks/
│   │   │   │   ├── lib/
│   │   │   │   └── main.tsx
│   │   │   ├── index.html
│   │   │   ├── vite.config.ts
│   │   │   ├── tsconfig.json
│   │   │   └── package.json
│   │   │
│   │   └── backend/
│   │       ├── src/
│   │       │   └── index.ts
│   │       ├── wrangler.jsonc
│   │       ├── tsconfig.json
│   │       └── package.json
│   │
│   └── mobile/
│       ├── app/
│       ├── assets/
│       ├── app.json
│       ├── eas.json
│       ├── tsconfig.json
│       └── package.json
│
├── packages/
│   └── shared/
│       ├── src/
│       │   ├── types.ts
│       │   └── index.ts
│       ├── package.json
│       └── tsconfig.json
│
├── scripts/
│   └── build-data.ts
│
├── package.json
└── tsconfig.json
```

Vite の生成物ディレクトリをソース構成として管理しない。

`dist/client` などを手動で責務分割する設計は採用しない。

### 7.2 Hono Worker Entry

Hono の Worker エントリは以下とする。

```text
apps/web/backend/src/index.ts
```

### 7.3 React

React アプリケーションは以下に配置する。

```text
apps/web/frontend/src/
```

アプリ全体の Provider、ルーティング、ページ単位の View は `apps/web/frontend/src/app/` に配置する。

機能固有の UI と状態は `apps/web/frontend/src/features/<feature>/` に配置する。

### 7.4 Shared Package

Web と Android で共有できる純粋な TypeScript コードのみ `packages/shared` に配置する。

対象例:

- 型
- 郵便番号レスポンス型
- バリデーション
- 純粋関数

React DOM や React Native に依存する UI コードは共有しない。

---

## 8. パッケージ管理

### 8.1 Runtime / Package Manager

Bun を使用する。

ルート workspace で Web Frontend / Web Backend / Mobile / Shared package を管理する。

### 8.2 Web Build

Web のビルドは Vite を使用する。

`bun build` を Web アプリケーションのバンドラとして直接使用しない。

Bun は以下の用途で使用する。

- パッケージ管理
- workspace 管理
- npm scripts の実行
- データ生成スクリプト実行

### 8.3 Web Development

React の開発サーバーには Vite を使用する。

Hono API は Wrangler のローカル開発環境で実行する。

React のローカル開発環境には、ローカル Hono API の URL を `VITE_API_BASE_URL` として設定する。

開発用 Hono API の CORS allowlist には、Vite 開発サーバーの Origin のみを追加する。

---

## 9. Android アプリ

### 9.1 技術構成

以下を使用する。

- React Native
- Expo
- TypeScript
- Expo Router

### 9.2 配布

Android アプリは Google Play で配布する。

ビルドには EAS Build を使用する。

配信形式は AAB とする。

### 9.3 API

Android アプリは Cloudflare Workers 上の Hono API を利用する。

```text
Android
   ↓ HTTPS
Cloudflare Workers
   ↓
Hono
   ↓
正規化済み郵便番号データ
```

郵便番号データ一式を Android アプリへ内包しない。

### 9.4 Mobile 画面

MVP は1画面を基本とする。

以下を表示する。

- Zipnami ロゴ / タイトル
- ランダム生成ボタン
- 郵便番号
- 該当するすべての住所
- 各住所の「地図で見る」
- 履歴
- バナー広告
- アプリ情報への導線

### 9.5 地図

Android MVP ではアプリ内に地図 SDK を組み込まない。

各住所の「地図で見る」を押した場合、その住所を使用して外部地図アプリまたはブラウザを開く。

これにより Google Maps SDK の導入、Google Maps SDK 用 API キー管理、位置情報権限を MVP では不要とする。

### 9.6 履歴

履歴は端末内で保持する。

サーバーには保存しない。

最大20件とする。

新しい結果を履歴の先頭へ追加する。

21件目が追加された場合は最も古い履歴を削除する。

同じ郵便番号が複数回生成された場合も、それぞれ独立した履歴として保持する。

Web と Android の履歴同期は行わない。

### 9.7 広告

Google Mobile Ads SDK / AdMob を使用する。

MVP ではバナー広告のみを表示する。

広告読み込み失敗によって郵便番号生成などの主要機能を利用不能にしない。

---

## 10. Expo / Monorepo

Expo アプリはルート workspace の一部として管理する。

Expo の monorepo サポートを利用する。

Metro の monorepo 対応について独自設定を追加しない。

Expo の標準設定で動作する限り、`watchFolders`、`resolver.nodeModulesPath`、`resolver.extraNodeModules` 等の monorepo 用手動設定は行わない。

`metro.config.js` が不要な場合は作成しない。

---

## 11. Google Maps

### 11.1 Web

Web では以下を提供する。

- Google Maps 埋め込み表示
- Google Maps で開く

埋め込みには **Google Maps Embed API** を使用する。

### 11.2 Google Cloud

Google Maps Embed API を利用する Google Cloud プロジェクトについて以下を必須とする。

- Billing を有効化する
- Maps Embed API を有効化する
- Web 用 API キーを発行する

Web 用 API キーはブラウザから Google Maps Embed API を利用するためクライアントへ公開される。

したがって Workers Secret のみに格納する設計とはしない。

### 11.3 API キー制限

Web 用 API キーには Google Cloud Console で制限を設定する。

Application restrictions:

```text
HTTP referrers (web sites)
```

本番環境では Zipnami の Web Origin のみ許可する。

開発時に必要な localhost は開発用キーにのみ設定する。

API restrictions:

```text
Maps Embed API
```

他の Google Maps API や Google Cloud API には使用できないよう制限する。

本番用と開発用の API キーは分離する。

API キーを Git リポジトリへコミットしない。

### 11.4 Google Maps で開く

すべての住所を一覧表示し、Web の埋め込み地図では利用者が選択した住所を表示する。

初期表示では `addresses` の先頭の住所を選択する。

各住所について、その住所をクエリとした Google Maps の URL を生成し、外部ページとして開けるようにする。

Google Maps の埋め込み表示が失敗した場合でも、郵便番号・住所・履歴等の主要機能は維持する。

以下によってランダム生成自体を失敗させない。

- Maps API 障害
- API キーエラー
- Billing エラー
- 地図読み込みエラー
- ネットワーク上の地図取得失敗

### 11.5 Android

Android では外部地図アプリまたはブラウザへ遷移する。

アプリ内 Google Maps SDK は MVP 対象外とする。

Android 用 Google Maps SDK API キーは MVP では使用しない。

### 11.6 位置情報

ユーザーの現在位置は取得しない。

位置情報権限も要求しない。

---

## 12. 履歴

### 12.1 共通仕様

Web / Android ともに履歴は最大 **20件** とする。

履歴は新しい順に表示する。

```text
index 0
最新

...

index 19
最古
```

新規生成時は生成結果を先頭へ追加する。

履歴が20件を超えた場合は末尾の最古データから削除する。

### 12.2 Web

Web の履歴はブラウザ側で保持する。

サーバーには保存しない。

最大20件とする。

### 12.3 Android

Android の履歴は端末側で保持する。

サーバーには保存しない。

最大20件とする。

### 12.4 重複

同一郵便番号が複数回生成された場合も履歴から除外しない。

生成結果を時系列の記録として扱い、それぞれ別の履歴として保持する。

### 12.5 個人情報

履歴をユーザーアカウントへ紐付けない。

Zipnami 独自のユーザー識別 ID を発行しない。

---

## 13. 広告

### 13.1 広告プロバイダー

MVP では Google の広告サービスを使用する。

Web:

```text
Google AdSense
```

Android:

```text
Google Mobile Ads SDK / AdMob
```

広告プロバイダーを実装者判断で変更しない。

### 13.2 Web 広告

Web では Google AdSense の広告を表示する。

MVP では結果領域とは独立した広告枠を1つ設けることを基本とする。

以下を禁止する。

- ランダム生成ボタンに重なる広告
- Google Maps の操作を妨げる広告
- 郵便番号・住所を隠す広告
- Zipnami の操作ボタンと誤認させる配置
- 意図しない広告クリックを誘発する配置

### 13.3 Android 広告

Android では Google Mobile Ads SDK / AdMob を使用する。

MVP では **バナー広告のみ**を使用する。

以下は MVP 対象外とする。

- インタースティシャル広告
- リワード広告
- App Open 広告

広告は主要操作を妨げない位置へ配置する。

ランダム生成ボタンと広告を近接させ、誤タップを誘発する UI にしない。

### 13.4 広告によるデータ処理

広告 SDK または広告サービスが以下のような技術情報を処理する可能性があることを前提とする。

- IP アドレス
- 端末情報
- 広告識別子
- 広告操作情報
- 利用状況に関する情報

実際に処理されるデータは、導入時点の SDK、設定、ユーザーの Consent 状態等によって決まる。

Zipnami 独自のユーザー ID と広告識別子を関連付けない。

Zipnami のサーバーへ広告識別子を保存しない。

### 13.5 Consent

広告サービスの利用にあたり、ユーザーの地域、適用法令、Google のポリシー上 Consent が必要な場合は、Google が提供する CMP / User Messaging Platform を使用する。

Consent が必要な地域では、必要な同意状態を確認してから広告要求を行う。

パーソナライズ広告への同意が得られない場合は、利用可能な範囲で非パーソナライズ広告または制限付き広告を使用する。

Consent の取得失敗によって Zipnami の主要機能を利用不能にしない。

### 13.6 広告読み込み失敗

広告読み込み失敗によって以下を失敗させない。

- 郵便番号生成
- 住所表示
- Google Maps 表示
- 外部地図起動
- 履歴
- その他の主要機能

広告読み込み失敗時は広告枠を非表示または空状態とする。

広告取得の無限再試行を行わない。

### 13.7 開発・テスト

開発・自動テスト・ストア審査前の確認では Google が提供するテスト広告またはテスト用広告 ID を使用する。

開発中に本番広告を意図的にクリックしない。

広告 Unit ID 等は開発環境と本番環境で分離する。

---

## 14. プライバシー

### 14.1 Zipnami が管理するデータ

Zipnami 自身は、MVP においてユーザーの個人情報をアプリケーションデータとして意図的に収集・保存しない。

以下を Zipnami のサーバー側アプリケーションデータとして保存しない。

- 氏名
- メールアドレス
- 電話番号
- 現在位置
- 端末位置情報
- ユーザーアカウント
- サーバー側生成履歴
- 広告識別子

ただし、Cloudflare、Google Maps、Google AdSense、Google Mobile Ads SDK / AdMob 等の第三者サービスが、サービス提供、セキュリティ、広告配信等の目的で技術情報を処理する可能性がある。

「Zipnami がサーバーへ保存しない」ことと「第三者サービスを含め一切のデータ処理が存在しない」ことを同一視しない。

### 14.2 プライバシーポリシー

Google Play 掲載および Web 利用者向けにプライバシーポリシーを Web 側で公開する。

```text
/privacy
```

最低限以下を記載する。

- Zipnami が独自に収集・保存するデータ
- Zipnami が保存しないデータ
- Google Maps の利用
- Google AdSense の利用
- Google Mobile Ads SDK / AdMob の利用
- 第三者サービスによるデータ処理の可能性
- 履歴が端末内のみへ保存されること
- 位置情報を取得しないこと
- Consent に関する事項
- 問い合わせ先

### 14.3 Google Play Data Safety

Google Play の Data Safety は、実際に使用する Google Mobile Ads SDK およびその他 SDK のバージョン・設定に基づいて申告する。

広告 SDK が収集・共有するデータについて、Zipnami 自身が保存していないことを理由に「収集なし」と判断しない。

Data Safety の内容と以下を一致させる。

- 実際のアプリ実装
- 使用 SDK
- SDK 設定
- Consent 処理
- プライバシーポリシー

### 14.4 外部サービス

Google Maps 等の外部サービスへ遷移した後は、遷移先サービスの利用規約およびプライバシーポリシーが適用される。

---

## 15. Google Play

### 15.1 アプリ識別子

```text
jp.kishimin.zipnami
```

### 15.2 ストア表示名

```text
Zipnami
```

### 15.3 配信形式

```text
Android App Bundle (.aab)
```

### 15.4 必要素材・情報

以下を用意する。

- アプリアイコン
- スクリーンショット
- フィーチャーグラフィック
- 短い説明
- 詳細説明
- プライバシーポリシー
- データセーフティ申告
- 日本郵便データの出典表記
- 広告に関する申告
- 必要な Consent 対応

### 15.5 権限

MVP では不要な Android 権限を要求しない。

特に以下は要求しない。

- 位置情報
- カメラ
- マイク
- 連絡先

広告 SDK が追加する権限・manifest 項目については production build で確認し、使用目的のない権限が追加されていないことを確認する。

---

## 16. データ出典

日本郵便が公開する郵便番号データを利用する。

アプリ内または情報ページに、郵便番号データの出典を明記する。

データそのものを Zipnami 独自の住所情報として表現しない。

---

## 17. 非機能要件

### 17.1 パフォーマンス

ランダム生成操作から結果表示まで、通常利用で待ち時間を意識させないことを目標とする。

郵便番号抽選のためだけに外部 API へ問い合わせない。

Google Maps や広告の読み込み完了を、郵便番号・住所の結果表示条件としない。

### 17.2 レスポンシブ

Web は以下を対象とする。

- Desktop
- Tablet
- Mobile

### 17.3 アクセシビリティ

最低限以下を満たす。

- キーボードで主要操作が可能
- ボタンに意味の分かるラベルを付ける
- 色だけで状態を伝えない
- 適切な HTML 要素を使用する
- フォーカス状態を視認できる
- エラー内容をテキストでも伝える
- 広告とアプリコンテンツを視覚的に区別できる

### 17.4 エラー

以下を考慮する。

- API 通信失敗
- 郵便番号データ読み込み失敗
- クリップボード操作失敗
- Google Maps 埋め込み失敗
- Google Maps を開けない場合
- 広告読み込み失敗
- Consent 処理失敗
- ネットワーク未接続

エラー後に再試行できる状態を維持する。

地図・広告等の第三者サービス障害によって、郵便番号生成などの主要機能を利用不能にしない。

---

## 18. セキュリティ

MVP では認証機能を持たない。

API は読み取り専用とする。

ユーザー入力を利用して任意の住所・郵便番号を検索する API は提供しない。

サーバー側の秘密情報をフロントエンドへ埋め込まない。

Cloudflare の秘密情報が必要になった場合は Workers Secrets を使用する。

### 18.1 Google Maps API キー

Google Maps Embed API の Web API キーはブラウザから利用されるため、公開されることを前提とする。

秘密情報として隠すのではなく、以下で保護する。

- HTTP Referrer 制限
- Maps Embed API のみに対する API 制限
- 本番用 / 開発用キーの分離
- Google Cloud 上での利用状況監視

API キーを無制限の状態で本番利用しない。

### 18.2 広告

広告サービスにサーバー側の秘密情報が必要となる場合はクライアントへ埋め込まない。

広告 SDK が取得した広告識別子を Zipnami の API へ送信しない。

### 18.3 CORS

Hono API はブラウザからのアクセスに対して CORS allowlist を適用する。

本番環境では Cloudflare Pages の公開 Origin と、明示的に管理された独自ドメインのみを許可する。

`Access-Control-Allow-Origin: *` は使用しない。

許可する method と request header は `GET /api/random` に必要な範囲へ限定する。

Origin の部分一致や前方一致を使用せず、scheme・host・port を含む完全な Origin で照合する。

---

## 19. テスト

### 19.1 Unit Test

最低限以下をテストする。

- 郵便番号データ変換
- 同一郵便番号のグループ化
- 完全に同一な住所の重複除去
- 同一郵便番号に対応する複数住所の保持
- 住所の出現順維持
- 正規化後の郵便番号一意性
- ランダム抽選
- 元レコード数によって抽選対象が重複しないこと
- レスポンス整形
- 履歴追加
- 履歴上限20件
- 21件目追加時の最古履歴削除
- 同一郵便番号の履歴重複保持
- 住所表示用文字列生成

### 19.2 API Test

以下をテストする。

```text
GET /api/random
```

確認項目:

- 200 を返す
- postalCode が存在する
- addresses が1件以上存在する
- 各 address に prefecture・city・town が存在する
- 不正なデータを返さない
- 正規化済みの一意な郵便番号データから結果を返す

### 19.3 Web UI Test

最低限以下を確認する。

- 初期状態
- 生成
- 再生成
- 同一郵便番号に対応するすべての住所表示
- 複数住所からの地図表示対象選択
- Google Maps 表示
- Google Maps 読み込み失敗
- Google Maps で開く
- 履歴
- 履歴20件制限
- 21件目追加時の最古履歴削除
- 広告表示領域
- 広告読み込み失敗
- 広告失敗時にも主要機能が利用できる
- 広告が主要操作を妨げない
- Consent が必要な場合の処理
- エラー
- レスポンシブ表示

### 19.4 Mobile

以下を確認する。

- Android 実機起動
- API 通信
- ランダム生成
- 同一郵便番号に対応するすべての住所表示
- 各住所の外部地図起動
- 履歴
- 履歴20件制限
- 21件目追加時の最古履歴削除
- 外部地図起動
- AdMob バナー広告表示
- テスト広告で動作確認できる
- 広告読み込み失敗時にも主要機能が利用できる
- Consent が必要な場合のフロー
- 不要な権限を要求しない
- オフライン時エラー
- Play 用 production build

### 19.5 Maps / Advertising / Privacy

以下を確認する。

- Maps Embed API が有効になっている
- Google Cloud Billing が有効になっている
- 本番 Maps API キーに HTTP Referrer 制限が設定されている
- Maps API キーが Maps Embed API のみに制限されている
- 本番 / 開発キーが分離されている
- `/privacy` が公開されている
- プライバシーポリシーと実装内容が一致する
- Google Play Data Safety と使用 SDK が一致する
- 本番広告 ID とテスト広告 ID が分離されている

---

## 20. CI/CD

### 20.1 Web Frontend

Vite で生成した静的アセットを Cloudflare Pages へデプロイする。

Pages のビルド時に、本番 Hono API の URL を `VITE_API_BASE_URL` として設定する。

本番デプロイに必要な Maps API キー等の環境設定が存在することを確認する。

### 20.2 Web API

Hono API を Cloudflare Workers へデプロイする。

本番 CORS allowlist には Cloudflare Pages の公開 Origin のみを設定する。

Frontend と API は独立してデプロイおよびロールバックできるようにする。

### 20.3 Android

EAS Build を使用する。

production build では AAB を生成する。

広告設定等は development / production で分離する。

### 20.4 CI

最低限以下を実行する。

```text
install
   ↓
lint
   ↓
typecheck
   ↓
test
   ↓
build
```

Web と Mobile の変更範囲に応じて必要なジョブを実行する。

---

## 21. 開発フェーズ

### M0 — プロジェクト基盤

- monorepo 作成
- workspace 設定
- TypeScript 設定
- lint / format
- test
- CI

### M1 — 郵便番号データ

- 日本郵便データ取得
- データ変換
- 郵便番号単位のグループ化
- 複数住所正規化
- 完全に同一な住所の重複除去
- 複数住所保持
- 一意な郵便番号データ生成
- ランダム抽選
- テスト

### M2 — Web 基盤

- React
- Vite
- Cloudflare Pages
- Pages の SPA fallback
- `VITE_API_BASE_URL`

### M3 — API 基盤

- Cloudflare Workers
- Hono
- CORS allowlist
- ローカル開発環境

### M4 — Web MVP

- Zipnami UI
- ランダム生成
- 郵便番号表示
- 住所表示
- コピー
- Google Maps
- 履歴
- 履歴20件制限
- エラー処理
- レスポンシブ

### M5 — Web 外部サービス

- Google Cloud プロジェクト設定
- Billing 有効化
- Maps Embed API 有効化
- 開発用 Maps API キー
- 本番用 Maps API キー
- HTTP Referrer 制限
- API 制限
- Google AdSense
- Consent 対応
- 広告失敗時処理

### M6 — Web 公開

- Cloudflare Pages デプロイ
- Cloudflare Workers API デプロイ
- 本番 API URL 設定
- 本番 CORS allowlist 設定
- `/privacy`
- 出典表記
- 広告に関する開示
- production 動作確認
- Maps API キー制限確認

### M7 — Mobile 基盤

- Expo
- React Native
- Expo Router
- monorepo 統合
- API Client

### M8 — Mobile MVP

- Zipnami UI
- ランダム生成
- 住所表示
- 履歴
- 履歴20件制限
- 外部地図連携
- 情報画面
- エラー処理
- Google Mobile Ads SDK / AdMob
- バナー広告
- Consent 対応

### M9 — Android 品質確認

- Android 実機確認
- production build
- 権限確認
- 広告 SDK の manifest 確認
- テスト広告確認
- Consent 確認
- アイコン
- adaptive icon
- splash
- レスポンシブ確認

### M10 — Google Play 準備

- `jp.kishimin.zipnami` 最終確認
- ストア掲載情報
- スクリーンショット
- フィーチャーグラフィック
- プライバシーポリシー
- データセーフティ
- 広告申告
- 使用 SDK と Data Safety の整合確認
- AAB

### M11 — Google Play テスト

- テスター確保
- 必要なテストトラックで配布
- フィードバック対応
- クラッシュ確認
- 広告動作確認
- Consent 動作確認

### M12 — Production

- Google Play 本番申請
- 公開
- production 動作確認
- Maps API 利用状況確認
- 広告動作確認

---

## 22. MVP 受入条件

### 22.1 Web

- [ ] Zipnami として表示される
- [ ] ランダムな郵便番号を1件生成できる
- [ ] 一意な郵便番号単位で一様ランダム抽選される
- [ ] 元データのレコード数によって抽選確率が変化しない
- [ ] 同一郵便番号に対応するすべての住所が欠落せず保持される
- [ ] 完全に同一な住所だけが重複除去される
- [ ] 郵便番号を表示できる
- [ ] 同一郵便番号に対応するすべての住所を表示できる
- [ ] 郵便番号をコピーできる
- [ ] 選択した住所を Maps Embed API で Google Maps に表示できる
- [ ] 各住所を Google Maps で開ける
- [ ] Google Cloud Billing が有効になっている
- [ ] Maps API キーに HTTP Referrer 制限が設定されている
- [ ] Maps API キーが Maps Embed API のみに制限されている
- [ ] 本番用と開発用の Maps API キーが分離されている
- [ ] Maps 読み込み失敗時も郵便番号・住所を利用できる
- [ ] 再生成できる
- [ ] 履歴を確認できる
- [ ] 履歴が最大20件に制限される
- [ ] 21件目追加時に最古の履歴が削除される
- [ ] Google AdSense 広告を表示できる
- [ ] 広告読み込み失敗時も主要機能を利用できる
- [ ] 広告が主要操作を妨げない
- [ ] 必要な場合に Consent 処理を実行できる
- [ ] Desktop / Tablet / Mobile で利用できる
- [ ] エラー後に再試行できる
- [ ] React SPA を Cloudflare Pages で公開できる
- [ ] Hono API を Cloudflare Workers で公開できる
- [ ] Pages から本番 API URL を参照できる
- [ ] 許可された Pages Origin から API を呼び出せる
- [ ] 許可されていないブラウザ Origin に CORS アクセスを許可しない

### 22.2 Android

- [ ] Zipnami として表示される
- [ ] Cloudflare Workers API から結果を取得できる
- [ ] 郵便番号を表示できる
- [ ] 同一郵便番号に対応するすべての住所を表示できる
- [ ] 各住所を外部地図アプリまたはブラウザで開ける
- [ ] ランダム再生成できる
- [ ] 履歴を確認できる
- [ ] 履歴が最大20件に制限される
- [ ] 21件目追加時に最古の履歴が削除される
- [ ] 外部地図を開ける
- [ ] Google Mobile Ads SDK / AdMob のバナー広告を表示できる
- [ ] 開発・テスト時にテスト広告を使用できる
- [ ] 広告読み込み失敗時にも主要機能を利用できる
- [ ] 必要な Consent フローを実行できる
- [ ] 不要な位置情報権限を要求しない
- [ ] AAB を生成できる
- [ ] Google Play Data Safety が実際の SDK と設定に一致する
- [ ] Google Play で配信可能な状態になる

### 22.3 Privacy / Advertising

- [ ] `/privacy` を公開できる
- [ ] Google Maps の利用をプライバシーポリシーへ記載する
- [ ] Google AdSense の利用を記載する
- [ ] Google Mobile Ads SDK / AdMob の利用を記載する
- [ ] Zipnami 自身のデータ保存と第三者サービスによるデータ処理を区別して説明する
- [ ] Zipnami のサーバーへ広告識別子を保存しない
- [ ] Google Play Data Safety と実装内容が一致する
- [ ] Consent が必要な地域で適切な処理を行える

### 22.4 Architecture

- [ ] React + TypeScript + Vite を使用する
- [ ] React SPA を Cloudflare Pages で配信する
- [ ] Hono を Cloudflare Worker 上で実行する
- [ ] React と Hono を独立してデプロイ・ロールバックできる
- [ ] React は `VITE_API_BASE_URL` から API URL を取得する
- [ ] Hono は設定された Pages Origin のみを CORS allowlist に含める
- [ ] Web の独自バンドル処理を持たない
- [ ] Web Frontend / Web Backend / Mobile / Shared を workspace monorepo で管理する
- [ ] Expo の標準 monorepo サポートを利用する
- [ ] 郵便番号データ取得のための実行時外部 API 依存を持たない
- [ ] Android に郵便番号データ一式を内包しない
- [ ] Android に Google Maps SDK を導入しない

---

## 23. 確定事項

以下は MVP の確定事項とする。

| 項目                    | 決定                                |
| ----------------------- | ----------------------------------- |
| プロダクト名            | Zipnami                             |
| Web                     | React + TypeScript                  |
| Web bundler             | Vite                                |
| API                     | Hono                                |
| Frontend hosting        | Cloudflare Pages                    |
| API hosting             | Cloudflare Workers                  |
| Frontend/API deployment | 別プロジェクト・独立デプロイ        |
| API URL                 | `VITE_API_BASE_URL`                 |
| Browser API access      | Pages Origin を許可する CORS        |
| Mobile                  | React Native + Expo + TypeScript    |
| Mobile routing          | Expo Router                         |
| Android build           | EAS Build                           |
| Package manager         | Bun                                 |
| Repository              | monorepo                            |
| Workspace               | Bun workspaces                      |
| 郵便番号データ          | 日本郵便公開データ                  |
| データ正規化単位        | 一意な郵便番号                      |
| 複数住所                | 重複除去後 `addresses` にすべて保持 |
| ランダム抽選単位        | 一意な郵便番号                      |
| 抽選確率                | 一意な郵便番号ごとに同一            |
| 実行時住所データ取得    | 外部 API を使用しない               |
| Web 地図                | Google Maps Embed API               |
| Maps Billing            | 有効化必須                          |
| Maps API key            | HTTP Referrer + Maps Embed API 制限 |
| Maps key environment    | 本番 / 開発で分離                   |
| Android 地図            | 外部地図アプリ / ブラウザ           |
| Android Maps SDK        | 使用しない                          |
| Web 履歴                | ブラウザ内・最大20件                |
| Android 履歴            | 端末内・最大20件                    |
| 履歴表示順              | 新しい順                            |
| 履歴超過                | 最古から削除                        |
| 重複履歴                | 保持する                            |
| サーバー履歴            | 保存しない                          |
| Web 広告                | Google AdSense                      |
| Android 広告            | Google Mobile Ads SDK / AdMob       |
| Android 広告形式        | バナーのみ                          |
| 広告失敗時              | 主要機能を継続                      |
| Consent                 | 必要な地域では Google CMP / UMP     |
| 広告識別子              | Zipnami サーバーへ保存しない        |
| ユーザー認証            | なし                                |
| 位置情報                | 取得しない                          |
| Android package         | `jp.kishimin.zipnami`               |

---

## 24. 残論点

MVP の実装を開始するうえでブロックする未決事項はない。

独自ドメインは `workers.dev` で開始し、必要になった時点で追加する。

Google Maps、Google AdSense、Google Mobile Ads SDK / AdMob、Google Play Data Safety、Consent、対象 API レベル、ストア申請要件等は外部プラットフォーム側で変更される可能性がある。

そのため、それぞれの導入・公開時点で最新の公式要件を確認する。

特に広告 SDK の Data Safety 申告内容については、要件定義時点の推測で固定せず、実際に採用した SDK バージョンと設定を基準として確定する。

`jp.kishimin.zipnami` は Google Play への初回公開前に最終確認する。

---

## 25. 実装方針

実装時は各フレームワークの独自構成を作ることより、公式ツールチェーンと標準構成を優先する。

React SPA は Vite で静的ビルドし、Cloudflare Pages から配信する。

Hono API は独立した Cloudflare Worker としてデプロイする。

Frontend は `VITE_API_BASE_URL` で API の公開 URL を参照し、API は Pages の公開 Origin を CORS allowlist で許可する。

Mobile は Expo の monorepo サポートと Expo Router の標準構成を基準とする。

郵便番号データはビルド時に一意な郵便番号単位へ正規化し、実行時ロジックを単純化する。

Google Maps の Web API キーは秘密化できるものとして扱わず、Google Cloud の HTTP Referrer 制限および API 制限によって保護する。

広告は Zipnami の主要機能から独立した付加機能として扱い、広告・Consent・第三者サービスの障害によって郵便番号生成機能を停止させない。

プライバシーについては「Zipnami 自身が保存するデータ」と「第三者 SDK / サービスが処理するデータ」を明確に分離して扱う。

公式ツールチェーンで解決できるものについて、独自のビルド・Metro・デプロイ構成を追加しない。
