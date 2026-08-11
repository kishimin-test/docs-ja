# mojica API Adapter設計書

## 1. 目的

`ImageGenerationPort` の契約を、Glyph Forge APIとのHTTP通信へ変換するAdapterの責務と境界を定義する。

AdapterはInfrastructure層に配置し、Glyph Forge API、HTTPクライアント、JSONライブラリ、ASP.NET Coreなどの外部技術への依存を閉じ込める。

## 2. 対象範囲

この文書では、次を定義する。

- `GlyphForgeImageGenerationAdapter` の責務
- `ImageGenerationPort` から外部APIへの変換境界
- `ImageType` に応じた外部エンドポイントの選択
- `HexColor` からRGB値への変換
- 外部APIの応答検証とDomain結果への変換
- 外部APIエラーの `ImageGenerationPortError` への変換
- Adapterのテスト契約

## 3. 依存方向

```text
ImageGenerationPort
    ▲
    │ implements
    │
GlyphForgeImageGenerationAdapter
    │
    ├── HTTP Client
    ├── Glyph Forge Request/Response DTO
    └── Glyph Forge API
```

`GlyphForgeImageGenerationAdapter` は `ImageGenerationPort` を実装し、Glyph Forge APIのURL、HTTPクライアント、外部DTOへの依存をAdapter内部に閉じ込める。

## 4. Adapterの責務

### 担当すること

- 検証済みの `ImageGenerationRequest` を受け取る
- `ImageType` をGlyph Forge APIのエンドポイントへ変換する
- `HexColor` の値をRGB値へ変換する
- Glyph Forge API固有のリクエストDTOを生成する
- HTTPリクエストを送信する
- タイムアウトとキャンセルを処理する
- HTTPステータス、ヘッダー、Content-Type、画像データを検証する
- Glyph Forge API固有のレスポンスDTOまたはバイナリを `GeneratedImage` へ変換する
- 外部API固有の失敗を `ImageGenerationPortError` へ変換する

## 5. Port実装契約

Adapterは次のPort契約を実装する。

```text
generate(
    request: ImageGenerationRequest
) -> Result<GeneratedImage, ImageGenerationPortError>
```

入力はPort契約を満たした `ImageGenerationRequest` に限定する。Adapterは未検証の文字列やHTTPリクエストを受け取らない。

成功時は画像バイナリとメディア形式を含む `GeneratedImage` を返す。ダウンロード用の `fileName` は `models.md` の契約に従う。

失敗時は、HTTPクライアントの例外、JSONデシリアライズ例外、外部API固有のエラー型を返さず、`ImageGenerationPortError` を返す。

## 6. リクエスト変換

Adapterは次の境界でDomain Modelを外部API DTOへ変換する。

```text
ImageGenerationRequest
        │
        ├── ImageType ───────────▶ Glyph Forge endpoint
        ├── RenderText ──────────▶ Glyph Forge text field
        ├── PatternCharacter ────▶ Glyph Forge character field
        ├── HexColor ────────────▶ RgbColor ──▶ Glyph Forge color DTO
        └── PatternCharacter ────▶ Glyph Forge character field
```

変換処理は外部APIのDTO構造に依存する。Glyph Forge APIの具体的なJSONフィールド名、認証方式、追加ヘッダー、レスポンスDTOの詳細は外部API契約を確認してから実装する。既存のmojica設計書にない外部仕様を推測して実装しない。

## 7. エンドポイント選択

`ImageType` とGlyph Forge APIのエンドポイントの対応は次のとおりとする。

| `ImageType` | Method | Path |
| --- | --- | --- |
| `standard` | `POST` | `/images` |
| `x-background` | `POST` | `/images/background` |
| `x-icon` | `POST` | `/images/x-icon` |

この対応表はAdapterのInfrastructure実装に閉じ込める。

未定義の `ImageType` はエンドポイント選択処理へ到達させない。防御的に未定義値を検出した場合は `FAILED` として扱い、外部APIへリクエストしない。

## 8. HEXからRGBへの変換

Adapterは外部APIへ送信する前に、`HexColor` からRGB値を取得する。

```text
HexColor("#FF69B4")
        │
        ▼
RgbColor(red: 255, green: 105, blue: 180)
        │
        ▼
Glyph Forge API-specific color DTO
```

`#RRGGBB` 形式の検証、16進数の解釈、各成分の範囲検証は `HexColor` と `RgbColor` が担当する。Adapterは、取得したRGB値をGlyph Forge API固有DTOへ詰め替えるだけとする。

## 9. HTTPリクエスト

Adapterは次の要件を満たすHTTPリクエストを送信する。

- `POST` メソッドを使用する
- `ImageType` に対応したパスを使用する
- Glyph Forge APIの契約で定義されたContent-Typeを使用する
- 必要な認証情報や設定値は安全な設定境界から取得する
- キャンセルトークンをHTTPリクエストへ伝播する
- 設定されたタイムアウトを適用する
- リクエスト本文へHTTP入力DTOや未検証の値を直接渡さない

Glyph Forge APIの認証方式、必須ヘッダー、具体的なリクエストDTOは、外部API契約で確定した値だけを使用する。秘密情報をソースコードやログへ記録しない。

## 10. HTTPレスポンス

Adapterは、外部APIのレスポンスを次の順序で処理する。

1. HTTPステータスを確認する
2. `429 Too Many Requests` の場合は `RATE_LIMITED` に変換する
3. `Retry-After` を安全に解釈できる場合は `retryAfter` に設定する
4. タイムアウトの場合は `TIMEOUT` に変換する
5. その他の通信失敗や利用不能の場合は `UNAVAILABLE` に変換する
6. 成功レスポンスのContent-Typeを確認する
7. 画像バイナリが存在し、読み取り可能であることを確認する
8. 画像データとメディア形式を `GeneratedImage` へ変換する
9. 期待する画像として解釈できない場合は `INVALID_RESPONSE` に変換する

Glyph Forge APIが画像生成失敗を示す場合は `FAILED` に変換する。外部APIのエラー本文、スタックトレース、内部URL、認証情報は上位層へ渡さない。

## 11. エラー変換

Adapterは外部APIの失敗を `ImageGenerationPortError` へ変換する。

| 外部で発生した事象 | Portエラー |
| --- | --- |
| レート制限 | `RATE_LIMITED` |
| タイムアウト | `TIMEOUT` |
| DNS、接続、TLS、HTTPクライアントの通信失敗 | `UNAVAILABLE` |
| 画像として解釈できない応答 | `INVALID_RESPONSE` |
| Glyph Forge APIの生成失敗 | `FAILED` |

## 12. タイムアウトとキャンセル

Adapterは設定されたタイムアウトを超えて外部APIを待ち続けない。

呼び出し元からキャンセルトークンが渡された場合は、HTTPクライアントへ伝播する。キャンセルとタイムアウトを呼び出し元が分類できるよう、内部の例外を `TIMEOUT` または適切なPortエラーへ変換する。

タイムアウト後の自動再試行は行わない。画像生成は再実行による重複生成の可能性があるため、再試行方針はGlyph Forge APIの冪等性契約を確認してから別途決定する。

## 13. 設定と秘密情報

Adapterが使用する環境依存値は設定境界から注入する。

- Glyph Forge APIのBase URL
- 接続・応答タイムアウト
- 認証情報またはAPIキー
- 必要なサービス固有ヘッダー

AdapterにBase URLや秘密情報をハードコードしない。秘密情報を例外、ログ、Portエラー、公開APIレスポンスへ含めない。

## 14. テスト契約

### Smallテスト

HTTPクライアントを差し替え、Adapterから観測できる変換結果を検証する。

- `standard` を `/images` へ変換する
- `x-background` を `/images/background` へ変換する
- `x-icon` を `/images/x-icon` へ変換する
- `#FF69B4` をRGBの255、105、180へ変換する
- 成功レスポンスを `GeneratedImage` へ変換する
- 429を `RATE_LIMITED` へ変換する
- タイムアウトを `TIMEOUT` へ変換する
- 通信失敗を `UNAVAILABLE` へ変換する
- 不正な画像レスポンスを `INVALID_RESPONSE` へ変換する
- 外部APIの内部詳細をエラー結果へ含めない

### Mediumテスト

利用可能なテスト用Glyph Forge APIまたはHTTPスタブを使用し、Adapterから外部API境界までの契約を検証する。

- 実際のContent-Typeと画像バイナリを処理できる
- `Retry-After` を安全に解釈できる
- タイムアウトとキャンセルが外部通信へ伝播する
- 外部APIの各エラー応答がPortエラーへ変換される

Glyph Forge APIの実通信を行うテストでは、秘密情報をリポジトリへ保存せず、テスト間で認証状態やポートを共有しない。

## 15. 外部API契約の確認事項

### 既存の設計書から確定している契約

| 項目 | 決定内容 | 根拠 |
| --- | --- | --- |
| 画像種別 | `standard`、`x-background`、`x-icon` | `models.md`、`mvp-api.md` |
| エンドポイント | `POST /images`、`POST /images/background`、`POST /images/x-icon` | `mvp-api.md` |
| 入力値 | 描画文字列、前景・背景文字、前景・背景色 | `models.md`、`mvp-api.md` |
| 色の変換 | HEXをRGBへ変換して外部APIへ送信 | `models.md`、`mvp-api.md` |
| 成功結果 | PNG画像を取得し、`GeneratedImage` へ変換 | `mvp-api.md`、`ports.md` |
| レート制限 | 外部APIの429を `RATE_LIMITED` へ変換 | `mvp-api.md`、`ports.md` |
| タイムアウト | 外部APIのタイムアウトを `TIMEOUT` へ変換 | `mvp-api.md`、`ports.md` |
| その他の失敗 | 通信失敗・不正応答・生成失敗をPortエラーへ変換 | `ports.md` |

### Adapter契約として確定する境界

- 外部APIのURL、HTTPクライアント、JSON DTOはAdapter内部に閉じ込める
- Adapterは検証済みの `ImageGenerationRequest` だけを受け取る
- Adapterは外部APIの結果を `GeneratedImage` または `ImageGenerationPortError` へ変換する
- 外部APIのエラー本文、例外、スタックトレース、内部URL、認証情報を上位層へ漏出させない

### 外部API仕様として未確定の項目

Glyph Forge APIの仕様書が既存リポジトリにないため、次の項目は外部API提供者の契約確認後に固定する。

- 各エンドポイントのリクエストJSONフィールド名と型
- RGBカラーDTOの正確な形状
- 認証方式と必須ヘッダー
- 成功時の正確なContent-Typeとレスポンス形式
- 失敗時のステータスコードとエラー形式
- `Retry-After` の単位と値の形式
- タイムアウト値
- タイムアウト時の再実行可否と冪等性

未確定項目がある間は、Adapter設計書やコードへ外部API DTOの具体的な型・フィールド名・認証方式を推測して固定しない。

## 16. 決定事項

- AdapterはInfrastructure層に配置する
- `GlyphForgeImageGenerationAdapter` は `ImageGenerationPort` を実装する
- 外部APIのURL、HTTPクライアント、DTO、認証方式はAdapterの外側へ漏出させない
- Domain Modelの検証をAdapterで重複実装しない
- 外部APIの失敗は `ImageGenerationPortError` へ変換する
- 外部API固有DTOの詳細はGlyph Forge API契約が確定するまで固定しない
