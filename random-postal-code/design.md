# Zipnami MVP 設計書

## 1. 文書の目的

この文書は、Zipnami MVP の現在のプロダクト設計と技術契約の正本である。実装タスクと個別の受入条件は [GitHub Issues](https://github.com/kishimin/random-postal-code/issues) で管理し、詳細な契約は [api-design.md](./api-design.md) と [ui-design.md](./ui-design.md) で管理する。

## 2. プロダクト範囲

Zipnami は、日本郵便の公開データから一意な郵便番号を一様ランダムに選び、その郵便番号に対応するすべての住所を Web と Android で表示する。

MVP に含める機能は次のとおりである。

- ランダムな郵便番号の生成、再生成、コピー
- 同一郵便番号に対応する全住所の表示
- Web の埋め込み Google Maps と住所ごとの外部地図リンク
- Android の住所ごとの外部地図リンク
- Web と Android それぞれの端末内履歴
- Web の Google AdSense と Android の AdMob バナー広告
- 必要な地域での Consent 処理
- プライバシーポリシーと日本郵便データの出典表示
- Cloudflare Pages、Cloudflare Workers、Google Play への配布

MVP に含めないものは、認証、ユーザーアカウント、お気に入り、履歴同期、地域フィルター、GPS、位置情報取得、SNS 投稿、Push 通知、決済、iOS、多言語 UI、Android の埋め込み地図である。

## 3. システム構成

```text
Japan Post CSV
      |
      v (build-time normalization)
packages/postal-data ---> apps/web/backend (Hono on Workers)
                                  |
                       GET /api/random over HTTPS
                         /                     \
                        v                       v
apps/web/frontend (React/Vite on Pages)   apps/mobile (Expo/Android)
        |                 |                         |
 browser storage   Google Maps/AdSense      device storage/AdMob
```

ワークスペースは Bun workspaces による monorepo とし、次の所有境界を持つ。

```text
apps/
  web/
    frontend/     # React、Vite、Pages、ブラウザ固有機能
    backend/      # Hono、Workers、公開API、CORS
  mobile/         # React Native、Expo Router、Android固有機能
packages/
  shared/         # クライアントとAPIが共有する純粋な型・検証契約
  postal-data/    # 日本郵便データの取得、正規化、生成物
```

Frontend と Backend は別プロジェクトとして独立にデプロイおよびロールバックできなければならない。Frontend はビルド時の `VITE_API_BASE_URL` から公開 API URL を取得する。

## 4. ドメインとデータ契約

### 4.1 型

```ts
type Address = {
  prefecture: string;
  city: string;
  town: string;
};

type PostalCode = {
  postalCode: string; // ハイフンなし7桁
  addresses: Address[];
};
```

### 4.2 不変条件

- 抽選母集団は元CSVの行ではなく、一意な7桁郵便番号である。
- 各一意郵便番号が選ばれる確率は等しい。
- 同一郵便番号の住所は `addresses` にすべて保持する。
- `prefecture`、`city`、`town` が完全一致する住所だけを重複除去する。
- 住所の順序は日本郵便データの初出順を保持する。
- 実行時に外部の住所検索 API を利用しない。
- Android アプリへ全郵便番号データを同梱しない。

### 4.3 データ生成

日本郵便の公式データをビルド時に取得して UTF-8 へ正規化し、上記の不変条件を満たす決定的な生成物を作る。同じ入力からは同じ順序・同じ内容の生成物を得られなければならない。取得元、取得日、変換処理、およびデータ更新手順を記録する。

## 5. API 契約

### 5.1 ランダム取得

```http
GET /api/random
Accept: application/json
```

成功時は `200 application/json` で `PostalCode` を返す。

```json
{
  "postalCode": "1000001",
  "addresses": [
    {
      "prefecture": "東京都",
      "city": "千代田区",
      "town": "千代田"
    }
  ]
}
```

API は認証、履歴保存、検索、フィルター、書き込み操作を提供しない。データを読み込めない場合は成功レスポンスへ偽装せず、JSON のエラー応答を返す。[api-design.md](./api-design.md) が共有エラーenvelopeとstatus mappingを定義する。

### 5.2 CORS と設定

- 本番では設定された Pages の完全一致 Origin のみを許可する。
- Origin は scheme、host、port の組で比較し、前方一致や部分一致を使わない。
- `Access-Control-Allow-Origin: *` を使用しない。
- 開発環境では明示した Vite Origin だけを許可する。
- 許可するメソッドとヘッダーは `GET /api/random` に必要な範囲へ限定する。
- Workers の秘密情報を Frontend の環境変数、配布物、Git 履歴へ含めない。

## 6. クライアント設計

### 6.1 共通状態

Web と Android は少なくとも次の状態を区別する。

- 初期状態: 自動取得せず、生成操作を表示する。
- Loading: 進行中の要求を示し、重複送信を防ぐ。
- Success: 郵便番号と全住所を表示する。
- Error: オフライン、API失敗、不正レスポンスを理解可能かつ再試行可能に示す。

再生成の成功時は現在結果を置き換え、履歴の先頭へ追加する。失敗時は直前の成功結果を不要に失わない。

### 6.2 Web

- React、TypeScript、Vite の SPA とする。
- Pages の直接URLアクセスと再読み込みで SPA fallback が機能する。
- 選択した住所を Maps Embed API で表示し、全住所に外部地図リンクを設ける。
- 郵便番号をクリップボードへコピーできる。
- 履歴はブラウザ内だけに保存し、新しい順で最大20件、重複を保持する。
- 21件目の追加時は最古の1件を削除する。

### 6.3 Android

- React Native、Expo、TypeScript、Expo Router を使用する。
- `jp.kishimin.zipnami` を最終 package name とする。
- HTTPS API から結果を取得する。
- 履歴は端末内だけに保存し、Web と同じ順序、上限、重複ルールを持つ。
- 各住所を外部地図アプリまたはブラウザで開く。
- Google Maps SDK および位置情報権限を追加しない。

## 7. 外部サービスと失敗境界

Google Maps、外部地図アプリ、AdSense、AdMob、Consent は主要機能から独立した任意依存である。これらの失敗によって、成功済みの郵便番号、住所、履歴、再生成を利用不能にしてはならない。

- Web の Maps API キーは本番用と開発用を分け、HTTP Referrer と Maps Embed API に制限する。
- Android の自動テストと開発では Google のテスト広告IDを使い、本番IDと分離する。
- 広告は主要操作を覆わず、無限再試行しない。
- Consent が必要な地域では、対象広告要求より前に Google CMP または UMP のフローを行う。
- Zipnami API は広告識別子を受信・保存しない。

## 8. プライバシーとアクセシビリティ

`/privacy` とアプリ内情報画面は、Zipnami 自身の保存データと、Google Maps・AdSense・AdMob・Consent SDK が処理するデータを区別して説明する。位置情報を取得しないこと、履歴が端末内だけに保存されること、日本郵便データの出典、問い合わせ方法を明記する。

Web はセマンティックHTML、キーボード操作、可視フォーカス、意味のある名前、動的な結果・エラー通知を提供する。状態を色だけで伝えない。Android はアクセシビリティサービスへ操作名と状態を公開する。広告領域はアプリ内容と識別可能にする。

## 9. 品質とテスト設計

コード変更はテストリストから1振る舞いずつ Red、Green、Refactor の順で進める。テストは実装詳細ではなく観測可能な契約を表現する。

テストサイズは使用ツールや Unit/E2E という種別ではなく、実際の依存範囲に基づき Small、Medium、Large へ分類する。リポジトリがファイル命名を定義した後は `.small.test.ts`、`.medium.test.ts`、`.large.test.ts` を使用する。

最低限、次を検証する。

- 正規化: グループ化、完全一致重複除去、順序、複数住所、決定性
- 抽選: 一意郵便番号が抽選単位であり、元レコード数に偏らないこと
- API: 成功、データ失敗、不正状態、CORS許可・拒否
- UI: 初期、Loading、Success、再生成、Error、履歴、地図、任意依存の失敗
- セキュリティ: 秘密情報が配布物へ含まれず、Originが完全一致すること
- リリース: PagesからWorkers、AndroidからWorkers、プライバシー公開、権限、広告設定

各コードブランチの完了前に、リポジトリで実際に定義された整形、型チェックまたはコンパイル、Lint・静的解析、テスト、カバレッジ、ビルドを実行する。カバレッジ全体サマリーの全指標は80%以上とする。定義されていないコマンドは推測せず、スキップ理由を記録する。

## 10. デプロイと運用

- Frontend: `apps/web/frontend` を Cloudflare Pages へ静的デプロイする。
- Backend: `apps/web/backend` を独立した Cloudflare Worker へデプロイする。
- Android: EAS Build で production AAB を生成する。
- Pages と Workers は個別にデプロイ・ロールバックする。
- 本番スモークテストで Pages から Workers を介した生成を確認する。
- デプロイログへ秘密情報を出力しない。
- Google Play の Data Safety、広告申告、Consent、権限は実際の production build と一致させる。

## 11. 完了条件とトレーサビリティ

MVP は [tracker Issue #22](https://github.com/kishimin/random-postal-code/issues/22) 配下の Issue #1〜#21が、それぞれの受入条件とこの設計書を満たして閉じられ、最終受入試験の証跡が記録された時点で完成とする。

設計変更時は、先にこの文書の契約と影響範囲を更新し、対応Issueの受入条件とブランチ計画を同期する。設計判断の理由や不採用案を長期保存する必要が生じた場合は、設計書へ履歴を混在させずADRを追加する。

## 12. 実装時に確定する詳細

次の項目はMVPの外部挙動を変えない範囲で、所有IssueのRed/Greenサイクル中に確定する。この文書だけを根拠に推測実装してはならない。

| 項目 | 所有Issue | 確定条件 |
| --- | ---: | --- |
| パッケージの正確なバージョンと品質コマンド | #1 | 公式ツールチェーンと互換性を確認し、lockfileとworkspace scriptsで固定する |
| Runtime schema実装 | #2 | 共有コードをclientまたはserver frameworkへ結合せずAPI契約を実装する |
| 郵便番号生成物のファイル形式 | #3 | 決定性、Workersのサイズ制約、読み込み失敗を検証して固定する |
| WebとAndroidの履歴保存キー・移行方法 | #7, #14 | 既存データがない初版の最小契約と永続化テストで固定する |
| 本番Pages Origin、Worker URL、プロジェクト名 | #19 | 実際に作成したCloudflareリソースとデプロイ設定で固定する |
| Google SDKの正確な開示とData Safety回答 | #20 | production buildに含まれるSDK、設定、権限を人間が確認して固定する |

外部サービスの課金、APIキー制限、Consent、Google Play申告は本番アカウントへ影響するため、自動化結果だけで最終判断せず、公開前に人間が設定画面とproduction buildを照合する。
