# mojica API Port設計書

## 1. 目的

画像生成ユースケースが外部の画像生成機能を利用するためのPort契約を定義する。

Portは、利用側が必要とする入力、成功結果、失敗結果を表現する。外部サービスの通信形式やHTTP形式はPort契約に含めない。

## 2. ImageGenerationPort

`ImageGenerationPort` は、検証済みの `ImageGenerationRequest` を受け取り、画像生成結果またはPortエラーを返す。

### 契約

```text
generate(
    request: ImageGenerationRequest
) -> Result<GeneratedImage, ImageGenerationPortError>
```

### 入力

入力はModelの不変条件を満たした `ImageGenerationRequest` とする。

Portは未検証の値やHTTP DTOを入力として受け取らない。

### 成功結果

成功時は `GeneratedImage` を返す。Portの成功結果に含まれる画像データは次のとおりとする。

- `content`：画像のバイナリデータ
- `mediaType`：画像のメディア形式

## 3. ImageGenerationPortError

`ImageGenerationPortError` は、画像生成機能の実行結果をPort契約の失敗として表現する。

### 属性

| 属性         | 内容                                       |
| ------------ | ------------------------------------------ |
| `code`       | 言語に依存しないエラーコード               |
| `retryAfter` | 再試行可能な秒数。取得できない場合は未設定 |
| `details`    | 外部へ公開してよい範囲に制限した補足情報   |

### エラーコード

| `code`             | 意味                                       |
| ------------------ | ------------------------------------------ |
| `RATE_LIMITED`     | 利用回数の制限により生成できない           |
| `TIMEOUT`          | 制限時間内に生成結果を取得できない         |
| `UNAVAILABLE`      | 画像生成機能を利用できない                 |
| `INVALID_RESPONSE` | 成功結果として解釈できない応答を受け取った |
| `FAILED`           | 画像生成に失敗した                         |

`ImageGenerationPortError` の公開属性に、認証情報、内部URL、スタックトレース、公開不要な通信詳細を含めない。

`retryAfter` は、再試行可能な時間を安全に判定できる場合だけ設定する。

## 4. Portのテスト契約

Portのテストは、Portから観測できる入力・成功・失敗の契約を検証する。

最低限、次の振る舞いを確認する。

- 有効な `ImageGenerationRequest` で `GeneratedImage` を返す
- 成功結果に画像データとメディア形式が含まれる
- 制限による失敗を `RATE_LIMITED` として返す
- 制限時間超過による失敗を `TIMEOUT` として返す
- 利用不能による失敗を `UNAVAILABLE` として返す
- 解釈できない応答による失敗を `INVALID_RESPONSE` として返す
- 生成失敗を `FAILED` として返す
- エラー結果に公開不要な通信詳細を含めない

## 5. 決定事項

- 画像生成のOutbound契約は `ImageGenerationPort` とする
- 成功結果は `GeneratedImage`、失敗結果は `ImageGenerationPortError` とする
- Portのエラーコードは `RATE_LIMITED`、`TIMEOUT`、`UNAVAILABLE`、`INVALID_RESPONSE`、`FAILED` とする
