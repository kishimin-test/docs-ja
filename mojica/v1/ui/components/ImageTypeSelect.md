# ImageTypeSelect

- レイヤー: 機能UI
- 配置: `features/image-generation/components/ImageTypeSelect/ImageTypeSelect.tsx`
- 実装基盤: shadcn/ui `Select`をラップ
- 責務: `standard`/`x-background`/`x-icon`の表示名とAPI値を対応付ける

## Props

```typescript
// features/image-generation/components/ImageTypeSelect
type ImageTypeSelectProps = {
  value: "standard" | "x-background" | "x-icon";
  onChange: (value: "standard" | "x-background" | "x-icon") => void;
  errorMessage?: string;
};
```

選択肢とAPIの`type`値は一致させる（既存API契約への影響はcomponent-design.md §5参照）。

## Storybook

| 主なStory状態 | 検証観点 |
| --- | --- |
| Default（`standard`）／各選択肢切り替え | キーボード操作での選択肢切り替え、`onChange`呼び出し（`play`） |

## テスト

- サイズ: Small
- 検証内容: props駆動の表示状態
