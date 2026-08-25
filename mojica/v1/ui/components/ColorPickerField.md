# ColorPickerField

- レイヤー: 共通UI
- 配置: `components/ColorPickerField/ColorPickerField.tsx`
- 実装基盤: ネイティブ`input[type=color]` + shadcn/ui `Input`（HEXテキスト）
- 責務: カラーピッカー入力欄。フロントエンドではHEX文字列で値を保持する

## Props

```typescript
// components/ColorPickerField
type ColorPickerFieldProps = Omit<
  React.ComponentPropsWithoutRef<"input">,
  "type" | "value" | "onChange"
> & {
  label: string;
  value: string; // HEX形式（例: "#FFD400"）
  onChange: (hex: string) => void;
  errorMessage?: string;
};
```

## Storybook

| 主なStory状態                                 | 検証観点                                                                                           |
| --------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| Default／Filled（HEX値表示）／Error／Disabled | HEXテキスト入力またはカラーピッカーを操作すると、両方に同じHEX値が表示されることを`play`で検証する |

## テスト

- サイズ: Small
- 検証内容: 初期HEX値とエラー文言が表示されること、HEXテキスト入力またはカラーピッカーを操作すると両方の表示値が同期すること、Disabledでは値を変更できないことを`userEvent`で検証する
