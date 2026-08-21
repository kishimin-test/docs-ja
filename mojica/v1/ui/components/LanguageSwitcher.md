# LanguageSwitcher

- レイヤー: 共通UI
- 配置: `components/LanguageSwitcher/LanguageSwitcher.tsx`
- 実装基盤: shadcn/ui `DropdownMenu`（Radix UI Dropdown Menu） + Lucide `ChevronDown`
- 責務: 選択中の言語名 + 展開アイコンのドロップダウン。controlled component

`locale`・`options`・`onChange`を受け取るcontrolled component。実際のi18nフックへの接続は[AppHeader](./AppHeader.md)が担う。

## Storybook

| 主なStory状態 | 検証観点 |
| --- | --- |
| Default（閉）／Open（ja選択）／Open（en選択）／Keyboard Focus | キーボード操作（矢印キー・Enter・Esc）、`role="menu"`のARIA表現 |

## テスト

- サイズ: Small
- 検証内容: props駆動の表示・`userEvent`による操作・状態遷移
