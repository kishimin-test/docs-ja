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

| 主なStory状態 | 検証観点 |
| --- | --- |
| Default／Filled（HEX値表示）／Error／Disabled | HEXテキスト入力とカラーピッカーの同期、`onChange`の呼び出し（`play`） |

## テスト

- サイズ: Small
- 検証内容: props駆動の表示・`userEvent`による操作・状態遷移
