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

`ErrorFallback`は翻訳関数（`useTranslations`等）を使わず、コンポーネント内に埋め込んだ最小限の辞書から表示文言を選択する。`I18nProvider`のContextツリーの外側で動作するため、`I18nProvider`自体がクラッシュしていてもロケールに応じた表示を維持できる。対応ロケール型は辞書のキーから導出し、言語追加時にロケール判定の条件分岐を増やさない。

```typescript
// features/error/views/ErrorFallback.tsx（イメージ）
const messages = {
  ja: {
    heading: "エラーが発生しました",
    description:
      "予期しない問題が発生しました。しばらくしてからページを再読み込みしてください。",
    button: "ページを再読み込み",
  },
  en: {
    heading: "An error occurred",
    description:
      "Something unexpected happened. Please reload the page and try again.",
    button: "Reload page",
  },
} as const;

type SupportedLocale = keyof typeof messages;

const defaultLocale: SupportedLocale = "ja";

const isSupportedLocale = (value: string): value is SupportedLocale =>
  Object.hasOwn(messages, value);

const readStoredLocale = (): string | null => {
  try {
    return localStorage.getItem("locale");
  } catch {
    return null;
  }
};

const resolveLocale = (): SupportedLocale => {
  const candidates = [readStoredLocale(), ...navigator.languages];

  for (const candidate of candidates) {
    const normalized = candidate?.toLowerCase();
    if (normalized && isSupportedLocale(normalized)) return normalized;

    const language = normalized?.split("-")[0];
    if (language && isSupportedLocale(language)) return language;
  }

  return defaultLocale;
};
```

`localStorage`のキー名`"locale"`は`I18nProvider`（`providers/I18nProvider.tsx`）が永続化に使うキーと同一のものを直接読み取る（component-design.md参照）。保存値が未設定・未対応、または`localStorage`を読み取れない場合は`navigator.languages`を順に確認する。言語タグ全体で照合した後、`en-US`から`en`のように先頭の言語サブタグでも照合する。対応するブラウザ言語もなければ既定ロケール`ja`へフォールバックする。辞書のキーは小文字の言語タグとし、新しい言語は辞書へ追加して`I18nProvider`側の対応ロケールと一致させる。

## 画面仕様（ui.md §20）

- 見出し: 「エラーが発生しました」（en: "An error occurred"）
- 説明文: 「予期しない問題が発生しました。しばらくしてからページを再読み込みしてください。」（en: "Something unexpected happened. Please reload the page and try again."）
- ボタン: 「ページを再読み込み」（en: "Reload page"）

ボタン押下時はクライアントサイドルーティングではなく、ブラウザの通常のページ再読み込み（`window.location.reload()`相当）を行う。ルートの`ErrorBoundary`ではProviderを含むReactツリーの状態が壊れている可能性があり、アプリ内遷移だけでは初期状態からの復旧を保証できないためである。クリックハンドラーからmojica APIを直接呼び出さない。再読み込み後に発生する通信は、その時点の通常の初期表示処理に従う。

## Storybook

| 主なStory状態                                                                           | 検証観点                                                                   |
| --------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Default（既定ロケール）／Supported Locale（代表的な非既定ロケール）／Unsupported Locale | 見出し・説明文・ボタンの表示、保存値・ブラウザ言語・既定ロケールの優先順位 |

## テスト

- サイズ: Small
- 検証内容: `ErrorFallback`単体の表示。辞書に定義された各ロケールの文言、保存値を優先すること、保存値が未設定・未対応・読み取り不能の場合にブラウザ言語または既定ロケールへフォールバックすることを検証する。`userEvent`で再読み込みボタンを押し、ページ再読み込み処理が呼ばれることも確認する。`ErrorBoundary`が実際に子の例外を捕捉して`ErrorFallback`を表示することの検証は`AppProviders.small.test.tsx`が担う（[App](./App.md)参照）
