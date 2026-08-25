# mojica MVP コンポーネント設計書

本書は[ui.md](./ui.md)の画面仕様を実現するためのフロントエンドコンポーネント設計を示す。API契約は[api.md](../api/api.md)に従う。

実装は**React + TypeScript + Vite**を使用する。サーバー状態は**TanStack Query**、HTTP通信は**Axios**、入力スキーマは**Zod**で扱う。APIクライアントと型は**Orval**から生成する。

個々の自前コンポーネントの責務・Props・Storybookの状態・テスト対象は[`components/`](./components/)配下のコンポーネントごとのファイルに分けて記載する。本書には特定の1コンポーネントに紐づかない横断的な内容（コンポーネントカテゴリと配置、状態モデル、非同期境界、i18n・アクセシビリティ・レスポンシブ、既存API契約への影響、テスト方針）のみを記載する。

---

# 1. コンポーネントカテゴリと配置

| レイヤー                      | 責務                                                                  | 依存してよいもの                                              | 配置先                       |
| ----------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------- | ---------------------------- |
| 共通UI（Common components）   | 表示とアクセシビリティのみ。ネイティブHTML要素のprops/eventを拡張する | なし（グローバル状態・ルーティング・fetchを直接importしない） | `components/`                |
| アプリシェル（App shell）     | ロケール状態など横断的な状態を共通UIへ橋渡しするラッパー              | React Contextなどのアプリ状態                                 | `app/components/`            |
| 機能UI（Business components） | mojica APIへの送信、クライアントバリデーション、ダウンロード処理      | フォーム状態、`POST /images`呼び出し                          | `features/image-generation/` |

`AppHeader`・`AppFooter`は「アプリシェル」として`app/components/`直下に置く。`components/`配下（`TextField`、`Select`等）はアプリ状態やデータ取得への依存を持たない。

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
│       ├── App.tsx              # AppProvidersでアプリ全体をラップするルートView
│       └── App.small.test.tsx   # 描画経路と、MSWで制御したPOST /imagesによるProvider配線から成功／エラーまでを検証
├── components/
│   ├── ui/                      # アプリ共通のUI primitive
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
│   │   │   │   └── ImageGenerationForm.large.test.tsx     # 外部サービス相当のAxios通信で入力→送信→成功/各エラーを検証
│   │   │   ├── ImageTypeSelect/              # 共通Select（ui/select）をラップ
│   │   │   │   ├── ImageTypeSelect.tsx
│   │   │   │   ├── ImageTypeSelect.stories.tsx
│   │   │   │   └── ImageTypeSelect.small.test.tsx
│   │   │   └── GenerateButton/                # 共通Button（ui/button）+ lucide-reactのLoader2
│   │   │   │   ├── GenerateButton.tsx
│   │   │   │   ├── GenerateButton.stories.tsx
│   │   │   │   └── GenerateButton.small.test.tsx
│   │   ├── errors/
│   │   │   ├── toImageGenerationErrorPresentation.ts      # APIのcodeを表示用の見出しへ変換
│   │   │   └── toImageGenerationErrorPresentation.small.test.ts
│   │   ├── hooks/
│   │   │   ├── useImageGenerationForm.ts   # Reactのフォーム状態とZodによる入力値・クライアントバリデーション
│   │   │   └── useImageGenerationForm.small.test.ts
│   │   ├── schemas/
│   │   │   ├── imageGenerationSchema.ts    # Zodスキーマによる入力値検証
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
│       └── images.ts            # 例: POST /imagesに対応するミューテーションフック（実際のファイル名・フック名はOpenAPIスペックのoperationIdに従う。）
├── lib/
│   ├── api.ts                   # Orval生成APIクライアントの設定
│   ├── queryClient.ts           # TanStack QueryのQueryClientインスタンス初期化設定
├── providers/
│   └── I18nProvider.tsx         # i18n実装本体。AppHeader/AppFooterが使うuseTranslations/useLocaleを提供する。localStorageのキー"locale"でロケールを永続化する
└── routes/                      # Viteアプリの画面ルート
    ├── __root.tsx                # createRootRouteでLayoutをcomponentに、NotFoundViewをnotFoundComponentに指定する
    ├── __root.small.test.tsx     # ルート境界でのナビゲーション検証
    ├── index.tsx                 # createFileRoute("/")でImageGenerationScreenを描画する
    └── routeTree.gen.ts          # ルート定義
```

`components/`配下は`features/`・`app/`からのグローバル状態・ルーティング・データ取得フックのimportを禁止する。

`I18nProvider`自体の実装は`providers/`に置き、`AppHeader`・`AppFooter`が使うロケール状態を提供する。ロケールは`localStorage`のキー`"locale"`へ永続化する（frontend-architecture.md参照）。`QueryClient`インスタンス自体は`lib/queryClient.ts`に置く。`app/providers/AppProviders.tsx`は`ErrorBoundary`（アプリのルート、§3参照）を最外周に、その内側に`QueryClientProvider`（`gen/api/`のOrval生成フックが必要とする）・`I18nProvider`の順で組み立てる。`ErrorBoundary`を最外周に置くのは、`I18nProvider`や`QueryClientProvider`自体が例外の原因になった場合でも`ErrorFallback`（`features/error/views/`）を表示できるようにするためである。`ErrorFallback`は`I18nProvider`のReact Contextに依存せず、コンポーネント内に埋め込んだ最小限の辞書から表示文言を選択する。対応ロケール型は辞書のキーから導出し、`localStorage`の保存値、ブラウザ言語、既定ロケール`ja`の順で解決する（ui.md §20）。新しい言語を追加する際は、`I18nProvider`と`ErrorFallback`の対応ロケールを一致させる。

`NotFoundView`は`features/not-found/views/`に、`ErrorFallback`は`features/error/views/`に置く。`features/`配下には、404表示や予期しないエラー表示のような外部依存のない自己完結した画面も含める。

`ErrorFallback`は`Layout`の`<Outlet />`を経由せず、アプリのルート（`app/providers/AppProviders.tsx`の`ErrorBoundary`）から直接描画される。`AppHeader`/`AppFooter`ごと置き換わるため、ルーティングの仕組み（`routes/__root.tsx`）には組み込まれない。

404 Not Found画面（ui.md §2, §19）はViteアプリのルート解決で処理する。`routes/`配下では画面コンポーネントを定義し、`ImageGenerationScreen`・`NotFoundView`自身はヘッダー・フッターを持たず、画面固有のコンテンツのみを描画する。

`app/views/App.tsx`はエントリポイント（`main.tsx`）から描画されるルートViewであり、`AppProviders`でアプリ全体をラップする。`app/views/App.small.test.tsx`は`App`を対象に、描画経路と、MSWが実ネットワークへ出る前に同一プロセス内で捕捉する`POST /images`を使った入力→送信→成功／エラーを1ファイルで検証する。`QueryClientProvider`・`I18nProvider`・`Layout`までを含むが、複数モジュールの統合自体はサイズをMediumへ上げる条件ではない。ルート間のナビゲーション（存在しないパスで404画面が表示されること）は`routes/__root.small.test.tsx`に集約する。

テストサイズの分類・命名規則（`.small.test.ts(x)`/`.medium.test.ts(x)`/`.large.test.ts(x)`）と、Playwrightを含むE2Eの扱いは本書§6「テスト項目・残存リスク」で定義する。

Storybookの`*.stories.tsx`は実装ファイルと同じディレクトリへcolocateする（CSF 3.0のcolocationパターン）。`components/`・`features/`配下のすべてのコンポーネント・ViewでStoryを作成する。自前コンポーネントでカバーする状態は各コンポーネントの設計書（[`components/`](./components/)配下）で定義する。

---

# 2. 状態モデル

送信状態（送信中・成功・失敗）は、`gen/api/`のOrval生成ミューテーションフック（TanStack Query `useMutation`）が返す`isPending`/`isError`/`isSuccess`/`error`をそのまま[`ImageGenerationForm`](./components/ImageGenerationForm.md)で使用する。フォーム入力の検証にはZodスキーマを使用する。

個々のバリデーション・エラーマッピング・ダウンロードの具体的な流れは[`ImageGenerationForm`](./components/ImageGenerationForm.md)を参照。

---

# 3. 非同期境界

この画面には初回表示時のデータ取得がなく、非同期処理はボタン押下で開始する`POST /images`（`gen/api/`のミューテーションフック）のみである。データ取得（`useQuery`）を伴わないため`<Suspense>`は使用しない。生成中・成功・失敗はミューテーションフックの状態（`isPending`/`isError`/`isSuccess`）で表現し、ページ全体を覆うローディング表示は行わない。

予期しないレンダリングエラーへの最終防衛線として、`<ErrorBoundary>`は`app/providers/AppProviders.tsx`でアプリのルートにのみ配置する。`POST /images`の失敗はミューテーションフックの`isError`/`error`で扱うため、[`ImageGenerationForm`](./components/ImageGenerationForm.md)配下に個別の`ErrorBoundary`は設けない。`ErrorBoundary`のfallbackは[`ErrorFallback`](./components/ErrorFallback.md)（`features/error/views/`）が担い、`AppHeader`/`AppFooter`を含むアプリ全体を置き換える（ヘッダー・フッターを残す404画面とは異なる。ui.md §20）。

---

# 4. i18n・アクセシビリティ・レスポンシブへの影響

- **i18n**: すべての表示文言（label、button、select選択肢、クライアントバリデーションメッセージ）は翻訳関数経由で描画する。APIのエラーメッセージは`Accept-Language`に応じてサーバー側でローカライズ済みのため、`code`/`errors[].field`のみをUI側の判定に使用し、`message`はそのまま表示する（ui.md §13）。
- **アクセシビリティ**: `aria-describedby`による[`TextField`](./components/TextField.md)/[`ColorPickerField`](./components/ColorPickerField.md)/`Select`と[`FieldError`](./components/FieldError.md)の関連付け、ロゴの`alt`、装飾アイコンへの`aria-hidden="true"`を個別に実装する。[`AlertBanner`](./components/AlertBanner.md)は`role="alert"`とする。[`GenerateButton`](./components/GenerateButton.md)は`aria-busy`と表示文言（「生成中...」）の両方で状態を伝える（ui.md §14）。
- **レスポンシブ**: フォームは1カラムを基本とし、最大幅設定と中央配置は[`ImageGenerationScreen`](./components/ImageGenerationScreen.md)（ページコンテナ）が担当する。各共通UIコンポーネントは`w-full`を基本とし、横スクロールが発生しないようにする（ui.md §14）。

---

# 5. 既存API契約への影響

本設計はmojica APIのリクエスト/レスポンス契約（[api.md](../api/api.md)）を変更しない。`gen/api/`はこの契約に対応するOpenAPIスペックからOrvalが生成するため、リクエスト/レスポンスの型はapi.mdの変更に追従して再生成される。[`ImageTypeSelect`](./components/ImageTypeSelect.md)のPropsは、API値の文字列Unionを再定義せず、生成されたリクエスト型の`type`プロパティから導出する。選択肢の値一覧は、Orvalが列挙値の実行時オブジェクトを生成する場合はその生成物から作り、型だけを生成する場合は生成型による静的検査を必須とする。「標準画像」などの表示ラベルはOpenAPI生成物ではなく、API値をキーとしてi18nの翻訳辞書から取得する。`imageGenerationSchema`から`z.infer`した`ImageGenerationFormValues`のキーは、生成されたリクエスト型のフィールド名と一致させ、`errors[].field`を`setError`のフィールド名としてそのまま使用できるようにする。

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
