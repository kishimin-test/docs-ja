# mojica UI 実装計画

対象設計書：

- [`ui.md`](./ui.md)
- [`component-design.md`](./component-design.md)
- [`design-tokens.md`](./design-tokens.md)

対象ブランチでは、MVPの画像生成UI、404画面、予期しないエラー画面を実装する。実装は既存の `frontend/package.json` に定義された React、TanStack Query、Axios、Zod、Orval、Storybook、Vitest、MSW、Playwright を使用する。

## 1. 実装前の確認

1. `frontend/` の既存構成、Viteエントリポイント、CSS、API生成設定、テスト設定を確認する。
2. `api.md` とOpenAPI定義の `POST /images` 契約を確認する。
3. `frontend/package.json` の実装済みスクリプトを確認する。
4. Material Design 3のトークン名と、[`design-tokens.md`](./design-tokens.md)のCSSカスタムプロパティを実装の基準にする。
5. 未実装のルーター、i18n、フォーム管理を追加する場合は、先に既存設定と依存関係を確認し、不要な抽象化を作らない。

## 2. 実装順序

### Phase 1: 基盤とデザイントークン

対象：`frontend/src/index.css`、`frontend/src/main.tsx`、共通設定ファイル

- Material 3のカラー役割、タイポグラフィ、角丸、エレベーション、ブレークポイントをCSSカスタムプロパティとして定義する。
- `background`、`surface`、`on-surface`、`primary`、`on-primary`、`error`、`error-container`、`outline`、`outline-variant`を共通参照できるようにする。
- フォント、ページ背景、横幅、余白の基礎スタイルを定義する。
- フォーカスリング、disabled、hover、pressed、loadingの状態を色だけに依存せず定義する。

完了条件：CSS変数が読み込まれ、後続コンポーネントから参照できる。既存のビルドが壊れていない。

### Phase 2: APIクライアントと入力スキーマ

対象：`frontend/src/gen/`、`frontend/src/lib/`、`frontend/src/features/image-generation/schemas/`

- OpenAPIからOrvalのAPIクライアントと型を生成する。
- Axiosの共通設定を作成し、現在のロケールを `Accept-Language` に反映する。
- `POST /images` のリクエスト型とレスポンス型を生成型に合わせる。
- Zodで次の入力を検証する。
  - 描画する文字列：必須、1〜64文字、空白のみ禁止、制御文字禁止
  - 描画に使う文字：必須、1〜128文字、制御文字禁止、空白のみ許可
  - 敷き詰める文字：必須、1〜128文字、制御文字禁止、空白のみ許可
  - 描画に使う文字と敷き詰める文字の両方が空白のみになる組み合わせは禁止
  - 色はHEX形式
  - 画像種類は `standard`、`x-background`、`x-icon`
- APIの422エラーは `errors[].field` をフォームのフィールドへ対応付ける。
- APIエラーのレスポンス形式を保ったまま、画面表示用の状態へ変換する。

検証：スキーマの正常値、境界値、空白、制御文字、組み合わせエラー、APIエラー形式をSmallテストで固定する。

### Phase 3: 共通UIコンポーネント

対象：`frontend/src/components/`

実装順は依存の少ないものから進める。

1. `Logo`
2. `FieldError`
3. `TextField`
4. `ColorPickerField`
5. `AlertBanner`
6. `LanguageSwitcher`
7. `ImageTypeSelect`
8. `GenerateButton`

各コンポーネントで実装する内容：

- Propsで値・状態・イベントを受け取り、APIやグローバル状態を直接参照しない。
- Material 3の色役割、タイポグラフィ、シェイプ、状態をCSSトークンから参照する。
- ラベル、説明、エラーを `aria-describedby` で関連付ける。
- キーボード操作、focus-visible、disabled、error、loadingを実装する。
- Storybookで通常状態、入力済み、エラー、disabled、loading、レスポンシブ状態を定義する。

検証：各コンポーネントの操作可能な振る舞いをSmallテストで検証し、Storybookのinteraction testとa11y検査を通す。

### Phase 4: アプリシェルと状態境界

対象：`frontend/src/app/`、`frontend/src/providers/`、`frontend/src/routes/`

- `I18nProvider`を実装し、ja/enの表示文言とロケール切り替えを管理する。
- ロケールを永続化し、APIリクエストの `Accept-Language` と同期する。
- `QueryClientProvider`、`I18nProvider`、ルートのErrorBoundaryを組み立てる。
- `Layout`に `AppHeader`、画面領域、`AppFooter`を配置する。
- 画像生成画面と404画面をルートへ接続する。
- ErrorBoundaryのfallbackはI18nProviderに依存せず、保存済みロケールから表示文言を選択する。

検証：Providerの組み合わせ、言語切り替え、404遷移、ErrorBoundaryのfallbackをSmallまたはMediumテストで検証する。

### Phase 5: 画像生成機能

対象：`frontend/src/features/image-generation/`

- 画面に見出し、説明文、フォームカード、6つの入力項目、生成ボタンを配置する。
- Zodスキーマで送信前検証を行い、エラーを各フィールドへ表示する。
- 送信時はTanStack Queryのmutationを実行し、`isPending` 中は多重送信を防止する。
- 成功時はPNGレスポンスをBlobとして扱い、自動ダウンロードする。
- APIエラー時は、フィールドエラーとフォーム上部のAPIエラーバナーを分けて表示する。
- 429で `Retry-After` がある場合は、カウントダウン中だけ再送信を無効化する。
- 成功、入力エラー、422、400、429、500、502、504、通信失敗を画面状態として扱う。
- 画像プレビュー、履歴、サーバー保存、手動ダウンロードは実装しない。

実装パターン：入力検証をAPI呼び出しより先に行い、正常系を最小実装した後、エラー状態と副作用を追加する。Green中に抽象化や大規模なリファクタリングを行わない。

検証：

- Small：入力、境界値、フィールドエラー、disabled、文言、状態遷移
- Medium：MSWで `POST /images` をモックし、入力から成功・エラー・自動ダウンロードまで
- Large：実ブラウザでPNGダウンロード、レスポンシブ、404、キーボード操作、VRT

### Phase 6: 404・予期しないエラー画面

対象：`frontend/src/features/not-found/`、`frontend/src/features/error/`

- 404画面に `404`、見出し、説明、トップページへ戻るボタンを配置する。
- ErrorBoundary fallbackにエラー見出し、説明、ページ再読み込みボタンを配置する。
- 404は共通ヘッダー・フッターを表示し、予期しないエラー画面は表示しない。
- 404ではAPIリクエストを発生させず、エラー画面では `window.location.reload()` 相当の通常再読み込みを行う。

検証：存在しないパス、トップページへの遷移、ErrorBoundary発生、再読み込み操作を検証する。

## 3. テストと検証の順序

各Phaseで次の順に実行する。

1. 対象テストを追加または確認する。
2. 対象テストを実行してRedを確認する。
3. 最小実装でGreenにする。
4. 既存テストを実行する。
5. Storybook build、Storybook Vitest、a11y検査を実行する。
6. TypeScript型チェック、Lint、ビルドを実行する。
7. カバレッジを取得し、Small/Medium/Large別に結果を記録する。
8. PlaywrightでLargeテストを実行する。

使用するコマンドは `frontend/package.json` に定義されたものを優先する。

## 4. 完了条件

- `ui.md` の画像生成、404、エラー、i18n、レスポンシブ、アクセシビリティ要件を満たす。
- `component-design.md` の配置、依存境界、状態モデル、テストサイズに一致する。
- `design-tokens.md` のMaterial Design 3トークンを使用する。
- `POST /images` のAPI契約とAPIエラー表示を壊さない。
- Storybook、型チェック、Lint、ビルド、テスト、カバレッジ、Playwrightの利用可能な検証が成功する。
- 実装対象外の機能を追加しない。
