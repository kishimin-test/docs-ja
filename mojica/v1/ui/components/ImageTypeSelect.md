# ImageTypeSelect

- レイヤー: 機能UI
- 配置: `features/image-generation/components/ImageTypeSelect/ImageTypeSelect.tsx`
- 実装基盤: shadcn/ui `Select`をラップ
- 責務: OpenAPIから生成された画像種類の値と、i18nの表示ラベルを対応付ける

## Props

```typescript
// features/image-generation/components/ImageTypeSelect
import type { ImageGenerationRequest } from "@/gen/api";

type ImageType = ImageGenerationRequest["type"];

type ImageTypeSelectProps = {
  value: ImageType;
  onChange: (value: ImageType) => void;
  errorMessage?: string;
};
```

実際の生成型名とimport元は、OpenAPIスペックのschema名およびOrvalの生成結果に従う。`ImageTypeSelect`内でAPI値の文字列Unionを再定義せず、Orvalが生成したリクエスト型の`type`プロパティから導出する。

Orvalが画像種類の列挙値を実行時オブジェクトとして生成する場合は、その生成物から選択肢の値一覧を作る。型だけが生成される場合は、UI側で定義する値一覧を`ImageType`で静的に検査し、OpenAPI契約に存在しない値を追加できないようにする。

「標準画像」などの表示ラベルはAPI値ではなくUI文言であるため、OpenAPI生成物には持たせず、API値をキーとしてi18nの翻訳辞書から取得する。選択肢のAPI値は生成型に、表示ラベルは翻訳辞書にそれぞれ追従させる（既存API契約への影響はcomponent-design.md §5参照）。

## Storybook

| 主なStory状態                           | 検証観点                                                       |
| --------------------------------------- | -------------------------------------------------------------- |
| Default（`standard`）／各選択肢切り替え | キーボード操作での選択肢切り替え、`onChange`呼び出し（`play`） |

## テスト

- サイズ: Small
- 検証内容: props駆動の表示状態
