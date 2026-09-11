# Zipnami API設計書

## 1. 適用範囲と責任

この文書はZipnami公開API契約とBackend境界の正本である。[design.md](./design.md) のシステム契約とIssue [#2](https://github.com/kishimin/random-postal-code/issues/2)、[#3](https://github.com/kishimin/random-postal-code/issues/3)、[#4](https://github.com/kishimin/random-postal-code/issues/4)、[#11](https://github.com/kishimin/random-postal-code/issues/11)の受入条件を具体化する。

APIは、正規化済みデータからのランダム抽選、HTTPレスポンス変換、CORS、運用エラー報告を所有する。UI状態、クライアント履歴、地図、広告、認証、検索、日本郵便からの実行時取得は所有しない。

## 2. Runtime境界

- Runtime: Cloudflare Workers
- HTTP framework: Hono
- Entry point: `apps/web/backend/src/index.ts`
- 公開Base URL: Frontendデプロイとは独立してクライアントへ設定する
- Transport: デプロイ環境ではHTTPSのみ
- API prefix: `/api`
- Representation: UTF-8のJSON

PagesとWorkersは別々にデプロイする。WorkerはSPAを配信せず、Pagesは本番APIをproxyしない。

## 3. 共有契約

正本となるTypeScript runtime schemaと導出型は、React、React Native、Hono、Cloudflareへ依存しない`packages/shared`に置く。

```ts
type Address = {
  prefecture: string;
  city: string;
  town: string;
};

type PostalCode = {
  postalCode: string;
  addresses: [Address, ...Address[]];
};

type ApiErrorCode =
  | "INVALID_REQUEST"
  | "NOT_FOUND"
  | "METHOD_NOT_ALLOWED"
  | "DATA_UNAVAILABLE"
  | "INTERNAL_ERROR";

type ApiErrorResponse = {
  error: {
    code: ApiErrorCode;
    message: string;
    requestId: string;
  };
};
```

HTTP境界へ入出力するデータはruntime schemaで検証する。`postalCode`は`^[0-9]{7}$`に一致し、各住所フィールドは文字列、`addresses`は1件以上とする。データ生成処理が完全な正規化レコードを作る責任を持ち、APIは不正な生成物を部分的な成功として返さない。

## 4. Endpoint契約

### 4.1 `GET /api/random`

一意な郵便番号から1件を一様ランダムに選び、関連する正規化済み住所をすべて返す。

```http
GET /api/random HTTP/1.1
Accept: application/json
```

path parameter、query parameter、request bodyは受け付けない。未知のquery parameterは`400`とし、誤記や未対応filterを暗黙に無視しない。

成功レスポンス:

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Cache-Control: no-store
X-Request-Id: 01JEXAMPLE0000000000000000
```

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

`Cache-Control: no-store`により、中間cacheが再生成操作を同じ応答へ固定することを防ぐ。ランダム抽選のため、連続して同じ郵便番号になること自体は正常である。

### 4.2 エラーレスポンス

| 条件 | Status | Code | 再試行方針 |
| --- | ---: | --- | --- |
| 未対応query parameterまたはrequest body | 400 | `INVALID_REQUEST` | 要求を修正し、同じ内容では再試行しない |
| データが存在しない、読めない、空、不正 | 503 | `DATA_UNAVAILABLE` | デプロイ復旧後に成功する可能性がある |
| 予期しないアプリケーション障害 | 500 | `INTERNAL_ERROR` | 上限付きbackoffで再試行する |
| 未知のroute | 404 | `NOT_FOUND` | 同じ内容では再試行しない |
| `/api/random`の未対応method | 405 | `METHOD_NOT_ALLOWED` | `GET`を使用する |

すべてのJSONエラーは次の安定したenvelopeを使う。

```json
{
  "error": {
    "code": "DATA_UNAVAILABLE",
    "message": "Postal code data is temporarily unavailable.",
    "requestId": "01JEXAMPLE0000000000000000"
  }
}
```

messageは安全で機密情報を含まない英語とする。stack trace、生成物path、環境値、framework error、第三者サービス詳細を公開しない。同一request IDを`X-Request-Id`とJSON envelopeへ返し、構造化server logにも記録する。Hono既定のtextまたはHTML errorを公開API境界から返さない。

## 5. 抽選とデータ読込

```text
HTTP Controller
      |
      v
RandomPostalCodeService
      |
      v
PostalCodeRepository (port)
      |
      v
GeneratedDatasetRepository (infrastructure)
```

- Modelは`Address`、`PostalCode`、不変条件を定義し、HTTPやframeworkへ依存しない。
- `PostalCodeRepository`はfileやWorker binding型を漏らさず、正規化済みcollectionをApplicationへ提供する。
- `RandomPostalCodeService`は一意な郵便番号件数から1 indexを一様に選ぶ。
- `GeneratedDatasetRepository`はbuild時生成物を読込・検証し、Infrastructure障害を`DataUnavailable`へ変換する。
- Controllerはrequest受付、Service呼出、HTTP変換だけを行う。

正規化済みデータは可能な場合Worker isolateごとに1回読み込み、immutableとして扱う。初期化時に実行時network requestを行わない。初期化失敗は明示的なunavailable状態とし、空の成功payloadへfallbackしない。

ランダム抽選は`[0, count)`の偏りのない整数を使う。上限を含む値の丸めや元CSV行からの抽選を行わない。統計テストだけで乱数品質を証明せず、決定的な境界テストで先頭・末尾index mappingを検証し、必要なら分布チェックで明白な重み付け回帰を検出する。

## 6. CORSとHTTP Security

- 設定されたWeb Originだけをscheme、host、portを含めて完全一致で許可する。
- 完全一致後に要求Originを返し、`*`を返さない。
- Originにより応答が変わる場合は`Vary: Origin`を付ける。
- `GET`とbrowser clientに必要な最小headerだけを許可する。
- CORS middlewareが必要とする場合だけ`OPTIONS`を処理し、製品endpointとは扱わない。
- 未対応method、過長URL、過長headerをplatformまたはframework境界で拒否する。
- API responseへ`X-Content-Type-Options: nosniff`を付ける。
- client指定のforwarding headerやrequest IDをsecurity判断に使わない。
- 開発・本番allowlistを分離し、必須設定がない場合は安全側へ失敗する。

公開endpointはread-onlyでcredentialやcookieを使わないため、credential付きCORSを有効にせず、CSRF対象のmutationも持たない。将来、認証またはwrite操作を追加する場合は、新しい脅威レビューと契約を必須とする。

## 7. ObservabilityとPrivacy

構造化logにはtimestamp、生成request ID、route、method、status、duration、上限付きerror codeを含める。完全なresponse body、住所、環境secret、広告識別子、未信頼headerは記録しない。データ利用不能と予期しない障害を区別する。

MVPはmemory上の読取とランダム抽選だけを行うため、application-level rate limitを定義しない。Cloudflare platform制限は適用される。明示的なadmissionまたはrate制御は、濫用や容量の実測根拠を得て、statusとretry契約を文書化した後に追加する。

## 8. テスト契約

- 7桁と1件以上の住所を含む共有schemaの受理・拒否
- 住所の完全保持と不正生成物の拒否
- 選択可能な先頭・末尾indexと一意郵便番号単位の抽選
- 成功headerとpayload
- 内部詳細を漏らさない`400`、`404`、`405`、`500`、`503` mapping
- 許可Origin、拒否Origin、`Vary: Origin`、wildcard CORS不使用
- 日本郵便または他の住所APIへの実行時requestがないこと
- runtimeで決定的に観測できる場合、isolateごとにimmutable datasetを1回だけ読むこと
- PagesからWorkers、AndroidからWorkersへのrelease journey

HonoやWorker runtimeのテストも、APIやE2Eという名称ではなく実際の依存範囲でSmall、Medium、Largeへ分類する。

## 9. 未確定の実装詳細

- runtime schema libraryと生成物の正確な形式
- request ID generatorとCloudflare traceの関連付け
- platformの正確なURL・header長制限
- 実測に基づくapplication-level容量保護の必要性

これらを確定する際も、公開契約を弱めたり`packages/shared`へInfrastructure型を漏らしたりしない。
