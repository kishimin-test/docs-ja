# mojica API Controller設計書

## 1. 目的

画像生成APIのHTTP境界を定義する。

ControllerはHTTPリクエストを受け取り、Domain Modelを生成してServiceへ委譲し、Serviceの結果をHTTPレスポンスへ変換する。

## 2. ImageController

画像生成エンドポイントは `ImageController` が担当する。

### エンドポイント

```http
POST /images
```

### 依存方向

```text
HTTP Request
     │
     ▼
ImageController
     │
     ▼
ImageGenerationService
     │
     ▼
HTTP Response
```

Controllerは `ImageGenerationService` の契約を利用して画像生成を実行する。

## 3. HTTPリクエスト

### Headers

| ヘッダー          | 必須 | Controllerでの扱い                |
| ----------------- | :--: | --------------------------------- |
| `Content-Type`    |  ○   | `application/json` として解析する |
| `Accept-Language` |  -   | エラーメッセージの言語を決定する  |

`Accept-Language` では `ja` と `en` をサポートする。未指定または未対応の値の場合は `ja` を使用する。

### Body DTO

Controllerは次のHTTP DTOを受け取る。

| 属性                  | 型     | 必須 |
| --------------------- | ------ | :--: |
| `type`                | string |  ○   |
| `text`                | string |  ○   |
| `foregroundCharacter` | string |  ○   |
| `foregroundColor`     | string |  ○   |
| `backgroundCharacter` | string |  ○   |
| `backgroundColor`     | string |  ○   |

HTTP DTOはDomain Modelと同一の型として扱わない。Controllerまたは入力Mapperが各値をDomain Modelへ変換する。

## 4. リクエスト処理

Controllerは次の順序で処理する。

1. HTTPメソッドとパスを受け付ける
2. JSONをHTTP DTOへ解析する
3. `Accept-Language` から表示言語を決定する
4. HTTP DTOの各値を `ImageType`、`RenderText`、`PatternCharacter`、`HexColor` へ変換する
5. Value Objectの生成結果から `ImageGenerationRequest` を生成する
6. Domain検証に失敗した場合は `422 Unprocessable Entity` を返す
7. 検証済みの `ImageGenerationRequest` を `ImageGenerationService` へ渡す
8. Serviceの成功結果を画像レスポンスへ変換する
9. Serviceのエラー結果を公開APIのエラー契約へ変換する

Domain Modelを生成できない状態ではServiceを呼び出さない。複数の入力エラーを検出できる場合は、`errors` 配列へまとめて返す。

## 5. HTTPエラー

### 400 Bad Request

HTTPリクエストをJSONとして解釈できない場合に返す。

- JSON構文が不正
- BodyをJSONとして解析できない
- `Content-Type` またはリクエスト形式が期待と異なる

レスポンスは次の形式とする。

```json
{
  "code": "BAD_REQUEST",
  "message": "リクエストの形式が正しくありません。"
}
```

### 422 Unprocessable Entity

JSONとして解析できるが、Domain Modelを生成できない場合に返す。

レスポンスの全体コードは `VALIDATION_ERROR` とし、詳細を `errors` 配列へ格納する。

```json
{
  "code": "VALIDATION_ERROR",
  "message": "入力内容に誤りがあります。",
  "errors": [
    {
      "field": "foregroundColor",
      "message": "HEXカラー形式（#RRGGBB）で指定してください。"
    }
  ]
}
```

`code`、`field`、検証理由は言語に依存しない。`message` だけを `Accept-Language` に応じて切り替える。

## 6. Service結果の変換

ControllerはServiceから返された結果を次のHTTPレスポンスへ変換する。

| Service結果        | HTTPステータス          | 公開APIコード              |
| ------------------ | ----------------------- | -------------------------- |
| 成功               | `200 OK`                | なし。画像を返す           |
| `RATE_LIMITED`     | `429 Too Many Requests` | `RATE_LIMIT_EXCEEDED`      |
| `TIMEOUT`          | `504 Gateway Timeout`   | `IMAGE_GENERATION_TIMEOUT` |
| `UNAVAILABLE`      | `502 Bad Gateway`       | `IMAGE_GENERATION_FAILED`  |
| `INVALID_RESPONSE` | `502 Bad Gateway`       | `IMAGE_GENERATION_FAILED`  |
| `FAILED`           | `502 Bad Gateway`       | `IMAGE_GENERATION_FAILED`  |

`retryAfter` が結果に含まれる場合、Controllerは `Retry-After` ヘッダーへ変換する。

## 7. 成功レスポンス

画像生成に成功した場合は、次のレスポンスを返す。

| 項目                | 値                                        |
| ------------------- | ----------------------------------------- |
| Status              | `200 OK`                                  |
| Content-Type        | `image/png`                               |
| Body                | `GeneratedImage.content`                  |
| Content-Disposition | `attachment` と `GeneratedImage.fileName` |

`GeneratedImage.fileName` はServiceが生成した値を使用する。Controllerはファイル名を再生成したり、ユーザー入力値から組み立てたりしない。

## 8. エラーメッセージ

Controllerは公開API用の日本語・英語メッセージを解決する。

### 言語決定

| `Accept-Language` | 使用言語 |
| ----------------- | -------- |
| `ja`              | 日本語   |
| `en`              | 英語     |
| 未指定            | 日本語   |
| 未対応の値        | 日本語   |

エラーメッセージに次の情報を含めない。

- 例外メッセージ
- スタックトレース
- Glyph Forge APIのレスポンス本文
- 内部URL
- 認証情報
- SQLやInfrastructureの内部情報

## 9. 予期しない例外

ServiceまたはController内で予期しない例外が発生した場合は、詳細をログへ記録し、クライアントには次を返す。

```json
{
  "code": "INTERNAL_SERVER_ERROR",
  "message": "画像生成中に予期しないエラーが発生しました。"
}
```

HTTPステータスは `500 Internal Server Error` とする。内部例外の詳細はレスポンスへ含めない。

## 10. Controllerテスト契約

ControllerのテストはServiceを差し替え、HTTP境界から観測できる契約を検証する。

最低限、次の振る舞いを確認する。

- 正しいJSONを受け取り、ServiceへDomain Modelを渡す
- 不正なJSONに `400 Bad Request` を返す
- 必須値、文字数、制御文字、HEX形式のエラーに `422` を返す
- 複数の検証エラーを `errors` 配列へ格納する
- `Accept-Language: ja` で日本語メッセージを返す
- `Accept-Language: en` で英語メッセージを返す
- 言語未指定・未対応言語で日本語へフォールバックする
- Service成功時に `200 image/png` とファイル名を返す
- `RATE_LIMITED`、`TIMEOUT`、`UNAVAILABLE`、`INVALID_RESPONSE`、`FAILED` を正しいHTTP契約へ変換する
- `retryAfter` を `Retry-After` ヘッダーへ変換する
- 内部エラー詳細をレスポンスへ含めない

## 11. 決定事項

- 画像生成HTTPエンドポイントは `POST /images` とする
- ControllerはHTTP DTOとDomain Modelを分離する
- Controllerは検証済みModelだけをServiceへ渡す
- ControllerはService結果を公開APIのHTTP契約へ変換する
- エラーメッセージは `Accept-Language` で日本語・英語を切り替える
- 予期しない内部エラーの詳細をクライアントへ公開しない
