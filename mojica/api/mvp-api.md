# mojica MVP API設計書

## 1. 概要

mojicaのバックエンドはASP.NET Coreで実装する。

フロントエンドから受け取った入力値をもとに、画像生成API「Glyph Forge API」へリクエストを送信し、生成された画像をクライアントへ返却する。

フロントエンドから呼び出す画像生成APIは1つとし、画像の種類に応じたGlyph Forge APIのエンドポイントの振り分けはバックエンドが担当する。

HEX形式で色を受け取り、RGB形式へ変換してGlyph Forge APIへ送信する。

mojica APIはi18n（国際化）に対応し、クライアントへ返却するエラーメッセージを日本語・英語で切り替えられる構成とする。

---

# 2. システム構成

```text
Frontend
    │
    │ POST /images
    │ Color: HEX
    │ Accept-Language: ja / en
    ▼
ASP.NET Core API
    │
    ├── 言語判定
    ├── リクエストバリデーション
    ├── レート制限
    ├── HEX → RGB変換
    └── typeによる振り分け
            │
            ▼
      Glyph Forge API
        ├── POST /images
        ├── POST /images/background
        └── POST /images/x-icon
            │
            ▼
          PNG画像
            │
            ▼
       ASP.NET Core API
            │
            ▼
         Frontend
```

---

# 3. API一覧

| メソッド | エンドポイント | 概要                     |
| -------- | -------------- | ------------------------ |
| POST     | `/images`      | 文字アート画像を生成する |

---

# 4. 画像生成API

## エンドポイント

```http
POST /images
```

## 概要

指定された内容から文字アート画像を生成する。

`type` に応じてASP.NET Coreが適切なGlyph Forge APIのエンドポイントへリクエストを送信する。

生成に成功した場合、生成されたPNG画像をレスポンスとして返却する。

---

# 5. リクエスト

## Headers

| ヘッダー          | 必須 | 説明                             |
| ----------------- | :--: | -------------------------------- |
| `Content-Type`    |  ○   | `application/json`               |
| `Accept-Language` |  -   | エラーメッセージの言語を指定する |

`Accept-Language` では以下の値をサポートする。

| 値   | 言語   |
| ---- | ------ |
| `ja` | 日本語 |
| `en` | 英語   |

日本語：

```http
Accept-Language: ja
```

英語：

```http
Accept-Language: en
```

`Accept-Language` が指定されていない場合は日本語を使用する。

サポートしていない言語が指定された場合も日本語へフォールバックする。

---

## Body

| 項目                  | 型     | 必須 | 説明                    |
| --------------------- | ------ | :--: | ----------------------- |
| `type`                | enum   |  ○   | 出力する画像の種類      |
| `text`                | string |  ○   | 描画する文字列          |
| `foregroundCharacter` | string |  ○   | 描画に使用する文字      |
| `foregroundColor`     | string |  ○   | 描画文字色（HEX）       |
| `backgroundCharacter` | string |  ○   | 敷き詰める文字          |
| `backgroundColor`     | string |  ○   | 敷き詰める文字色（HEX） |

## type

| 値             | 説明          |
| -------------- | ------------- |
| `standard`     | 標準画像      |
| `x-background` | X背景画像     |
| `x-icon`       | Xアイコン画像 |

## リクエスト例

```http
POST /images
Content-Type: application/json
Accept-Language: ja
```

```json
{
  "type": "x-icon",
  "text": "KA",
  "foregroundCharacter": "🌻",
  "foregroundColor": "#FFD400",
  "backgroundCharacter": "☀",
  "backgroundColor": "#FF69B4"
}
```

---

# 6. リクエストバリデーション

mojica APIは、Glyph Forge APIを呼び出す前にすべての画像生成リクエストを検証する。

フロントエンドでバリデーション済みの場合でも、バックエンド側で必ずバリデーションを行う。

バリデーションに失敗した場合、Glyph Forge APIは呼び出さず `422 Unprocessable Entity` を返却する。

## `text`

描画する文字列。

制約：

- 必須
- 1文字以上
- 64文字以下
- 空白文字のみは禁止
- 制御文字は禁止

---

## `foregroundCharacter`

描画に使用する文字。

制約：

- 必須
- 1文字以上
- 128文字以下
- 制御文字は禁止
- 空白文字のみは許可する

---

## `foregroundColor`

描画に使用する文字の色。

制約：

- 必須
- `#RRGGBB` 形式の有効なHEXカラーであること

例：

```text
#FFD400
```

バリデーション成功後、ASP.NET CoreでHEXからRGBへ変換してGlyph Forge APIへ送信する。

---

## `backgroundCharacter`

描画文字の周囲に敷き詰める文字。

制約：

- 必須
- 1文字以上
- 128文字以下
- 制御文字は禁止
- 空白文字のみは許可する

---

## `backgroundColor`

敷き詰める文字の色。

制約：

- 必須
- `#RRGGBB` 形式の有効なHEXカラーであること

例：

```text
#FF69B4
```

バリデーション成功後、ASP.NET CoreでHEXからRGBへ変換してGlyph Forge APIへ送信する。

---

## `type`

生成する画像の種類。

制約：

- 必須
- 以下のいずれかであること
  - `standard`
  - `x-background`
  - `x-icon`

---

## 文字の組み合わせ

`foregroundCharacter` と `backgroundCharacter` は、それぞれ単独では空白文字のみを指定できる。

ただし、両方を空白文字のみにすることは禁止する。

少なくともどちらか一方には表示可能な文字を含める必要がある。

---

# 7. Glyph Forge APIとの連携

## エンドポイント振り分け

ASP.NET Coreは `type` の値に応じて、呼び出すGlyph Forge APIのエンドポイントを切り替える。

| type           | Glyph Forge API           |
| -------------- | ------------------------- |
| `standard`     | `POST /images`            |
| `x-background` | `POST /images/background` |
| `x-icon`       | `POST /images/x-icon`     |

この振り分けをバックエンドで行うことで、フロントエンドはGlyph Forge APIのエンドポイント構成を意識しない。

---

# 8. 色情報の変換

フロントエンドではカラーピッカーを使用し、色をHEX形式で取得する。

例えば、以下の値をmojica APIへ送信する。

```text
#FF69B4
```

mojica APIは受け取ったHEXカラーをASP.NET Core側でRGBへ変換する。

```text
#FF69B4

↓

R: 255
G: 105
B: 180
```

変換したRGB値をGlyph Forge APIへ送信する。

この変換をバックエンドで行うことで、フロントエンドはGlyph Forge API固有の色表現を意識する必要がない。

---

# 9. i18n（国際化）

mojica APIは、クライアントへ返却するエラーメッセージの多言語化に対応する。

MVPでは日本語と英語をサポートする。

## 対応言語

| 言語コード | 言語   |
| ---------- | ------ |
| `ja`       | 日本語 |
| `en`       | 英語   |

## 言語指定

クライアントはHTTPの `Accept-Language` ヘッダーを使用して言語を指定する。

```http
Accept-Language: ja
```

または、

```http
Accept-Language: en
```

`Accept-Language` が指定されていない場合は日本語を使用する。

サポートしていない言語が指定された場合も日本語へフォールバックする。

## エラーコードとメッセージ

`code` および `field` は言語に依存しない固定値とする。

`message` および `errors[].message` のみ、指定された言語に応じて切り替える。

### 日本語

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

### 英語

```json
{
  "code": "VALIDATION_ERROR",
  "message": "The input contains validation errors.",
  "errors": [
    {
      "field": "foregroundColor",
      "message": "The value must be specified in HEX color format (#RRGGBB)."
    }
  ]
}
```

フロントエンドは表示言語に依存せず、`code` および `field` を使用してエラーを判定できる。

---

# 10. 正常レスポンス

## 200 OK

画像生成に成功した場合に返却する。

本APIでは生成した画像をサーバー上の永続的なリソースとして作成せず、その場で生成したPNG画像をレスポンスとして返却するため、`201 Created` ではなく `200 OK` とする。

| 項目             | 内容            |
| ---------------- | --------------- |
| ステータスコード | `200 OK`        |
| Content-Type     | `image/png`     |
| レスポンス       | 生成したPNG画像 |

ダウンロード用のファイル名は `Content-Disposition` の `filename` に設定する。ファイル名はリクエストごとに一意とし、形式は `mojica-{imageType}-{UUID}.png` とする。例：`mojica-x-icon-550e8400-e29b-41d4-a716-446655440000.png`。ユーザー入力値はファイル名に含めない。

ASP.NET CoreはGlyph Forge APIから取得したPNG画像をクライアントへ返却する。

---

# 11. エラーレスポンス

エラーレスポンスはJSON形式で返却する。

エラーメッセージは `Accept-Language` に応じて日本語または英語で返却する。

`code` および `field` は言語によらず固定値とし、`message` および `errors[].message` のみローカライズする。

---

## 400 Bad Request

HTTPリクエストとして正しく解釈できない場合に返却する。

主なケース：

- JSONの構文が不正
- リクエストボディをJSONとして解析できない
- APIが期待するリクエスト形式ではない

### 日本語

```json
{
  "code": "BAD_REQUEST",
  "message": "リクエストの形式が正しくありません。"
}
```

### 英語

```json
{
  "code": "BAD_REQUEST",
  "message": "The request format is invalid."
}
```

---

## 422 Unprocessable Entity

リクエスト自体は解析できるが、入力値がAPIの要件を満たしていない場合に返却する。

主なケース：

- 必須項目が未入力
- `type` に定義されていない値が指定されている
- カラーがHEX形式ではない
- 文字数の上限を超えている
- 制御文字が含まれている
- `text` が空白文字のみ
- `foregroundCharacter` と `backgroundCharacter` の両方が空白文字のみ

可能な限り、検出したすべてのバリデーションエラーを `errors` 配列に格納する。

### 必須項目エラー

日本語：

```json
{
  "code": "VALIDATION_ERROR",
  "message": "入力内容に誤りがあります。",
  "errors": [
    {
      "field": "text",
      "message": "描画する文字列は必須です。"
    }
  ]
}
```

英語：

```json
{
  "code": "VALIDATION_ERROR",
  "message": "The input contains validation errors.",
  "errors": [
    {
      "field": "text",
      "message": "The text field is required."
    }
  ]
}
```

### 文字数エラー

日本語：

```json
{
  "code": "VALIDATION_ERROR",
  "message": "入力内容に誤りがあります。",
  "errors": [
    {
      "field": "text",
      "message": "描画する文字列は64文字以内で入力してください。"
    }
  ]
}
```

英語：

```json
{
  "code": "VALIDATION_ERROR",
  "message": "The input contains validation errors.",
  "errors": [
    {
      "field": "text",
      "message": "The text must be 64 characters or fewer."
    }
  ]
}
```

### カラー形式エラー

日本語：

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

英語：

```json
{
  "code": "VALIDATION_ERROR",
  "message": "The input contains validation errors.",
  "errors": [
    {
      "field": "foregroundColor",
      "message": "The value must be specified in HEX color format (#RRGGBB)."
    }
  ]
}
```

### typeエラー

日本語：

```json
{
  "code": "VALIDATION_ERROR",
  "message": "入力内容に誤りがあります。",
  "errors": [
    {
      "field": "type",
      "message": "standard、x-background、x-iconのいずれかを指定してください。"
    }
  ]
}
```

英語：

```json
{
  "code": "VALIDATION_ERROR",
  "message": "The input contains validation errors.",
  "errors": [
    {
      "field": "type",
      "message": "The value must be one of: standard, x-background, or x-icon."
    }
  ]
}
```

### 文字の組み合わせエラー

`foregroundCharacter` と `backgroundCharacter` の両方が空白文字のみの場合は、それぞれのフィールドにエラーを設定する。

日本語：

```json
{
  "code": "VALIDATION_ERROR",
  "message": "入力内容に誤りがあります。",
  "errors": [
    {
      "field": "foregroundCharacter",
      "message": "描画に使う文字または敷き詰める文字のどちらかに、表示可能な文字を入力してください。"
    },
    {
      "field": "backgroundCharacter",
      "message": "描画に使う文字または敷き詰める文字のどちらかに、表示可能な文字を入力してください。"
    }
  ]
}
```

英語：

```json
{
  "code": "VALIDATION_ERROR",
  "message": "The input contains validation errors.",
  "errors": [
    {
      "field": "foregroundCharacter",
      "message": "Either the foreground or background characters must contain at least one visible character."
    },
    {
      "field": "backgroundCharacter",
      "message": "Either the foreground or background characters must contain at least one visible character."
    }
  ]
}
```

---

## 429 Too Many Requests

以下のいずれかの場合に返却する。

- mojica APIのレート制限を超過した場合
- Glyph Forge APIのレート制限を超過した場合

画像生成は比較的負荷の高い処理であるため、mojica API側でもレート制限を設け、過剰な画像生成リクエストからmojica APIおよびGlyph Forge APIを保護する。

mojica API自身のレート制限を超過した場合は、Glyph Forge APIを呼び出さず `429 Too Many Requests` を返却する。

Glyph Forge APIから `429 Too Many Requests` が返却された場合も、mojica APIはクライアントへ `429 Too Many Requests` を返却する。

### 日本語

```json
{
  "code": "RATE_LIMIT_EXCEEDED",
  "message": "リクエスト回数の上限に達しました。時間をおいて再度お試しください。"
}
```

### 英語

```json
{
  "code": "RATE_LIMIT_EXCEEDED",
  "message": "The request limit has been exceeded. Please try again later."
}
```

再試行可能なタイミングが分かる場合は、`Retry-After` ヘッダーを返却する。

Glyph Forge APIから `Retry-After` が返却された場合は、その値を考慮してmojica APIのレスポンスにも設定する。

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
Content-Type: application/json
```

---

## 500 Internal Server Error

ASP.NET Core内部で予期しないエラーが発生した場合に返却する。

### 日本語

```json
{
  "code": "INTERNAL_SERVER_ERROR",
  "message": "画像生成中に予期しないエラーが発生しました。"
}
```

### 英語

```json
{
  "code": "INTERNAL_SERVER_ERROR",
  "message": "An unexpected error occurred while generating the image."
}
```

内部の例外メッセージ、スタックトレースなどの情報はレスポンスに含めない。

---

## 502 Bad Gateway

Glyph Forge APIへの通信またはGlyph Forge API側で問題が発生し、画像を正常に取得できなかった場合に返却する。

主なケース：

- Glyph Forge APIがエラーを返した
- Glyph Forge APIから期待するレスポンスを取得できなかった
- Glyph Forge APIとの通信に失敗した

Glyph Forge APIの `429 Too Many Requests` およびタイムアウトについては、それぞれmojica APIの `429 Too Many Requests`、`504 Gateway Timeout` として扱う。

### 日本語

```json
{
  "code": "IMAGE_GENERATION_FAILED",
  "message": "画像の生成に失敗しました。時間をおいて再度お試しください。"
}
```

### 英語

```json
{
  "code": "IMAGE_GENERATION_FAILED",
  "message": "Image generation failed. Please try again later."
}
```

Glyph Forge APIの内部エラー内容は、そのままクライアントへ公開しない。

---

## 504 Gateway Timeout

Glyph Forge APIから一定時間以内にレスポンスを取得できなかった場合に返却する。

### 日本語

```json
{
  "code": "IMAGE_GENERATION_TIMEOUT",
  "message": "画像の生成に時間がかかっています。時間をおいて再度お試しください。"
}
```

### 英語

```json
{
  "code": "IMAGE_GENERATION_TIMEOUT",
  "message": "Image generation is taking too long. Please try again later."
}
```

---

# 12. HTTPステータスコード一覧

| ステータス                  | 用途                                              |
| --------------------------- | ------------------------------------------------- |
| `200 OK`                    | 画像生成成功                                      |
| `400 Bad Request`           | リクエスト形式が不正                              |
| `422 Unprocessable Entity`  | 入力値のバリデーションエラー                      |
| `429 Too Many Requests`     | mojica APIまたはGlyph Forge APIのレート制限を超過 |
| `500 Internal Server Error` | mojica API内部の予期しないエラー                  |
| `502 Bad Gateway`           | Glyph Forge APIの呼び出し・画像生成に失敗         |
| `504 Gateway Timeout`       | Glyph Forge APIがタイムアウト                     |

---

# 13. レート制限

画像生成処理への過剰なリクエストを防止するため、ASP.NET Core側でレート制限を行う。

また、依存先であるGlyph Forge APIにもレート制限が存在する。

そのため、以下の2種類のレート制限を考慮する。

- mojica API自身のレート制限
- Glyph Forge APIのレート制限

mojica API自身のレート制限を超過した場合は、Glyph Forge APIを呼び出さず `429 Too Many Requests` を返却する。

Glyph Forge APIから `429 Too Many Requests` が返却された場合は、mojica APIでも `429 Too Many Requests` としてクライアントへ返却する。

Glyph Forge APIから `Retry-After` が返却された場合は、その値を考慮してクライアントへ返却する。

mojica API側のレート制限は、原則としてGlyph Forge API側のレート制限に到達する前にリクエストを制限できる値を設定する。

具体的なリクエスト回数および制限時間については、Glyph Forge APIのレート制限と実際の運用環境を考慮して決定する。

---

# 14. リクエスト処理フロー

画像生成リクエストは以下の順序で処理する。

1. HTTPリクエストを解析する
2. `Accept-Language` から言語を判定する
3. リクエスト項目をバリデーションする
4. バリデーションエラーがある場合は `422 Unprocessable Entity` を返却する
5. mojica APIのレート制限を確認する
6. HEXカラーをRGBへ変換する
7. `type` に応じてGlyph Forge APIのエンドポイントを決定する
8. Glyph Forge APIへリクエストする
9. Glyph Forge APIのレスポンスを処理する
10. 生成されたPNG画像をクライアントへ返却する

mojica APIのバリデーションまたはレート制限によりリクエストを拒否する場合、Glyph Forge APIは呼び出さない。

---

# 15. MVPにおけるバックエンドの責務

ASP.NET Coreバックエンドは以下を担当する。

- フロントエンドからの画像生成リクエスト受付
- HTTPリクエストの解析
- リクエストのバリデーション
- Glyph Forge APIの入力制約と整合したバリデーション
- `Accept-Language` に基づく日本語・英語の言語判定
- 日本語・英語のエラーメッセージのローカライズ
- `Accept-Language` が未指定の場合の日本語へのフォールバック
- 未対応言語が指定された場合の日本語へのフォールバック
- エラーコードおよびフィールド名を言語に依存しない固定値として管理
- HEXカラーからRGBカラーへの変換
- `type` に応じたGlyph Forge APIエンドポイントの選択
- Glyph Forge APIへのリクエスト
- Glyph Forge APIから取得したPNG画像の返却
- Glyph Forge APIのエラーの適切な変換
- Glyph Forge APIのレート制限のハンドリング
- タイムアウト処理
- mojica API自身のレート制限
- 内部エラー情報をクライアントへ公開しないためのエラーハンドリング
