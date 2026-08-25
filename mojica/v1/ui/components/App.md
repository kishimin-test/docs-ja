# App

- レイヤー: App View（`app/views/`）
- 配置: `app/views/App.tsx`
- 実装基盤: `AppProviders`・`RouterProvider`の合成
- 責務: ルートView。`main.tsx`から描画され、`AppProviders`で`RouterProvider`（`lib/router.ts`の`router`。`createRouter({ routeTree })`で`routes/routeTree.gen.ts`から生成）をラップする

`AppProviders`（`app/providers/AppProviders.tsx`）は`ErrorBoundary`（アプリのルート）を最外周に、その内側に`QueryClientProvider`・`I18nProvider`の順で組み立てるワイヤリングを担う。

## テスト

| サイズ                        | 検証内容                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Small（`App.small.test.tsx`） | `AppProviders`・`RouterProvider`の描画経路と、`App`をエントリポイントにした入力→送信→成功／エラーを1ファイルで検証する。`POST /images`はMSWが実ネットワークへ出る前に同一プロセス内で捕捉し、`QueryClientProvider`・`I18nProvider`・`RouterProvider`・`Layout`の配線までを対象にする。複数モジュールの統合とMSWの使用自体はサイズをMediumへ上げる条件ではない。ルート間遷移の検証は`routes/__root.tsx`のテストに譲る。[ImageGenerationForm](./ImageGenerationForm.md)のテストとのシナリオ重複は、対象範囲（単体の振る舞い vs Provider配線を含む統合）が異なるため意図的に許容する。 |

`app/providers/AppProviders.tsx`自体のSmallテストは、`ErrorBoundary`が子の例外を捕捉し[ErrorFallback](./ErrorFallback.md)を表示することを検証する（`AppProviders.small.test.tsx`）。
