# GenerateButton

- レイヤー: 機能UI
- 配置: `features/image-generation/components/GenerateButton/GenerateButton.tsx`
- 実装基盤: shadcn/ui `Button` + Lucide `Loader2`（`animate-spin`）
- 責務: 親から受け取った排他的な状態に応じて、文言、アイコン、`aria-busy`、`disabled`、Retryableバリアントを表示する

## Props

```typescript
// features/image-generation/components/GenerateButton
type GenerateButtonProps = {
  state:
    | { kind: "idle" }
    | { kind: "submitting" }
    | { kind: "retryable" }
    | { kind: "cooldown"; remainingSeconds: number };
};
// 内部実装: state.kindがsubmittingのときはLucideの<Loader2 className="animate-spin" aria-hidden="true" />を
// 共通Buttonの子要素として表示し、テキストを「生成中...」へ切り替える
```

`Button`自体に`isLoading`のようなpropは持たせず、`Loader2`を子要素として渡す合成でローディング表示を実現する。`Button`のprimitiveを拡張しないことで、`Button`のAPIを機能UI固有の事情（送信中表示）で汚染しない。`aria-busy`と表示文言（「生成中...」）の両方で状態を伝える（ui.md §14）。

`state`は同時に成立しない状態をDiscriminated Unionで表す。`submitting`と`cooldown`は`disabled`、`idle`と`retryable`は有効とし、呼び出し側から独立した`disabled`を受け取らない。

## 状態ごとの表示

| `kind`       | 表示                                     | `disabled` | `aria-busy` | バリアント  |
| ------------ | ---------------------------------------- | ---------- | ----------- | ----------- |
| `idle`       | 画像を生成する                           | false      | false       | 通常        |
| `submitting` | Loader2 + 生成中...                      | true       | true        | 通常        |
| `retryable`  | 画像を生成する                           | false      | false       | Retryable   |
| `cooldown`   | `{remainingSeconds}秒後に再試行できます` | true       | false       | Retryable   |

`GenerateButton`は`Retry-After`ヘッダーを解釈せず、タイマーも所有しない。カウントダウンは`useRetryAfterCountdown`、APIエラーから`state`への変換は[ImageGenerationForm](./ImageGenerationForm.md)が担当する。

## Storybook

| 主なStory状態                                              | 検証観点                                                           |
| ---------------------------------------------------------- | ------------------------------------------------------------------ |
| Idle／Submitting（Lucide `Loader2`表示）／Retryable／Cooldown | 各`state.kind`に対応する文言、`aria-busy`、`disabled`、バリアント |

## テスト

- サイズ: Small
- 検証内容: 各`state.kind`に対応する表示文言、Loader2、`aria-busy`、`disabled`、Retryableバリアント。時間経過やタイマー実装は検証しない
