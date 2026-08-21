# mojica API Model設計書

## 1. 目的

画像生成に必要な値と不変条件をDomain Modelとして定義する。

この文書でいうModelは、画像生成に必要な値と不変条件を表すDomain Modelである。HTTPリクエスト/レスポンス用DTOやGlyph Forge APIの通信モデルはModelに含めない。

## 2. Modelの責務

Modelは以下を担当する。

- 画像生成に必要な値の表現
- 値の形式・範囲の検証
- 画像種類の表現
- HEXカラーとRGBカラーの値表現および変換
- 画像生成リクエスト全体の不変条件の維持
- Domainで発生した検証結果の表現

## 3. Domain Model一覧

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

## 4. ImageType

生成する画像の種類を、自由な文字列ではなく定義済みの値として表現する。

| 値 | 説明 |
| --- | --- |
| `standard` | 標準画像 |
| `x-background` | X背景画像 |
| `x-icon` | Xアイコン画像 |

`ImageType` は画像種別の値だけを表現し、外部システムの情報を保持しない。

未定義の値は `ImageType` として生成できない。

## 5. RenderText

`RenderText` は、文字アートとして描画する対象の文字列を表す。

### 制約

- 必須
- 1文字以上
- 64文字以下
- 空白文字のみは禁止
- 制御文字は禁止

### 生成規則

生成時に制約を満たさない値を拒否する。生成後の値は常に制約を満たす。

文字数はUnicodeの書記素クラスタ単位で数える。絵文字や結合文字を、ユーザーが認識する1文字として扱う。サロゲートペアを構成する文字を2文字として扱わない。

## 6. PatternCharacter

`PatternCharacter` は、描画または背景のパターンを構成する文字列を表す。

文字数の数え方は `RenderText` と同じく、Unicodeの書記素クラスタ単位とする。

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

## 7. HexColor

`HexColor` は、HEXカラーを正規化して表すValue Objectである。

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

`HexColor` はRGB値を計算できる。

```text
#FF69B4
↓
R: 255
G: 105
B: 180
```

## 8. RgbColor

`RgbColor` は赤・緑・青の各成分を値として保持する。

| 値 | 型 | 制約 |
| --- | --- | --- |
| `red` | 整数 | 0以上255以下 |
| `green` | 整数 | 0以上255以下 |
| `blue` | 整数 | 0以上255以下 |

`RgbColor` は、生成時に各成分の範囲を検証する。負数、255超過、小数、未設定値は生成できない。

## 9. ImageGenerationRequest

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

### 生成

外部入力を直接 `ImageGenerationRequest` として扱わない。各Value Objectを生成した後に `ImageGenerationRequest` を生成する。

生成に失敗した場合、`ImageGenerationRequest` を生成しない。

## 10. GeneratedImage

`GeneratedImage` は、画像生成に成功した結果を表すDomain Resultである。

### 属性

| 属性 | 内容 |
| --- | --- |
| `content` | 画像のバイナリデータ |
| `mediaType` | 画像のメディア形式 |
| `fileName` | ダウンロード時に使用するファイル名 |

## 11. ModelValidationError

Modelの生成または不変条件の検証に失敗した場合は、利用側がエラーを分類できるDomainエラーを返す。

### 属性

| 属性 | 内容 |
| --- | --- |
| `code` | 言語に依存しないエラーコード |
| `target` | 問題のあるDomain属性または属性の組み合わせ |
| `reason` | `ModelValidationReason` として表現する機械的に判定できる失敗理由 |
| `details` | 必要に応じた安全な補足情報 |

`ModelValidationError` は表示メッセージを持たない。`reason` は閉じた型である `ModelValidationReason` として表現する。

想定内の検証失敗は例外ではなく、`Result<T, ModelValidationError>` 相当の戻り値で扱う。予期しない実行時障害の例外処理はModelの責務外とする。

## 12. Domainエラーの例

| `code` | `target` | 発生条件 |
| --- | --- | --- |
| `REQUIRED` | 属性名 | 必須値が存在しない |
| `LENGTH_OUT_OF_RANGE` | 属性名 | 文字数が許容範囲外 |
| `CONTROL_CHARACTER` | 属性名 | 制御文字を含む |
| `INVALID_HEX_COLOR` | 色属性名 | `#RRGGBB` 形式ではない |
| `UNSUPPORTED_IMAGE_TYPE` | `type` | 定義されていない画像種類 |
| `VISIBLE_CHARACTER_REQUIRED` | 文字属性の組み合わせ | 両方が空白文字のみ |

## 13. テスト契約

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

## 14. 決定事項

未決事項はない。文字数、生成画像のファイル名、検証失敗の表現は本書の定義に従う。
