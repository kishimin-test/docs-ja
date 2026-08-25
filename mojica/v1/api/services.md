# mojica API Service設計書

## 1. 目的

画像生成ユースケースの処理順序とオーケストレーションを定義する。

Serviceは、検証済みの画像生成リクエストを画像生成Portへ渡し、成功結果にダウンロード用のファイル名を付与して返す。

## 2. ImageGenerationService

画像生成ユースケースは `ImageGenerationService` が担当する。

### 契約

```text
generate(
    request: ImageGenerationRequest
) -> Result<GeneratedImage, ImageGenerationPortError>
```

入力はModelの不変条件を満たした `ImageGenerationRequest` とする。

## 3. 依存方向

```text
ImageGenerationService
          │
          ▼
ImageGenerationPort
```

Serviceは `ImageGenerationPort` の契約を利用して画像生成を実行する。

## 4. 処理フロー

Serviceは次の順序で処理する。

1. `ImageGenerationRequest` を受け取る
2. `ImageGenerationPort` を1回呼び出す
3. Portが成功した場合、画像データとメディア形式を受け取る
4. 画像種別とUUIDからダウンロード用ファイル名を生成する
5. `GeneratedImage` を完成させて返す
6. Portが失敗した場合、Portエラーを結果として返す

Serviceは、1回のユースケース実行でGlyph Forge APIへの自動再試行を行わない。

## 5. Port呼び出し

Serviceは、受け取った `ImageGenerationRequest` を変更せずに `ImageGenerationPort` へ渡す。

## 6. ファイル名生成

Serviceは、Portの成功結果にダウンロード用の一意なファイル名を付与する。

形式は次のとおりとする。

```text
mojica-{imageType}-{UUID}.png
```

例：

```text
mojica-x-icon-550e8400-e29b-41d4-a716-446655440000.png
```

### 生成規則

- `imageType` は正規化された `ImageType` の値を使用する
- UUIDは画像生成の成功結果ごとに新しく生成する
- ユーザー入力値をファイル名に含めない
- 拡張子は `.png` に固定する
- UUIDにより、画像生成の成功結果ごとに一意性を確保する

## 7. 成功結果

Portが成功した場合、Serviceは次の値を持つ `GeneratedImage` を返す。

| 属性        | 設定元                                            |
| ----------- | ------------------------------------------------- |
| `content`   | `ImageGenerationPort` の成功結果                  |
| `mediaType` | `ImageGenerationPort` の成功結果                  |
| `fileName`  | Serviceが生成した `mojica-{imageType}-{UUID}.png` |

生成結果は呼び出し元へ返して処理を終了する。

## 8. エラー結果

Portが返した `ImageGenerationPortError` は、Serviceの結果としてそのまま返す。

Serviceは `ImageGenerationPortError` の分類を壊さず、呼び出し元がエラーコードと `retryAfter` を利用できる状態で返す。

## 9. 再試行と冪等性

Serviceは画像生成処理を自動再試行しない。

画像生成は外部API上で副作用を持つため、タイムアウト後に再実行すると複数画像が生成される可能性がある。再試行が必要になった場合は、外部APIの冪等性キー契約を追加で定義してから変更する。

## 10. テスト契約

Serviceのテストは `ImageGenerationPort` を差し替えて、Serviceから観測できる振る舞いを検証する。

最低限、次の振る舞いを確認する。

- 検証済みの `ImageGenerationRequest` をPortへ渡す
- 1回の実行でPortを1回だけ呼び出す
- Portの成功結果に正しい `content` と `mediaType` を保持する
- `ImageType` とUUIDを使ったファイル名を生成する
- `standard`、`x-background`、`x-icon` のファイル名を正しく生成する
- ユーザー入力値をファイル名に含めない
- Portの `RATE_LIMITED` を変更せず返す
- Portの `TIMEOUT` を変更せず返す
- Portの `UNAVAILABLE`、`INVALID_RESPONSE`、`FAILED` を変更せず返す
- Portが失敗した場合にファイル名を生成しない
- Portの失敗後に自動再試行しない

## 11. 決定事項

- 画像生成ユースケースは `ImageGenerationService` がオーケストレーションする
- Serviceは `ImageGenerationPort` だけを通じて画像生成を実行する
- Serviceは成功結果に一意なファイル名を付与する
- Serviceは画像生成を自動再試行しない
