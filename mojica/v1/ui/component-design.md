# mojica MVP コンポーネント設計書

本書は[ui.md](./ui.md)の画面仕様を実現するためのフロントエンドコンポーネント設計を示す。API契約は[api.md](../api/api.md)に従う。

デザイン実装は**Tailwind CSS + shadcn/ui + Lucide**を採用する。shadcn/uiはRadix UIベースのコンポーネントをリポジトリへコピーして所有する方式であり、共通UIの実装パターンとして本書§1「コンポーネントカテゴリと配置」の`components/`層に組み込む。

個々の自前コンポーネントの責務・Props・Storybookの状態・テスト対象は[`components/`](./components/)配下のコンポーネントごとのファイルに分けて記載する。ただし、`components/ui/`に置くshadcn/ui CLI生成物は既製primitiveの薄いラッパーであり、個別の設計判断を持たないため、コンポーネントごとの設計書は作成しない。本書には特定の1コンポーネントに紐づかない横断的な内容（コンポーネントカテゴリと配置、状態モデル、非同期境界、採用パターン、i18n・アクセシビリティ・レスポンシブ、既存API契約への影響、テスト方針）のみを記載する。

---

# 1. コンポーネントカテゴリと配置

| レイヤー                      | 責務                                                                  | 依存してよいもの                                              | 配置先                       |
| ----------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------- | ---------------------------- |
| 共通UI（Common components）   | 表示とアクセシビリティのみ。ネイティブHTML要素のprops/eventを拡張する | なし（グローバル状態・ルーティング・fetchを直接importしない） | `components/`                |
| アプリシェル（App shell）     | ロケール状態など横断的な状態を共通UIへ橋渡しするラッパー              | i18nフック（`useTranslations`/`useLocale`など）               | `app/components/`            |
| 機能UI（Business components） | mojica APIへの送信、クライアントバリデーション、ダウンロード処理      | フォーム状態、`POST /images`呼び出し                          | `features/image-generation/` |

`AppHeader`・`AppFooter`は「アプリシェル」として`app/components/`直下に置く。`components/`配下（`TextField`、`Select`等）はi18nフックを含め一切のHooks依存を持たない。

テストコードは`*.stories.tsx`と同様、対象の実装ファイルと同じディレクトリへcolocateする（ファイル名規約`.small.test.ts(x)`/`.medium.test.ts(x)`/`.large.test.ts(x)`を使用し、独立した`tests/`フォルダへ集約しない）。

```text
src/
├── app/
│   ├── components/
│   │   ├── AppHeader/            # Logo + LanguageSwitcher を合成し、i18nフックへ接続する
│   │   │   ├── AppHeader.tsx
│   │   │   ├── AppHeader.stories.tsx
│   │   │   └── AppHeader.small.test.tsx
│   │   ├── AppFooter/
│   │   │   ├── AppFooter.tsx
│   │   │   ├── AppFooter.stories.tsx
│   │   │   └── AppFooter.small.test.tsx
│   │   └── Layout/               # AppHeader + <Outlet /> + AppFooter。routes/__root.tsxのcomponentとして描画される
│   │       ├── Layout.tsx
│   │       ├── Layout.stories.tsx
│   │       └── Layout.small.test.tsx
│   ├── providers/
│   │   ├── AppProviders.tsx     # ErrorBoundary（最外周）→ QueryClientProvider → I18nProviderの順で組み立てるワイヤリング
│   │   └── AppProviders.small.test.tsx  # ErrorBoundaryのfallback表示・Provider配下でのレンダリングを検証
│   └── views/
│       ├── App.tsx              # AppProvidersでRouterProvider（lib/router.tsのrouter）をラップするルートView
│       ├── App.small.test.tsx   # 描画経路のみ（フォーム送信はシミュレートしない）
│       └── App.medium.test.tsx  # MSWで実際にPOST /imagesを発火させ、Provider配線からダウンロードまでを一気通貫で検証
├── components/
│   ├── ui/                      # shadcn/ui CLIが生成するprimitive（Radix UIベース）
│   │   ├── button.tsx
│   │   ├── button.stories.tsx
│   │   ├── input.tsx
│   │   ├── input.stories.tsx
│   │   ├── label.tsx
│   │   ├── label.stories.tsx
│   │   ├── select.tsx
│   │   ├── select.stories.tsx
│   │   ├── alert.tsx
│   │   ├── alert.stories.tsx
│   │   ├── dropdown-menu.tsx
│   │   └── dropdown-menu.stories.tsx
│   ├── TextField/               # ui/input + ui/label + FieldError を合成
│   │   ├── TextField.tsx
│   │   ├── TextField.stories.tsx
│   │   └── TextField.small.test.tsx
│   ├── ColorPickerField/        # input[type=color] + ui/input（HEXテキスト）を合成
│   │   ├── ColorPickerField.tsx
│   │   ├── ColorPickerField.stories.tsx
│   │   └── ColorPickerField.small.test.tsx
│   ├── FieldError/
│   │   ├── FieldError.tsx
│   │   ├── FieldError.stories.tsx
│   │   └── FieldError.small.test.tsx
│   ├── AlertBanner/             # ui/alert をラップ
│   │   ├── AlertBanner.tsx
│   │   ├── AlertBanner.stories.tsx
│   │   └── AlertBanner.small.test.tsx
│   ├── Logo/
│   │   ├── Logo.tsx
│   │   ├── Logo.stories.tsx
│   │   └── Logo.small.test.tsx
│   └── LanguageSwitcher/        # ui/dropdown-menu をラップ。locale・options・onChangeを受け取るcontrolled component
│       ├── LanguageSwitcher.tsx
│       ├── LanguageSwitcher.stories.tsx
│       └── LanguageSwitcher.small.test.tsx
├── features/
│   ├── image-generation/
│   │   ├── components/
│   │   │   ├── ImageGenerationForm/
│   │   │   │   ├── ImageGenerationForm.tsx
│   │   │   │   ├── ImageGenerationForm.stories.tsx
│   │   │   │   └── ImageGenerationForm.medium.test.tsx    # MSW（Orval生成フックが内部で呼ぶfetch）で入力→送信→成功/各エラーを検証
│   │   │   ├── ImageTypeSelect/              # 共通Select（ui/select）をラップ
│   │   │   │   ├── ImageTypeSelect.tsx
│   │   │   │   ├── ImageTypeSelect.stories.tsx
│   │   │   │   └── ImageTypeSelect.small.test.tsx
│   │   │   ├── GenerateButton/                # 共通Button（ui/button）+ lucide-reactのLoader2
│   │   │   │   ├── GenerateButton.tsx
│   │   │   │   ├── GenerateButton.stories.tsx
│   │   │   │   └── GenerateButton.small.test.tsx
│   │   │   └── ApiErrorBanner/
│   │   │       ├── ApiErrorBanner.tsx
│   │   │       ├── ApiErrorBanner.stories.tsx
│   │   │       └── ApiErrorBanner.small.test.tsx
│   │   ├── hooks/
│   │   │   ├── useImageGenerationForm.ts   # React Hook Form（useForm + zodResolver）による入力値・クライアントバリデーション
│   │   │   └── useImageGenerationForm.small.test.ts
│   │   ├── schemas/
│   │   │   ├── imageGenerationSchema.ts    # Zodスキーマ。React Hook Formのresolverとして使用する
│   │   │   └── imageGenerationSchema.small.test.ts
│   │   └── views/
│   │       ├── ImageGenerationScreen.tsx    # ページ本体。ImageGenerationFormを描画する（AppHeader/AppFooterはapp/components/Layoutが担う）
│   │       ├── ImageGenerationScreen.stories.tsx
│   │       └── ImageGenerationScreen.small.test.tsx
│   ├── not-found/
│   │   └── views/
│   │       ├── NotFoundView.tsx             # 404 Not Found画面
│   │       ├── NotFoundView.stories.tsx
│   │       └── NotFoundView.small.test.tsx
│   └── error/
│       └── views/
│           ├── ErrorFallback.tsx            # ErrorBoundaryのfallback UI。ui.md §20の見出し・説明文・再読み込みボタンを描画する
│           ├── ErrorFallback.stories.tsx
│           └── ErrorFallback.small.test.tsx
├── gen/
│   └── api/                     # OrvalがmojicaのOpenAPIスペックから生成するTanStack Queryフック・型。手動編集禁止
│       └── images.ts            # 例: POST /imagesに対応するミューテーションフック（実際のファイル名・フック名はOpenAPIスペックのoperationIdに従う。未確定）
├── lib/
│   ├── utils.ts                 # cn()（clsx + tailwind-merge）。shadcn/ui CLIが生成する
│   ├── queryClient.ts           # TanStack QueryのQueryClientインスタンス初期化設定
│   └── router.ts                # TanStack RouterのcreateRouter({ routeTree })によるrouterインスタンス初期化設定
├── providers/
│   └── I18nProvider.tsx         # i18n実装本体。AppHeader/AppFooterが使うuseTranslations/useLocaleを提供する。localStorageのキー"locale"でロケールを永続化する
└── routes/                      # TanStack Routerのfile-based routing対象ディレクトリ
    ├── __root.tsx                # createRootRouteでLayoutをcomponentに、NotFoundViewをnotFoundComponentに指定する
    ├── __root.small.test.tsx     # ルーター境界でのナビゲーション検証（routeTree.gen.tsから作った本番同等のrouterを使用）
    ├── index.tsx                 # createFileRoute("/")でImageGenerationScreenを描画する
    └── routeTree.gen.ts          # `@tanstack/router-plugin/vite`が自動生成するルートツリー。手動編集禁止
```

`components/`配下は`features/`・`app/`からのグローバル状態・ルーティング・データ取得フックのimportを禁止する。

`I18nProvider`自体の実装は`providers/`に置き、`AppHeader`・`AppFooter`が使うロケール状態を提供する。ロケールは`localStorage`のキー`"locale"`（値は`"ja"`または`"en"`）に永続化する（frontend-architecture.md参照）。`QueryClient`インスタンス自体は`lib/queryClient.ts`に置く。`app/providers/AppProviders.tsx`は`ErrorBoundary`（アプリのルート、§3参照）を最外周に、その内側に`QueryClientProvider`（`gen/api/`のOrval生成フックが必要とする）・`I18nProvider`の順で組み立てる。`ErrorBoundary`を最外周に置くのは、`I18nProvider`や`QueryClientProvider`自体が例外の原因になった場合でも`ErrorFallback`（`features/error/views/`）を表示できるようにするためである。`ErrorFallback`は`I18nProvider`のReact Contextに依存せず、`localStorage`のキー`"locale"`を`I18nProvider`と同じ形式で直接読み取り、コンポーネント内に埋め込んだja/en辞書から表示文言を選択する（ui.md §20）。

`NotFoundView`は`features/not-found/views/`に、`ErrorFallback`は`features/error/views/`に置く。どちらもAPI呼び出しやフォーム状態を持たない点で`features/image-generation/`とは性質が異なるが、「featureそのものの画面」という点は共通するため、`app/views/`や`app/components/`ではなく独立した`features/<feature>/views/`として扱う（frontend-folder-structureの配置決定ワークフロー）。`features/`配下は「mojica APIを呼び出す機能」に限定されず、404表示や予期しないエラー表示のような外部依存のない自己完結した画面も含む。

`ErrorFallback`は`NotFoundView`と異なり、`Layout`の`<Outlet />`を経由せずアプリのルート（`app/providers/AppProviders.tsx`の`ErrorBoundary`）から直接描画される。「featureの画面である」という配置根拠は共通だが、`AppHeader`/`AppFooter`ごと置き換わる点でルーティングの仕組み（`routes/__root.tsx`）には組み込まれない。

`NotFoundView`は`features/not-found/views/`に置く。API呼び出しやフォーム状態を持たない点で`features/image-generation/`とは性質が異なるが、「featureそのものの画面」という点は共通するため、`app/views/`ではなく独立した`features/<feature>/views/`として扱う（frontend-folder-structureの配置決定ワークフロー）。`features/`配下は「mojica APIを呼び出す機能」に限定されず、404表示のような外部依存のない自己完結した画面も含む。

frontend-architecture.mdの通り、404 Not Found画面（ui.md §2, §19）のためにTanStack Routerを導入する。file-based routingを採用し、`routes/`配下のファイルからVite plugin（`@tanstack/router-plugin/vite`）が`routes/routeTree.gen.ts`を自動生成する（手動編集禁止）。`routes/__root.tsx`は`createRootRoute`で`component`に`app/components/Layout.tsx`（`AppHeader` + `<Outlet />` + `AppFooter`）を、`notFoundComponent`に`NotFoundView`を指定する。`routes/index.tsx`は`createFileRoute("/")`で`ImageGenerationScreen`を`component`に指定する。これにより`ImageGenerationScreen`・`NotFoundView`自身はヘッダー・フッターを持たず、画面固有のコンテンツのみを描画する。

`app/views/App.tsx`はエントリポイント（`main.tsx`）から描画されるルートViewであり、`AppProviders`で`RouterProvider`（`lib/router.ts`の`router`。`createRouter({ routeTree })`で`routes/routeTree.gen.ts`から生成）をラップする。`app/views/App.small.test.tsx`は`App`を対象に、ui.md §8の画像生成フロー（入力 → 生成 → 自動ダウンロード）の描画経路のみを検証し、フォーム送信はシミュレートしない。ルート間のナビゲーション（存在しないパスで404画面が表示されること）は個別コンポーネントのテストへ持ち込まず、ルーター境界である`routes/__root.small.test.tsx`に集約する。

`app/views/App.medium.test.tsx`は`App`をエントリポイントとしてMSWで`POST /images`をモックし、実際に入力→送信→成功／エラーまでを一気通貫で検証する。`ImageGenerationForm.medium.test.tsx`とシナリオは重なるが、対象範囲が異なる。`ImageGenerationForm.medium.test.tsx`は`ImageGenerationForm`単体の振る舞いを検証するのに対し、`App.medium.test.tsx`は`QueryClientProvider`・`I18nProvider`・`RouterProvider`・`Layout`までを含めた実際の配線（Provider抜け、ルーティング誤り等）を検証する。この重複は意図的なものであり、Provider配線ミスのような単体テストでは検出できない統合レベルの不具合を拾うために許容する。

テストサイズの分類・命名規則（`.small.test.ts(x)`/`.medium.test.ts(x)`/`.large.test.ts(x)`）と、Playwrightを含むE2Eの扱いは本書§6「テスト項目・残存リスク」で定義する。

Storybookの`*.stories.tsx`は実装ファイルと同じディレクトリへcolocateする（CSF 3.0のcolocationパターン）。`components/ui/`のshadcn/ui primitiveを含め、`components/`・`features/`配下のすべてのコンポーネント・ViewでStoryを作成する。`*.stories.tsx`は`button.tsx`等の実装ファイルとは別ファイルであり、`shadcn add --overwrite`によるCLIの再生成では上書きされない。Storyでカバーする状態は、自前コンポーネントでは各コンポーネントの設計書（[`components/`](./components/)配下）で定義する。`components/ui/`のshadcn/ui primitiveでは、shadcn/uiが公開するvariantと状態をStoryに直接反映する。

shadcn/ui CLIのデフォルトのalias（`components.json`の`aliases.ui`は`@/components/ui`、`aliases.lib`は`@/lib`）をそのまま使用するため、`components.json`のカスタマイズは不要である。

`components/ui/`はshadcn/ui CLIが生成・上書きするvendorレイヤーであり、ファイル名はプロジェクトのPascalCase規則ではなくshadcn/ui標準のkebab-case（例: `button.tsx`、`button.stories.tsx`）をそのまま用いる。`ui/`配下のコンポーネント本体（`button.tsx`等）は**一切手動編集しない**（`*.stories.tsx`はCLIが生成しない自作ファイルのため、この制約を受けない）。カスタマイズは次の方法で行い、`ui/`配下を編集する必要が生じないようにする。

- 色: [design-tokens.md](./design-tokens.md)のCSS変数を変更する（`ui/`のクラスは`bg-primary`等のトークン参照のまま）
- 個別インスタンスの見た目調整: `components/`直下の自前コンポーネント（`TextField`等）から`className` propを渡す（`cn()`＝clsx + tailwind-mergeにより末尾のクラスが優先される）
- Tailwindのスケール全体の変更: `tailwind.config`の`extend`で対応する

構造的な変更や合成自体は、従来通り`components/`直下の自前コンポーネント側で行う。

---

# 2. 状態モデル

送信状態（送信中・成功・失敗）は独自のDiscriminated Unionを定義せず、`gen/api/`のOrval生成ミューテーションフック（TanStack Query `useMutation`）が返す`isPending`/`isError`/`isSuccess`/`error`をそのまま[`ImageGenerationForm`](./components/ImageGenerationForm.md)で使用する。サーバー状態をUI側の型へ複製しないというfrontend-stateの原則に従い、以前検討した独自`SubmissionState`型は採用しない。

個々のバリデーション・エラーマッピング・ダウンロードの具体的な流れは[`ImageGenerationForm`](./components/ImageGenerationForm.md)を参照。

---

# 3. 非同期境界

この画面には初回表示時のデータ取得がなく、非同期処理はボタン押下で開始する`POST /images`（`gen/api/`のミューテーションフック）のみである。データ取得（`useQuery`）を伴わないため`<Suspense>`は使用しない。生成中・成功・失敗はミューテーションフックの状態（`isPending`/`isError`/`isSuccess`）で表現し、ページ全体を覆うローディング表示は行わない。

予期しないレンダリングエラーへの最終防衛線として、`<ErrorBoundary>`は`app/providers/AppProviders.tsx`でアプリのルートにのみ配置する。`POST /images`の失敗はミューテーションフックの`isError`/`error`で扱うため、[`ImageGenerationForm`](./components/ImageGenerationForm.md)配下に個別の`ErrorBoundary`は設けない。`ErrorBoundary`のfallbackは[`ErrorFallback`](./components/ErrorFallback.md)（`features/error/views/`）が担い、`AppHeader`/`AppFooter`を含むアプリ全体を置き換える（ヘッダー・フッターを残す404画面とは異なる。ui.md §20）。

---

# 4. i18n・アクセシビリティ・レスポンシブへの影響

- **i18n**: すべての表示文言（label、button、select選択肢、クライアントバリデーションメッセージ）は翻訳関数経由で描画する。APIのエラーメッセージは`Accept-Language`に応じてサーバー側でローカライズ済みのため、`code`/`errors[].field`のみをUI側の判定に使用し、`message`はそのまま表示する（ui.md §13）。
- **アクセシビリティ**: shadcn/uiの`Select`と[`LanguageSwitcher`](./components/LanguageSwitcher.md)はRadix UI（shadcn/uiの実装基盤）のprimitiveを利用するため、キーボード操作・フォーカス管理・ARIA属性は標準実装として得られる。ただし`aria-describedby`による[`TextField`](./components/TextField.md)/[`ColorPickerField`](./components/ColorPickerField.md)/`Select`と[`FieldError`](./components/FieldError.md)の関連付け、ロゴの`alt`、Lucideアイコンへの`aria-hidden="true"`（アイコン自体は装飾でありテキストラベルが意味を担うため）は個別に実装する。[`AlertBanner`](./components/AlertBanner.md)は`role="alert"`とする。[`GenerateButton`](./components/GenerateButton.md)は`aria-busy`と表示文言（「生成中...」）の両方で状態を伝える（ui.md §14）。
- **レスポンシブ**: フォームは1カラムを基本とし、最大幅設定と中央配置は[`ImageGenerationScreen`](./components/ImageGenerationScreen.md)（ページコンテナ）が担当する。各共通UIコンポーネントは`w-full`を基本とし、横スクロールが発生しないようにする（ui.md §14）。

---

# 5. 既存API契約への影響

本設計はmojica APIのリクエスト/レスポンス契約（[api.md](../api/api.md)）を変更しない。`gen/api/`はこの契約に対応するOpenAPIスペックからOrvalが生成するため、リクエスト/レスポンスの型はapi.mdの変更に追従して再生成される。[`ImageTypeSelect`](./components/ImageTypeSelect.md)の選択肢とAPIの`type`値、`imageGenerationSchema`から`z.infer`した`ImageGenerationFormValues`のキーは、生成されたリクエスト型のフィールド名と一致させ、`errors[].field`を`setError`のフィールド名としてそのまま使用できるようにする。

---

# 6. テスト項目・残存リスク

## テスト方針

テストサイズ（Small/Medium/Large）は、テストの種別やツールではなく実際の依存範囲（外部I/O・複数プロセス・実サービス相当の有無）で判定し、ファイル名で明示する（`.small.test.ts(x)`/`.medium.test.ts(x)`/`.large.test.ts(x)`）。E2E（Playwright）もこの3階層の外に置かず、個々のテストが使用する依存範囲に応じていずれかのサイズへ分類する。ユーザー操作は`userEvent`で行い、`fireEvent`は使用しない。CSSクラス・色・寸法・レイアウトなど視覚的な見た目は自動テストの期待値にしない（VRTに限り、基準画像との比較という形で例外的に視覚差分を検証する）。テストは利用者が実行できる操作と状態遷移を検証する。

テストファイルは本書§1のツリーの通り、対象の実装ファイルと同じディレクトリへcolocateする。独立した`tests/`フォルダは作らない。個々のコンポーネント・View・Hook・スキーマのテスト対象とサイズは、[`components/`](./components/)配下の各ファイルに記載する。

## Storybook・アクセシビリティ

- 各コンポーネントファイルで定義した各Storyが`storybook build`と、`@storybook/addon-vitest`によるVitestテスト実行（`vitest --project=storybook`）で成功すること
- `@storybook/addon-a11y`のaxe検査が全Storyで違反なしであること
- キーボードのみでの入力・色選択・画像種類選択・言語切替・送信（ui.md §15）
- スクリーンリーダーでのエラーメッセージ関連付け
- 言語切り替え時に全表示文言（label、button、エラーメッセージ）が追従すること

CIでは`@storybook/addon-vitest`により、Storyを`storybook`用のVitestプロジェクトとして実行する（`vitest --project=storybook`）。これによりinteraction test（play関数の実行）と、`--coverage`フラグによるカバレッジレポート出力の両方を同じ仕組みでまかなう。これにより`components/ui/`配下のような専用の`.small.test.tsx`を持たないコンポーネントも、CI上でカバレッジとして可視化される。

## E2E（`frontend/e2e/`）

E2E（Playwright）もSmall/Medium/Largeのいずれかへ、実際の依存範囲に基づいて分類する。「E2Eだから」という理由だけでLargeへ分類しない。想定する検証内容：

- ゴールデンパス：入力→生成→実際のダウンロード検出（jsdomでは検証できないPNGダウンロードのブラウザ互換性、§リスク参照）
- VRT：Default／Filled／Submitting／API Error（Retryableボタン+バナー）／404の基準画像比較。Chromium Desktop Chrome固定、動的な時刻・乱数は比較前に固定する
- 存在しないパスへの実URLアクセスと「トップページへ戻る」導線
- レスポンシブ（mobile/tablet、Figmaのフレームに対応）

具体的なファイル（`smoke.spec.ts`等）がどのサイズに該当するかは、実装時に依存範囲を確認して個別に判定する（未決定事項参照）。

## CI実行スケジュール

| イベント                     | 実行するSmall/Medium/Large |
| ---------------------------- | -------------------------- |
| push (main)                  | Small                      |
| pull_request                 | Small, Medium              |
| schedule / workflow_dispatch | Small, Medium, Large       |

E2E専用の実行頻度ルールは設けない。個々のE2Eテストは、そのテストが分類されたサイズに応じたイベントで実行される。

## リスク

すでに決定した設計の結果として受け入れている制約、または別途検証が必要な事項。

- `input[type=color]`のブラウザ間UI差異（OS標準ピッカー）は本設計では吸収しない（ネイティブ要素を採用した結果として受け入れる）
- 生成PNGのダウンロード処理（Blob変換、`Content-Disposition`の`filename`解析）のブラウザ互換性はE2Eで別途検証が必要
