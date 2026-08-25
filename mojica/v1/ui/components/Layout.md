# Layout

- レイヤー: アプリシェル（`app/components/`）
- 配置: `app/components/Layout/Layout.tsx`
- 実装基盤: [AppHeader](./AppHeader.md) + `<Outlet />` + [AppFooter](./AppFooter.md)
- 責務: ルーターの各ルートを共通レイアウトでラップする

`routes/__root.tsx`の`createRootRoute`で`component`に指定される。TanStack Routerの`<Outlet />`はマッチした子ルートのコンテンツを差し込むプレースホルダーで、通常時は`routes/index.tsx`（[ImageGenerationScreen](./ImageGenerationScreen.md)）、未定義パスでは`notFoundComponent`（[NotFoundView](./NotFoundView.md)）が差し込まれる。これにより`ImageGenerationScreen`・`NotFoundView`自身はヘッダー・フッターを持たない。

## Storybook

| 主なStory状態                                       | 検証観点                                                                   |
| --------------------------------------------------- | -------------------------------------------------------------------------- |
| Default（`<Outlet />`にダミーコンテンツを流し込む） | `AppHeader`・`AppFooter`が常に表示され、中央に子要素が描画されることの確認 |

## テスト

- サイズ: Small
- 検証内容: `Layout`の`<Outlet />`と`AppHeader`/`AppFooter`の常時表示
