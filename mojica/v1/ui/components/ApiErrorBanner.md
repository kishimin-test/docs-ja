# ApiErrorBanner

- レイヤー: 機能UI
- 配置: `features/image-generation/components/ApiErrorBanner/ApiErrorBanner.tsx`
- 実装基盤: [AlertBanner](./AlertBanner.md)をラップ
- 責務: APIのステータスコードから表示文言を決定する。`ImageGenerationForm`の先頭に表示する

`AlertBanner`は見た目とアクセシビリティだけを持つ汎用パーツで、`ApiErrorBanner`はAPIのステータスコード（400/429/500/502/504）に応じた表示文言を決定して`AlertBanner`へ渡す機能側のラッパー。表示位置は入力フォームカードの先頭（各入力項目より上）。

## Storybook

| 主なStory状態 | 検証観点 |
| --- | --- |
| 400／429／500／502／504（ui.md §12の文言ごと） | ステータスコードと表示文言の対応 |

## テスト

- サイズ: Small
- 検証内容: props駆動の表示状態（ステータスコード→文言、ui.md §12）
