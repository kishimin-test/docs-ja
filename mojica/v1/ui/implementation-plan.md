# mojica UI ブランチ実装計画

対象設計書：

- [`ui.md`](./ui.md)
- [`component-design.md`](./component-design.md)
- [`design-tokens.md`](./design-tokens.md)

## 1. ブランチ構成

共有ブランチへ直接実装せず、MVP UI全体を親ブランチとして、作業単位ごとの子ブランチで進める。

```text
main
└── feat/mojica-mvp-ui
    ├── feat/mojica-ui-foundation
    ├── feat/mojica-ui-components
    ├── feat/mojica-ui-image-generation
    ├── feat/mojica-ui-error-pages
    └── test/mojica-ui-e2e
```

### 親ブランチ

`feat/mojica-mvp-ui`

- `main` から作成する。
- 子ブランチのマージ先にする。
- 子ブランチを順番にマージし、常にビルド可能な状態を維持する。

### 子ブランチの順序

#### 1. `feat/mojica-ui-foundation`

担当：基盤とデザイントークン

- Material Design 3のCSSカスタムプロパティ
- アプリの共通CSSとレスポンシブ基盤
- Axios、Orval、TanStack Queryの接続基盤
- Zodスキーマの共通配置
- i18nとProviderの最小構成

完了後、`feat/mojica-mvp-ui`へマージする。

#### 2. `feat/mojica-ui-components`

担当：共通UIコンポーネント

- `Logo`
- `FieldError`
- `TextField`
- `ColorPickerField`
- `AlertBanner`
- `LanguageSwitcher`
- `ImageTypeSelect`
- `GenerateButton`
- `AppHeader`、`AppFooter`、`Layout`

完了後、`feat/mojica-mvp-ui`へマージする。

#### 3. `feat/mojica-ui-image-generation`

担当：画像生成画面

- 画像生成フォーム
- クライアントバリデーション
- `POST /images`
- 送信中、成功、422、400、429、500、502、504の表示
- `Retry-After` のカウントダウン
- PNGの自動ダウンロード

依存：`feat/mojica-ui-foundation`、`feat/mojica-ui-components`

完了後、`feat/mojica-mvp-ui`へマージする。

#### 4. `feat/mojica-ui-error-pages`

担当：エラー系画面とルート接続

- 404 Not Found画面
- ErrorBoundaryのfallback画面
- ルート接続
- 404からトップページへの遷移
- エラー画面からの通常ページ再読み込み

依存：`feat/mojica-ui-foundation`、`feat/mojica-ui-components`

完了後、`feat/mojica-mvp-ui`へマージする。

#### 5. `test/mojica-ui-e2e`

担当：外部サービスを含むLargeテスト

- 実ブラウザでの画像生成フロー
- 外部サービス相当の `POST /images`
- PNGダウンロード
- レスポンシブ表示
- 404とエラー画面
- キーボード操作
- VRT

依存：`feat/mojica-ui-image-generation`、`feat/mojica-ui-error-pages`

検証完了後、`feat/mojica-mvp-ui`へマージする。

## 2. マージ順序

```text
feat/mojica-ui-foundation
  ↓
feat/mojica-ui-components
  ↓
feat/mojica-ui-image-generation ─┐
                                 ├─→ feat/mojica-mvp-ui → main
feat/mojica-ui-error-pages ──────┘
                                 ↑
test/mojica-ui-e2e ──────────────┘
```

実際のマージ順は次のとおりとする。

1. `feat/mojica-ui-foundation`
2. `feat/mojica-ui-components`
3. `feat/mojica-ui-image-generation`
4. `feat/mojica-ui-error-pages`
5. `test/mojica-ui-e2e`
6. `feat/mojica-mvp-ui` から `main` へPRを作成する

## 3. コミット単位

各子ブランチでは、次の単位でコミットを分ける。

- `feat:` 基盤、コンポーネント、画面、機能の追加
- `test:` Small、Medium、Largeテストの追加
- `refactor:` 振る舞いを変えない整理
- `fix:` 実装中に発見した不具合の修正
- `docs:` 設計書と実装の差分調整

1コミットに複数の子ブランチの責務を混在させない。自動生成ファイルは生成元の変更と同じ子ブランチで更新する。

## 4. 各ブランチの完了条件

- 担当範囲の設計書と実装が一致している。
- 対象ブランチで利用可能なコード整形が成功している。
- 対象ブランチで利用可能なテスト・型チェック・Lint・ビルドが成功している。
- 対象ブランチでテストカバレッジを取得し、カバレッジレポートの全体サマリーで報告されるすべての指標が80%以上である。
- Largeテストは `test/mojica-ui-e2e` で実行し、親ブランチへマージする前に結果を確認する。
- 子ブランチのPRには、変更範囲、検証結果、未完了事項を記載する。
- 親ブランチから `main` へマージする前に、全子ブランチを統合した状態で全検証を実行する。
