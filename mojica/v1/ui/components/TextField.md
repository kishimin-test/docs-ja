# TextField

- レイヤー: 共通UI
- 配置: `components/TextField/TextField.tsx`
- 実装基盤: shadcn/ui `Input` + `Label` + [FieldError](./FieldError.md)を合成
- 責務: label・input・`FieldError`を1組にした入力欄。「描画する文字列」「描画に使う文字」「敷き詰める文字」で使用

## Props

```typescript
// components/TextField（ui/Input・ui/Labelを合成する自前コンポーネント）
type TextFieldProps = React.ComponentPropsWithoutRef<"input"> & {
  label: string;
  errorMessage?: string;
};
```

`onClick`・`disabled`・`type`・`aria-*`などのネイティブpropsは独自シグネチャで再定義しない。

## Storybook

| 主なStory状態 | 検証観点 |
| --- | --- |
| Default／Filled／Error（バリデーションメッセージ付き）／Disabled | `getByLabelText`でinputとlabelの関連付け、`aria-describedby`によるエラー関連付け |

## テスト

- サイズ: Small
- 検証内容: props駆動の表示・`userEvent`による操作・状態遷移
