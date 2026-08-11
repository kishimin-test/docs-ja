# mojica API Port設計書

## 1. 目的

画像生成ユースケースからGlyph Forge APIへの通信詳細を分離するためのOutbound Portと、そのInfrastructure実装の契約を定義する。

この文書でいうPortは、内側の層が外部システムへ依存するために定義するインターフェースである。HTTPクライアント、JSON DTO、Glyph Forge APIのステータスコード、ASP.NET Coreの型はPortの契約に含めない。

## 2. 対象範囲

この文書では、次を定義する。

- `ImageGenerationPort` の入力、成功結果、失敗結果
- `GlyphForgeImageGenerationAdapter` の責務
- `ImageType` からGlyph Forge APIエンドポイントへの振り分け
- `HexColor` から `RgbColor` への変換境界
- 外部APIエラーのDomain/Applicationエラーへの変換
- PortとAdapterのテスト契約

次はこの文書の対象外とする。

- HTTPリクエストの解析とレスポンスの生成
- `Accept-Language` の解釈とメッセージのローカライズ
- Glyph Forge APIのHTTP DTOそのもの
- データベースへの保存・取得
- レート制限の判定ロジック

## 3. 依存方向

```text
Controller
    │
    ▼
 Service ───────────────▶ Model
    │
    ▼
ImageGenerationPort
    ▲
    │ implements
    │
GlyphForgeImageGenerationAdapter
    │
    ▼
Glyph Forge API
```

`ImageGenerationPort` はServiceが利用する内側の契約として定義する。`GlyphForgeImageGenerationAdapter` はInfrastructure側に配置し、Portを実装する。

Model、Service、PortはGlyph Forge API、HTTPクライアント、JSONライブラリ、ASP.NET Coreを参照しない。Adapterだけが外部APIおよび通信ライブラリを参照する。

## 4. ImageGenerationPort

`ImageGenerationPort` は、検証済みの `ImageGenerationRequest` を受け取り、画像生成結果を返す。

### 契約

```text
generate(
    request: ImageGenerationRequest
) -> Result<GeneratedImage, ImageGenerationPortError>
```

### 入力

入力は、Controllerまたは入力Mapperで生成され、Modelの不変条件を満たした `ImageGenerationRequest` とする。

Portは未検証のHTTP DTO、JSONオブジェクト、自由な文字列の組み合わせを受け取らない。入力の必須値、文字数、色形式、画像種類の検証はPortの呼び出し前に完了している必要がある。

### 成功結果

成功時は `GeneratedImage` を返す。

- `content`：画像のバイナリデータ
- `mediaType`：画像のメディア形式
- `fileName`：Serviceが生成した一意なダウンロード用ファイル名

`fileName` の形式は `models.md` の定義に従い、Serviceが生成する。Glyph Forge APIのレスポンスや外部APIのファイル名をそのまま公開しない。

### 失敗結果

外部API呼び出しで想定される失敗は、`ImageGenerationPortError` として返す。HTTPクライアントやGlyph Forge SDKの例外をServiceへそのまま漏出させない。

## 5. ImageGenerationPortError

`ImageGenerationPortError` は、Serviceが外部API呼び出しの結果を判断するためのApplication境界エラーである。

### 属性

| 属性 | 内容 |
| --- | --- |
| `code` | 言語に依存しないエラーコード |
| `retryAfter` | 再試行可能な秒数。取得できない場合は未設定 |
| `details` | 外部へ公開してよい範囲に制限した補足情報 |

### エラーコード

| `code` | 発生条件 | Controllerでの変換先 |
| --- | --- | --- |
| `RATE_LIMITED` | Glyph Forge APIがレート制限を返した | `429 Too Many Requests` |
| `TIMEOUT` | Glyph Forge APIが制限時間内に応答しなかった | `504 Gateway Timeout` |
| `UNAVAILABLE` | Glyph Forge APIへの通信に失敗した、または利用できない | `502 Bad Gateway` |
| `INVALID_RESPONSE` | 応答が期待する画像データとして解釈できない | `502 Bad Gateway` |
| `FAILED` | Glyph Forge APIが画像生成に失敗した | `502 Bad Gateway` |

外部APIのHTTPステータス、レスポンス本文、例外メッセージ、スタックトレース、内部URL、認証情報は `ImageGenerationPortError` の公開属性に含めない。

`retryAfter` は、Glyph Forge APIから安全に解釈できる値を取得できた場合だけ設定する。Controllerは設定された値を `Retry-After` ヘッダーへ変換できる。

## 6. GlyphForgeImageGenerationAdapter

`GlyphForgeImageGenerationAdapter` は `ImageGenerationPort` のInfrastructure実装である。

### 責務

- `ImageGenerationRequest` をGlyph Forge API向けのリクエストへ変換する
- `ImageType` に対応するエンドポイントを選択する
- `HexColor` を `RgbColor` へ変換する
- Glyph Forge APIへHTTPリクエストを送信する
- 応答のステータス、ヘッダー、Content-Type、画像データを検証する
- Glyph Forge APIの応答を `GeneratedImage` へ変換する
- 通信エラー、タイムアウト、外部APIエラーを `ImageGenerationPortError` へ変換する

### 責務に含めないもの

- HTTP入力値の必須・形式・範囲の検証
- `ImageGenerationRequest` の不変条件の維持
- `Accept-Language` に基づくメッセージの生成
- 公開APIのHTTPステータスコードの決定
- リクエスト単位のレート制限の判定
- Glyph Forge APIの業務ルールをmojica側へ複製すること

## 7. エンドポイント振り分け

Adapterは `ImageType` の値を次のGlyph Forge APIエンドポイントへ変換する。

| `ImageType` | Glyph Forge API |
| --- | --- |
| `standard` | `POST /images` |
| `x-background` | `POST /images/background` |
| `x-icon` | `POST /images/x-icon` |

この変換表はAdapter内に閉じ込める。`ImageType`、Service、Controllerは外部APIのパスを保持しない。

未定義の `ImageType` はModelで生成できないため、Adapterのエンドポイント変換へ到達させない。

## 8. 色情報の変換

Portの契約では、Modelの `HexColor` を使用する。Glyph Forge API固有の色DTOはPortの入力に含めない。

Adapterは外部APIへ送信する直前に、次の変換を行う。

```text
HexColor("#FF69B4")
        │
        ▼
RgbColor(red: 255, green: 105, blue: 180)
        │
        ▼
Glyph Forge API color DTO
```

HEX形式の検証とRGB各成分の範囲検証はModelのValue Objectが担当する。AdapterはGlyph Forge APIのリクエスト形状への変換だけを担当し、同じ業務ルールを重複実装しない。

## 9. 外部API応答の変換

Adapterは次の順序で応答を処理する。

1. HTTPステータスと必要なヘッダーを確認する
2. レート制限と `Retry-After` を確認する
3. Content-Typeが画像形式として扱えるか確認する
4. 画像バイナリが存在し、読み取り可能か確認する
5. 外部APIの応答をDomainの画像結果へ変換する
6. 失敗した場合は `ImageGenerationPortError` を返す

外部APIのエラー本文をそのままService、Controller、公開APIレスポンスへ渡さない。公開メッセージへの変換はControllerまたはエラー表示の境界で行う。

## 10. タイムアウトとキャンセル

ServiceまたはControllerから渡されたキャンセルトークンがある場合、AdapterはHTTPリクエストへ伝播させる。

Glyph Forge APIが設定された制限時間内に応答しない場合は `TIMEOUT` を返す。タイムアウト後に同じリクエストをAdapterが自動再試行しない。再試行方針が必要になった場合は、重複生成のリスクとGlyph Forge APIの契約を確認したうえで別途決定する。

## 11. Portのテスト契約

Portのテストは外部APIの実通信ではなく、Portから観測できる契約を検証する。

最低限、次の振る舞いを確認する。

- 有効な `ImageGenerationRequest` で画像生成結果を返す
- `standard` が `/images` へ振り分けられる
- `x-background` が `/images/background` へ振り分けられる
- `x-icon` が `/images/x-icon` へ振り分けられる
- HEXカラーが正しいRGB値へ変換される
- 成功結果に画像データとメディア形式が含まれる
- Glyph Forge APIの429を `RATE_LIMITED` へ変換する
- Glyph Forge APIのタイムアウトを `TIMEOUT` へ変換する
- 通信失敗を `UNAVAILABLE` へ変換する
- 不正な外部API応答を `INVALID_RESPONSE` へ変換する
- 外部APIの内部エラー詳細をPortのエラー結果へ漏出させない

AdapterのUnitテストではHTTPクライアント境界を差し替える。Glyph Forge APIとの実通信を伴う契約テストはInfrastructureのMediumテストとして分離し、利用可能なテスト環境が整った場合に実施する。

## 12. 決定事項

- Glyph Forge APIへの通信契約は `ImageGenerationPort` として表現する
- Infrastructure実装は `GlyphForgeImageGenerationAdapter` とする
- 外部APIのパスとDTOはAdapter内に閉じ込める
- 外部APIエラーは `ImageGenerationPortError` へ変換してからServiceへ返す
- PortはModel、HTTP、ASP.NET Core、Glyph Forge API固有型に依存しない
