# mojica API Model設計書

## 1. 目的

画像生成APIの業務ルールを、HTTP、ASP.NET Core、データベース、Glyph Forge APIなどの外部技術から分離して定義する。

この文書でいうModelは、画像生成に必要な値と不変条件を表すDomain Modelである。HTTPリクエスト/レスポンス用DTOやGlyph Forge APIの通信モデルはModelに含めない。

## 2. Modelの責務

Modelは以下を担当する。

- 画像生成に必要な値の表現
- 値の形式・範囲の検証
- 画像種類の表現
- HEXカラーとRGBカラーの値表現および変換
- 画像生成リクエスト全体の不変条件の維持
- Domainで発生した検証結果の表現

Modelは以下を担当しない。

- HTTPリクエストの解析
- `Accept-Language` の解釈
- HTTPステータスコードの決定
- JSONのシリアライズ・デシリアライズ
- データベースへの保存・取得
- Glyph Forge APIへの通信
- ASP.NET CoreやORMへの依存

## 3. 依存方向

```text
Controller / Infrastructure
            │
            ▼
          Model
            ▲
            │
          Service
```

ModelからController、Service、Repository、Infrastructure、HTTP、DB、外部APIを参照しない。

Repository InterfaceをModel側に置く場合も、Modelはその実装やDBの型を参照しない。Repositoryの詳細は別のRepository設計書で定義する。

## 4. Domain Model一覧

| Model | 種類 | 役割 |
| --- | --- | --- |
| `ImageGenerationRequest` | Aggregate / Domain Model | 画像生成に必要な値をまとめ、不変条件を維持する |
| `ImageType` | Enum / Value | 生成する画像の種類を表す |
| `RenderText` | Value Object | 描画対象の文字列を表す |
| `PatternCharacter` | Value Object | 描画または背景に使用する文字列を表す |
| `HexColor` | Value Object | `#RRGGBB` 形式の色を表す |
| `RgbColor` | Value Object | RGB形式の色を表す |
| `GeneratedImage` | Domain Result | 生成された画像データを表す |
| `ModelValidationError` | Domain Error | Modelの検証失敗を表す |

## 5. ImageType

生成する画像の種類を、自由な文字列ではなく定義済みの値として表現する。

| 値 | 説明 | Glyph Forge APIの振り分け先 |
| --- | --- | --- |
| `standard` | 標準画像 | `POST /images` |
| `x-background` | X背景画像 | `POST /images/background` |
| `x-icon` | Xアイコン画像 | `POST /images/x-icon` |

`ImageType` はGlyph Forge APIのパスを保持しない。外部APIのエンドポイントへの変換はInfrastructure側のAdapterが担当する。

未定義の値は `ImageType` として生成できない。外部入力の文字列から `ImageType` へ変換できない場合は、Serviceまたは入力境界でDomainの検証エラーへ変換する。

## 6. RenderText

`RenderText` は、文字アートとして描画する対象の文字列を表す。

### 制約

- 必須
- 1文字以上
- 64文字以下
- 空白文字のみは禁止
- 制御文字は禁止

### 生成規則

生成時に制約を満たさない値を拒否する。生成後の値は常に制約を満たすため、ServiceやInfrastructureが同じ検証を重複して実装しない。

文字数の数え方は、実装時にUnicodeの扱いを固定する。少なくとも、サロゲートペアを構成する文字を不正に2文字として扱わない。

## 7. PatternCharacter

`PatternCharacter` は、描画または背景のパターンを構成する文字列を表す。

用途によって次の2種類を区別する。

| 用途 | Domain上の意味 |
| --- | --- |
| `foregroundCharacter` | 描画に使用する文字 |
| `backgroundCharacter` | 描画文字の周囲に敷き詰める文字 |

### 共通制約

- 必須
- 1文字以上
- 128文字以下
- 制御文字は禁止
- 空白文字のみは単独では許可する

### 相互不変条件

`foregroundCharacter` と `backgroundCharacter` の両方が空白文字のみになることは禁止する。

どちらか一方には、少なくとも1つの表示可能な文字を含める必要がある。これは個々の `PatternCharacter` ではなく、`ImageGenerationRequest` が検証する相互制約である。

## 8. HexColor

`HexColor` は、API境界から受け取ったHEXカラーを正規化して表すValue Objectである。

### 制約

- 必須
- `#RRGGBB` 形式である
- `#` に続く6桁が16進数である
- RGB各値が `0` から `255` の範囲に収まる

### 正規化

内部値は大文字・小文字に依存しない。外部へ文字列化する場合の表記は、正規化された一つの形式に統一する。

```text
#ff69b4
↓
#FF69B4
```

### RGBへの変換

`HexColor` はRGB値を計算できるが、Glyph Forge APIのリクエスト形式は知らない。

```text
#FF69B4
↓
R: 255
G: 105
B: 180
```

Glyph Forge API固有のカラーDTOへの変換はInfrastructure側で行う。

## 9. RgbColor

`RgbColor` は赤・緑・青の各成分を値として保持する。

| 値 | 型 | 制約 |
| --- | --- | --- |
| `red` | 整数 | 0以上255以下 |
| `green` | 整数 | 0以上255以下 |
| `blue` | 整数 | 0以上255以下 |

`RgbColor` は、生成時に各成分の範囲を検証する。負数、255超過、小数、未設定値は生成できない。

## 10. ImageGenerationRequest

`ImageGenerationRequest` は、画像生成ユースケースに渡す検証済みのDomain Modelである。

### 属性

| 属性 | 型 | 必須 |
| --- | --- | :---: |
| `type` | `ImageType` | ○ |
| `text` | `RenderText` | ○ |
| `foregroundCharacter` | `PatternCharacter` | ○ |
| `foregroundColor` | `HexColor` | ○ |
| `backgroundCharacter` | `PatternCharacter` | ○ |
| `backgroundColor` | `HexColor` | ○ |

### 不変条件

- すべての属性が存在する
- 各Value Objectの制約を満たす
- `foregroundCharacter` と `backgroundCharacter` の両方が空白文字のみではない
- 外部APIのエンドポイントやHTTP情報を保持しない

### 生成

外部入力を直接 `ImageGenerationRequest` として扱わない。Controllerまたは入力MapperでHTTP DTOを受け取り、Value Objectを生成した後に `ImageGenerationRequest` を生成する。

生成に失敗した場合、部分的に不正なModelをServiceへ渡さない。

## 11. GeneratedImage

`GeneratedImage` は、画像生成に成功した結果を表すDomain Resultである。

### 属性

| 属性 | 内容 |
| --- | --- |
| `content` | 画像のバイナリデータ |
| `mediaType` | 画像のメディア形式 |
| `fileName` | ダウンロード時に使用するファイル名（必要な場合） |

MVPでは生成画像を永続化しないため、`GeneratedImage` は保存先やデータベースIDを持たない。

Glyph Forge APIのレスポンスDTOから `GeneratedImage` への変換はInfrastructure境界で行う。Controllerは `GeneratedImage` をHTTPレスポンスへ変換する。

## 12. ModelValidationError

Modelの生成または不変条件の検証に失敗した場合は、利用側がエラーを分類できるDomainエラーを返す。

### 属性

| 属性 | 内容 |
| --- | --- |
| `code` | 言語に依存しないエラーコード |
| `target` | 問題のあるDomain属性または属性の組み合わせ |
| `reason` | 機械的に判定できる失敗理由 |
| `details` | 必要に応じた安全な補足情報 |

`ModelValidationError` は日本語・英語の表示メッセージやHTTPステータスコードを持たない。

ServiceまたはControllerが `ModelValidationError` を公開APIのエラー契約へ変換する。表示メッセージは `Accept-Language` に基づいて外側の層で解決する。

## 13. Domainエラーの例

| `code` | `target` | 発生条件 |
| --- | --- | --- |
| `REQUIRED` | 属性名 | 必須値が存在しない |
| `LENGTH_OUT_OF_RANGE` | 属性名 | 文字数が許容範囲外 |
| `CONTROL_CHARACTER` | 属性名 | 制御文字を含む |
| `INVALID_HEX_COLOR` | 色属性名 | `#RRGGBB` 形式ではない |
| `UNSUPPORTED_IMAGE_TYPE` | `type` | 定義されていない画像種類 |
| `VISIBLE_CHARACTER_REQUIRED` | 文字属性の組み合わせ | 両方が空白文字のみ |

公開APIの `VALIDATION_ERROR` やHTTP `422 Unprocessable Entity` への変換はController境界の責務とする。

## 14. API DTOとの対応

APIのJSON DTOとDomain Modelは同一の型として扱わない。

| API DTO | Domain Model |
| --- | --- |
| `type: string` | `ImageType` |
| `text: string` | `RenderText` |
| `foregroundCharacter: string` | `PatternCharacter` |
| `foregroundColor: string` | `HexColor` |
| `backgroundCharacter: string` | `PatternCharacter` |
| `backgroundColor: string` | `HexColor` |

`Accept-Language` はDomain Modelへ渡さず、エラー表示のローカライズに必要な実行コンテキストとして外側の層で扱う。

## 15. テスト契約

Modelは外部サービスやDBを使わずに検証できるようにする。

最低限、次の振る舞いをテストする。

- 有効な `ImageType` を生成できる
- 未定義の `ImageType` を拒否する
- `RenderText` の必須・長さ・空白・制御文字を検証する
- `PatternCharacter` の必須・長さ・制御文字を検証する
- 2つのパターン文字が同時に空白のみになる状態を拒否する
- 有効な `#RRGGBB` を `HexColor` として生成できる
- 不正なHEX形式を拒否する
- HEXからRGBへの変換結果が正しい
- RGB各成分の範囲を検証する
- 有効な値から `ImageGenerationRequest` を生成できる
- 不正な値を含む `ImageGenerationRequest` を生成できない

テストはHTTPステータスコードやGlyph Forge APIの通信ではなく、Modelから観測できる値と検証結果を確認する。HTTP契約と外部API契約のテストは、それぞれController・Infrastructureの境界で行う。

## 16. 未決事項

- 文字数をUnicodeスカラー値で数えるか、ユーザーが認識する書記素クラスタで数えるかを実装前に確定する。
- `GeneratedImage.fileName` の命名規則をControllerまたはServiceの設計時に確定する。
- `ModelValidationError.reason` の具体的な型を、実装言語のResult/Exception方針と合わせて確定する。

未決事項を確定した場合は、この文書と `mvp-api.md` の制約が矛盾しないことを確認する。
