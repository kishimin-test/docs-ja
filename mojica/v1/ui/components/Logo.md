# Logo

- レイヤー: 共通UI
- 配置: `components/Logo/Logo.tsx`
- 実装基盤: `src/assets/logo.svg` + 「mojica」ワードマークテキスト
- 責務: ロゴ画像とワードマークを隣接して表示し、ブランドを示す

ロゴ画像には[kishimin/mojicaの参照アセット](https://github.com/kishimin/mojica/blob/8fc1ef9995d52ec02a6fee242eb7498e9a7c1b49/frontend/src/assets/logo.svg)である1254×1254のSVGを使用し、`src/assets/logo.svg`からimportする。ワードマークテキストと同じブランド名を表すため、画像には`alt=""`を指定して支援技術から隠す。可視テキスト「mojica」がアクセシブルネームを担う（ui.md §15）。

## Storybook

| 主なStory状態 | 検証観点                                                   |
| ------------- | ---------------------------------------------------------- |
| Default       | ロゴ画像と可視テキスト「mojica」が隣接して表示され、テキストがアクセシブルネームを担うことの確認 |

## テスト

- サイズ: Small
- 検証内容: ロゴ画像と可視テキスト「mojica」が提供され、画像の代替テキストが空で重複して読み上げられないこと
