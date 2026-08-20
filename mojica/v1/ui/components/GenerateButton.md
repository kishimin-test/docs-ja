# GenerateButton

- レイヤー: 機能UI
- 配置: `features/image-generation/components/GenerateButton/GenerateButton.tsx`
- 実装基盤: shadcn/ui `Button` + Lucide `Loader2`（`animate-spin`）
- 責務: 「画像を生成する」/「生成中...」の文言と`disabled`を制御する。APIエラー後はRetryableバリアント（ui.md §12）を表示する

## Props

```typescript
// features/image-generation/components/GenerateButton
type GenerateButtonProps = {
  isSubmitting: boolean;
  disabled: boolean;
  retryAfterSeconds?: number; // 429のRetry-Afterヘッダー由来（秒）。指定時はカウントダウン表示でdisabledにする
};
// 内部実装: isSubmitting時はLucideの<Loader2 className="animate-spin" aria-hidden="true" />を
// 共通Buttonの子要素として表示し、テキストを「生成中...」へ切り替える
```

`Button`自体に`isLoading`のようなpropは持たせず、`Loader2`を子要素として渡す合成でローディング表示を実現する。`Button`のprimitiveを拡張しないことで、`Button`のAPIを機能UI固有の事情（送信中表示）で汚染しない。`aria-busy`と表示文言（「生成中...」）の両方で状態を伝える（ui.md §14）。

## Retry-Afterカウントダウン（ui.md §12）

`retryAfterSeconds`が指定されている間は、`disabled`propの値によらず内部で強制的に`disabled`にし、表示文言を「{秒数}秒後に再試行できます」に切り替える。1秒ごとに内部タイマーで秒数を減らし、0になったら通常の「画像を生成する」（Retryableバリアント）表示へ戻して有効化する。カウントダウン中もAPIへの追加リクエストは発生しない。

`retryAfterSeconds`の値は[ImageGenerationForm](./ImageGenerationForm.md)が、生成フックの`error`から`429`レスポンスの`Retry-After`ヘッダーを読み取って渡す。ヘッダーが存在しない場合は`retryAfterSeconds`を渡さず、ボタンは即座に有効なままとする。

## Storybook

| 主なStory状態 | 検証観点 |
| --- | --- |
| Default／Loading（`isSubmitting`、Lucide `Loader2`表示）／Disabled／Retryable（Retry-Afterなし）／RetryCountdown（`retryAfterSeconds`指定、カウントダウン表示） | `aria-busy`と表示文言「生成中...」の一致、カウントダウン中は`disabled`かつ秒数表示が1秒ごとに更新されること |

## テスト

- サイズ: Small
- 検証内容: props駆動の表示状態（`isPending`→`aria-busy`・Retryableバリアント）。`retryAfterSeconds`指定時のカウントダウン表示・`disabled`強制・0到達後の自動有効化（フェイクタイマーを使用）
