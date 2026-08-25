# NotFoundView

- レイヤー: 機能UI（`features/not-found/views/`）
- 配置: `features/not-found/views/NotFoundView.tsx`
- 実装基盤: Tailwindのみ
- 責務: 404 Not Found画面。`routes/__root.tsx`の`notFoundComponent`として描画される。トップページへ戻るリンクを提供する

API呼び出しやフォーム状態を持たない点で[ImageGenerationForm](./ImageGenerationForm.md)等とは性質が異なるが、「featureそのものの画面」という点は共通するため、`app/views/`ではなく独立した`features/not-found/views/`として扱う（frontend-folder-structureの配置決定ワークフロー）。`features/`配下は「mojica APIを呼び出す機能」に限定されず、404表示のような外部依存のない自己完結した画面も含む。

## Storybook

| 主なStory状態 | 検証観点                                           |
| ------------- | -------------------------------------------------- |
| Default       | 見出し・説明文・「トップページへ戻る」リンクの表示 |

## テスト

- サイズ: Small
- 検証内容: 見出し・説明文・「トップページへ戻る」リンクの表示
