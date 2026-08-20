# AlertBanner

- レイヤー: 共通UI
- 配置: `components/AlertBanner/AlertBanner.tsx`
- 実装基盤: shadcn/ui `Alert`/`AlertTitle`/`AlertDescription` + Lucide `AlertCircle`
- 責務: `role="alert"`のバナー。フィールドに紐づかないエラー表示に使用

特定の入力フィールドに紐づかないエラーメッセージを画面上部にバナー表示するための共通UIコンポーネント。`message`とオプションの`onDismiss`（閉じるボタン）だけを受け取る汎用コンポーネントで、文言の中身までは知らない。実際の利用箇所は[ApiErrorBanner](./ApiErrorBanner.md)（`AlertBanner`をラップし、APIのステータスコードに応じた文言を決定する）。

## Props

```typescript
// components/AlertBanner（ui/Alertを合成する自前コンポーネント）
type AlertBannerProps = {
  message: string;
  onDismiss?: () => void;
};
```

## Storybook

| 主なStory状態 | 検証観点 |
| --- | --- |
| Default（`role="alert"`）／Dismissible（`onDismiss`呼び出し） | `getByRole("alert")`、`onDismiss`のクリック検証（`play`） |

## テスト

- サイズ: Small
- 検証内容: props駆動の表示・`userEvent`による操作・状態遷移
