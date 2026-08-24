# AlertBanner

- レイヤー: 共通UI
- 配置: `components/AlertBanner/AlertBanner.tsx`
- 実装基盤: shadcn/ui `Alert`/`AlertTitle`/`AlertDescription` + Lucide `AlertCircle`
- 責務: `role="alert"`のバナー。フィールドに紐づかないエラー表示に使用

特定の入力フィールドに紐づかないエラーを画面上部にバナー表示するための共通UIコンポーネント。見出しと説明文を受け取り、それぞれを`AlertTitle`と`AlertDescription`へ描画する。APIのステータスコードや翻訳文言は知らず、実際の表示内容は[ApiErrorBanner](./ApiErrorBanner.md)が決定する。

## Props

```typescript
// components/AlertBanner（ui/Alertを合成する自前コンポーネント）
type AlertBannerProps = {
  title: string;
  description: string;
};
```

## Storybook

| 主なStory状態                 | 検証観点                                   |
| ----------------------------- | ------------------------------------------ |
| Default（見出し・説明文あり） | `getByRole("alert")`、見出しと説明文の表示 |

## テスト

- サイズ: Small
- 検証内容: `title`が`AlertTitle`、`description`が`AlertDescription`として`role="alert"`内に表示されること
