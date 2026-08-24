# AppHeader

- レイヤー: アプリシェル（`app/components/`）
- 配置: `app/components/AppHeader/AppHeader.tsx`
- 実装基盤: [Logo](./Logo.md)・[LanguageSwitcher](./LanguageSwitcher.md)の合成
- 責務: `Logo`と`LanguageSwitcher`を配置し、ロケール状態をi18nフックへ橋渡しする

`components/`配下（[Logo](./Logo.md)、[LanguageSwitcher](./LanguageSwitcher.md)等）はi18nフックを含め一切のHooks依存を持たないが、`AppHeader`はアプリシェルとしてi18nフック（`useTranslations`/`useLocale`など）への依存を持つ。

## Storybook

| 主なStory状態                                | 検証観点                                                          |
| -------------------------------------------- | ----------------------------------------------------------------- |
| Default（ja）／Default（en）／Mobile／Tablet | i18n Providerを`decorators`で注入し、ロケールごとの表示文言を確認 |

## テスト

- サイズ: Small
- 検証内容: i18nロケール切り替えによる表示文言
