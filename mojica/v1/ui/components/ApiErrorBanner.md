# ApiErrorBanner

- レイヤー: 機能UI
- 配置: `features/image-generation/components/ApiErrorBanner/ApiErrorBanner.tsx`
- 実装基盤: [AlertBanner](./AlertBanner.md)をラップ
- 責務: APIのステータスコードから見出しを決定し、APIのローカライズ済み`message`を説明文として`ImageGenerationForm`の先頭に表示する

`AlertBanner`は見出し・説明文の表示とアクセシビリティだけを持つ共通UIで、`ApiErrorBanner`はAPIのステータスコード（400/429/500/502/504）に応じた見出しを決定し、APIから返されたローカライズ済み`message`を説明文として渡す機能側のラッパー。表示位置は入力フォームカードの先頭（各入力項目より上）。

## Storybook

| 主なStory状態                                  | 検証観点                         |
| ---------------------------------------------- | -------------------------------- |
| 400／429／500／502／504（ui.md §12の文言ごと） | ステータスコードと表示文言の対応 |

## テスト

- サイズ: Small
- 検証内容: ステータスコードから見出しを決定し、APIのローカライズ済み`message`を説明文として表示すること（ui.md §12）
