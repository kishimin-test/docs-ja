# ImageGenerationScreen

- レイヤー: 機能UI
- 配置: `features/image-generation/views/ImageGenerationScreen.tsx`
- 実装基盤: [ImageGenerationForm](./ImageGenerationForm.md)の合成
- 責務: ページ本体。`ImageGenerationForm`を描画する（`AppHeader`/`AppFooter`は[Layout](./Layout.md)が担う）

フォームは1カラムを基本とし、最大幅設定と中央配置は`ImageGenerationScreen`（ページコンテナ）が担当する（ui.md §14）。

## Storybook

| 主なStory状態                                                                  | 検証観点                                                                      |
| ------------------------------------------------------------------------------ | ----------------------------------------------------------------------------- |
| Default／Submitting／Success（自動ダウンロード）／Error（400/429/500/502/504） | `POST /images`をMSWでモックし、成功・各エラー・タイムアウトを固定データで再現 |

`POST /images`はMSWの`http.post`でモックし、実APIへ接続しない。

レスポンシブ表示はStory状態として`Mobile`や`Tablet`を追加せず、必要なStoryを390px・768px・1440pxのviewportで検証する。

## テスト

- サイズ: Small
- 検証内容: `ImageGenerationForm`が描画されることの確認のみ（深い検証は[ImageGenerationForm](./ImageGenerationForm.md)側に譲る）
