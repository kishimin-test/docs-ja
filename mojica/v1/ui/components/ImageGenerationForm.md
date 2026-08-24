# ImageGenerationForm

- レイヤー: 機能UI
- 配置: `features/image-generation/components/ImageGenerationForm/ImageGenerationForm.tsx`
- 実装基盤: 共通UIコンポーネントの合成
- 責務: フォーム状態・送信・エラー表示の統括。`useImageGenerationForm`と`gen/api/`のOrval生成ミューテーションフックを利用する

## Props

```typescript
// features/image-generation/components/ImageGenerationForm
type ImageGenerationFormProps = {
  locale: "ja" | "en";
};
```

## 状態モデル

送信状態（送信中・成功・失敗）は独自のDiscriminated Unionを定義せず、`gen/api/`のOrval生成ミューテーションフック（TanStack Query `useMutation`）が返す`isPending`/`isError`/`isSuccess`/`error`をそのまま使用する。サーバー状態をUI側の型へ複製しないというfrontend-stateの原則に従い、独自`SubmissionState`型は採用しない。生成フックの`isPending`/`isError`/`error`で`SubmissionState`相当の状態を代替でき、独自Discriminated Unionはサーバー状態をUI側へ複製するアンチパターンになるため不採用とした。

- フィールド単位のバリデーションエラーは独自の`FieldErrors`/`FieldName`型を定義せず、`imageGenerationSchema`から`z.infer`した`ImageGenerationFormValues`を型引数とするReact Hook Formの`formState.errors`（`FieldErrors<ImageGenerationFormValues>`）をそのまま使用する。
- クライアントバリデーション（ui.md §11）は`imageGenerationSchema.ts`のZodスキーマとして定義し、`useForm({ resolver: zodResolver(imageGenerationSchema) })`で接続する。`handleSubmit`はバリデーションを通過した場合のみ`onSubmit`を呼び出し、送信をブロックする責務を自前実装しない。
- `handleSubmit`の`onSubmit`から、Orval生成のミューテーションフック（`mutate`/`mutateAsync`）を呼び出す。`422 Unprocessable Entity`は生成フックの`onError`コールバックで受け取り、`errors[].field`をキーとしてReact Hook Formの`setError(field, { type: "server", message })`で同じフィールドへ反映する。フロントエンドのバリデーションを通過していてもAPI側の結果を最終的な正とする（ui.md §11 API側のバリデーションエラー）。
- `400`/`429`/`500`/`502`/`504`は生成フックの`isError`/`error`（フィールドに紐づかないエラー）として[ApiErrorBanner](./ApiErrorBanner.md)に表示する。表示文言はui.md §12の表と、APIレスポンスのローカライズ済み`message`をそのまま使用する（内部エラー情報は表示しない）。
- `429`のレスポンスに`Retry-After`ヘッダーが含まれる場合、その秒数を[GenerateButton](./GenerateButton.md)の`retryAfterSeconds`propへ渡し、カウントダウン表示・`disabled`を制御する（ui.md §12「429のRetry-After」）。
- 生成成功時は`onSuccess`コールバックでレスポンスのPNGを`Content-Disposition`の`filename`を使って自動ダウンロードする（ui.md §10の通りプレビューは保持しない）。
- 送信中は生成フックの`isPending`（および`formState.isSubmitting`）を用いて[GenerateButton](./GenerateButton.md)を`disabled`にし、多重リクエストを防止する（ui.md §9）。

## バリデーションスキーマ（Zod）

```typescript
// features/image-generation/schemas/imageGenerationSchema.ts
import { z } from "zod";

export const imageGenerationSchema = z
  .object({
    text: z.string().trim().min(1).max(64),
    foregroundCharacter: z.string().min(1).max(128),
    foregroundColor: z.string(),
    backgroundCharacter: z.string().min(1).max(128),
    backgroundColor: z.string(),
    type: z.enum(["standard", "x-background", "x-icon"]),
  })
  .refine(
    (values) =>
      values.foregroundCharacter.trim() !== "" ||
      values.backgroundCharacter.trim() !== "",
    {
      message: "foregroundOrBackgroundRequired",
      path: ["foregroundCharacter"],
    },
  );

export type ImageGenerationFormValues = z.infer<typeof imageGenerationSchema>;
```

各制約（文字数・必須・制御文字禁止・空白文字のみ禁止など）はui.md §11の規則をそのままZodのメソッドチェーンへ対応付ける。制御文字禁止のような正規表現制約は`.regex()`で表現する。エラーメッセージはキー（例: `foregroundOrBackgroundRequired`）のみを保持し、実際の表示文言はi18n（ui.md §13）側の翻訳関数で解決する。

`useImageGenerationForm.ts`（React Hook Form + zodResolverによる入力値・クライアントバリデーション）は`imageGenerationSchema`のresolver配線・defaultValuesのみを担い、バリデーションルールの網羅は`imageGenerationSchema`側の責務であり重複させない。

## 非同期境界

この画面には初回表示時のデータ取得がなく、非同期処理はボタン押下で開始する`POST /images`（`gen/api/`のミューテーションフック）のみである。データ取得（`useQuery`）を伴わないため`<Suspense>`は使用しない。生成中・成功・失敗はミューテーションフックの状態（`isPending`/`isError`/`isSuccess`）で表現し、ページ全体を覆うローディング表示は行わない。

予期しないレンダリングエラーへの最終防衛線として、`<ErrorBoundary>`は`app/providers/AppProviders.tsx`でアプリのルートにのみ配置する。`POST /images`の失敗はミューテーションフックの`isError`/`error`で扱うため、`ImageGenerationForm`配下に個別の`ErrorBoundary`は設けない。fallbackは[ErrorFallback](./ErrorFallback.md)が表示する。

## Storybook

| 主なStory状態                                                                                             | 検証観点                                                                                      |
| --------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Default（空）／Filled／ValidationError（Zodスキーマ由来）／ServerError（422の`setError`反映）／Submitting | クライアントバリデーション、`422`時のフィールドエラー反映、送信中の多重クリック防止（`play`） |

`POST /images`はMSWの`http.post`でモックし、実APIへ接続しない。

## テスト

| サイズ | 対象                        | 検証内容                                                                                                                                                                                                                                                                                                  |
| ------ | --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Small  | `imageGenerationSchema.ts`  | ui.md §11の各制約（必須・文字数上限・空白文字のみ禁止・制御文字禁止・文字の組み合わせ）を網羅                                                                                                                                                                                                             |
| Small  | `useImageGenerationForm.ts` | resolver配線・defaultValuesの確認                                                                                                                                                                                                                                                                         |
| Medium | `ImageGenerationForm.tsx`   | MSWで`POST /images`をモックし（Orval生成ミューテーションフックが内部で行うAxiosリクエストをインターセプトする）、入力→送信→成功／422（`errors[].field`の`setError`反映）／400・429・500・502・504（[ApiErrorBanner](./ApiErrorBanner.md)表示）の一連を`userEvent`で検証する、この機能の中心的な統合テスト |

`gen/api/`自体（Orvalが生成したミューテーションフックの実装）に個別のテストは作らない。生成コードは手動編集禁止であり、Orval自身が生成ロジックの正しさを保証する対象のため、二重にテストしない。`ImageGenerationForm.medium.test.tsx`が実際の使用経路として型・リクエスト・レスポンス処理を検証する。
