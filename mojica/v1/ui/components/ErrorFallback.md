# ErrorFallback

- レイヤー: 機能UI（`features/error/views/`）
- 配置: `features/error/views/ErrorFallback.tsx`
- 実装基盤: Tailwindのみ
- 責務: `ErrorBoundary`のfallback UI。レンダリング中に予期しない例外が発生した場合に、アプリ全体（[AppHeader](./AppHeader.md)・[AppFooter](./AppFooter.md)を含む）の代わりに表示される（ui.md §20）

API呼び出しやフォーム状態は持たないが、「featureそのものの画面」という点で[NotFoundView](./NotFoundView.md)と同じ扱いとし、`app/components/`ではなく`features/error/views/`に置く（frontend-folder-structureの配置決定ワークフロー）。

## 404画面との違い

`NotFoundView`は[Layout](./Layout.md)の`<Outlet />`の中身だけが差し替わるため`AppHeader`/`AppFooter`は残るが、`ErrorFallback`は`ErrorBoundary`がアプリのルート（`app/providers/AppProviders.tsx`）に配置されているため、`AppHeader`/`AppFooter`を含む画面全体が置き換わる。`NotFoundView`は`routes/__root.tsx`の`notFoundComponent`としてルーティングの仕組みに組み込まれるが、`ErrorFallback`はルーティングとは無関係に`ErrorBoundary`から直接描画される。

`ErrorBoundary`は`AppProviders`の最外周（`QueryClientProvider`・`I18nProvider`より外側）に配置する。`I18nProvider`自体が例外の原因になった場合でも`ErrorFallback`を表示できるようにするためで、`ErrorFallback`は`I18nProvider`のReact Contextを使わない。

## i18n

`ErrorFallback`は翻訳関数（`useTranslations`等）を使わず、`localStorage`に保存されたロケール値を直接読み取り、コンポーネント内に埋め込んだ最小限のja/en辞書から表示文言を選択する。`I18nProvider`のContextツリーの外側で動作するため、`I18nProvider`自体がクラッシュしていてもロケールに応じた表示を維持できる。

```typescript
// features/error/views/ErrorFallback.tsx（イメージ）
const messages = {
  ja: {
    heading: "エラーが発生しました",
    description: "予期しない問題が発生しました。しばらくしてからページを再読み込みしてください。",
    button: "ページを再読み込み",
  },
  en: {
    heading: "An error occurred",
    description: "Something unexpected happened. Please reload the page and try again.",
    button: "Reload page",
  },
} as const;

function getLocale(): "ja" | "en" {
  const stored = localStorage.getItem("locale"); // I18nProviderと同じキー（component-design.md参照）
  return stored === "en" ? "en" : "ja";
}
```

`localStorage`のキー名`"locale"`は`I18nProvider`（`providers/I18nProvider.tsx`）が永続化に使うキーと同一のものを直接読み取る（component-design.md参照）。値の形式（`"ja"`/`"en"`）もI18nProvider側と一致させ、それ以外の値やキー未設定時は`"ja"`にフォールバックする。

## 画面仕様（ui.md §20）

- 見出し: 「エラーが発生しました」（en: "An error occurred"）
- 説明文: 「予期しない問題が発生しました。しばらくしてからページを再読み込みしてください。」（en: "Something unexpected happened. Please reload the page and try again."）
- ボタン: 「ページを再読み込み」（en: "Reload page"）

ボタン押下時はクライアントサイドルーティングではなく、ブラウザの通常のページ再読み込み（`window.location.reload()`相当）を行う。Reactの状態自体が壊れている可能性があり、アプリ内遷移では復旧を保証できないため。mojica APIへのリクエストは発生しない。

## Storybook

| 主なStory状態 | 検証観点 |
| --- | --- |
| Default（ja、`localStorage`未設定時のフォールバック）／en（`localStorage`に`en`を設定） | 見出し・説明文・ボタンの表示、`localStorage`の値に応じた言語切り替え |

## テスト

- サイズ: Small
- 検証内容: `ErrorFallback`単体の表示。`localStorage`の値（未設定／`ja`／`en`）に応じて見出し・説明文・ボタンの言語が切り替わることを`userEvent`不要のprops/環境駆動テストとして検証する。`ErrorBoundary`が実際に子の例外を捕捉して`ErrorFallback`を表示することの検証は`AppProviders.small.test.tsx`が担う（[App](./App.md)参照）
