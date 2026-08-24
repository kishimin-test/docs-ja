# mojica デザイントークン（Material Design 3）

本書はFigma（`mojica MVP UI`。`mojica / Default`、`UI Validation Error`、`API Error`、`404 Not Found`、`Tablet / JA`、`Mobile / JA`、`Desktop / EN`の各フレーム）から抽出したデザイントークンを、Google Material Design 3の役割名・タイプスケール・シェイプ・レイアウトへ対応付けたものである。

Material Design 3のカラー役割、状態、レイアウトの考え方に従う。Web実装では、Material Webの依存追加は行わず、CSSカスタムプロパティと既存のReactコンポーネントでトークンを実装する。

Figma上の色・数値は各フレームの塗り（fill）・線（stroke）・レイアウト（padding/gap）・テキストスタイルから抽出しており、ユーザーが入力する「描画文字色」「敷き詰める文字色」のサンプル値（例: `#FFD400`、`#FF69B4`）はアプリの色トークンではないため対象外とする。

---

# 1. 色

色はOKLCHをCSSカスタムプロパティとして定義する（HEXはFigma由来の元値として併記）。OKLCH値はHEXから算出（sRGB→線形RGB→OKLab→OKLCH）。

`background`・`foreground`・`muted-foreground`・`primary`・`primary-foreground`・`border`・`destructive`を共通トークンとして定義する。`surface`・`helper-foreground`・`border-accent`・`inverse`・`inverse-foreground`・`destructive-background`・`destructive-border`はこのプロジェクト固有のトークンとする。

| トークン名（確定）       | HEX       | OKLCH                      | Material 3の役割             | Figma上の用途                                                                              |
| ------------------------ | --------- | -------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------ |
| `background`             | `#F2F9FC` | `oklch(0.978 0.008 225.1)` | `background`（既定）         | ページ全体の背景                                                                           |
| `surface`                | `#FFFFFF` | `oklch(1.000 0.000 0)`     | `card`（既定に対応）         | ヘッダー・フッター・フォームカード・入力欄の背景                                           |
| `foreground`             | `#1F1C1A` | `oklch(0.229 0.006 56.1)`  | `foreground`（既定）         | 見出し・ラベル・本文の主要テキスト色（`card-foreground`もこれと同値）                      |
| `muted-foreground`       | `#6B635C` | `oklch(0.505 0.015 63.7)`  | `muted-foreground`（既定）   | 説明文（Intro）、言語切り替えのシェブロンアイコン                                          |
| `helper-foreground`      | `#496978` | `oklch(0.502 0.044 228.7)` | 独自追加                     | 文字数ヒント（「1〜64文字」等）、フッターのコピーライト                                    |
| `border`                 | `#DBD4C9` | `oklch(0.872 0.017 79.3)`  | `border`（既定）             | 言語切り替え・未入力状態の入力欄の枠線                                                     |
| `border-accent`          | `#BEDEEB` | `oklch(0.881 0.038 224.3)` | outline-variant              | 入力欄・カラーピッカーの枠線（`Input / Empty`、`Select`）                                  |
| `primary`                | `#7CC7E8` | `oklch(0.793 0.088 227.9)` | `primary`（既定）            | 画像生成ボタンの背景（通常時）                                                             |
| `primary-foreground`     | `#193A48` | `oklch(0.330 0.046 228.2)` | `primary-foreground`（既定） | 画像生成ボタンのテキスト色（通常時）                                                       |
| `inverse`                | `#211F1C` | `oklch(0.240 0.006 78.2)`  | 独自追加                     | ロゴのシンボル背景、Retryableバリアントのボタン背景、404の「トップページへ戻る」ボタン背景 |
| `inverse-foreground`     | `#FFFFFF` | `oklch(1.000 0.000 0)`     | 独自追加                     | `inverse`背景上のテキスト色                                                                |
| `destructive`            | `#C72929` | `oklch(0.541 0.194 26.7)`  | `destructive`（既定）        | クライアントバリデーションエラーの枠線・エラーメッセージ文字色                             |
| `destructive-background` | `#FFF2F2` | `oklch(0.971 0.014 17.4)`  | 独自追加                     | APIエラーバナー（`ApiErrorBanner`）の背景                                                  |
| `destructive-border`     | `#EB8C8C` | `oklch(0.740 0.116 20.2)`  | 独自追加                     | APIエラーバナーの枠線                                                                      |

`ring`（フォーカスリング色）はFigma上に定義がなく未抽出（§9参照）。

---

# 2. タイポグラフィ

フォントファミリーはすべて`Inter`。

## フォントサイズ・ウェイト

| トークン名（案） | サイズ | ウェイト                    | 用途                                                                     |
| ---------------- | ------ | --------------------------- | ------------------------------------------------------------------------ |
| `text-xs`        | 12px   | Regular 400                 | 文字数ヒント、フッターのコピーライト、「自動でダウンロードされます」注記 |
| `text-xs-medium` | 13px   | Semi Bold 600 / Regular 400 | 言語切り替えの言語名（600）、APIエラーバナーの説明文（400）              |
| `text-sm`        | 14px   | Regular 400 / Semi Bold 600 | Introの説明文（400）、フィールドラベル・APIエラーバナーの見出し（600）   |
| `text-base`      | 15px   | Regular 400 / Semi Bold 600 | カラーコード・Selectの表示値（400）、「トップページへ戻る」ボタン（600） |
| `text-md`        | 16px   | Semi Bold 600               | 画像生成ボタンの文言                                                     |
| `text-lg`        | 18px   | Semi Bold 600               | Selectの展開シェブロン                                                   |
| `text-xl`        | 19px   | Bold 700                    | ロゴのシンボル文字「M」                                                  |
| `text-2xl`       | 22px   | Bold 700                    | ロゴのワードマーク「mojica」                                             |
| `text-3xl`       | 24px   | Bold 700                    | Introの見出し（Mobile）                                                  |
| `text-4xl`       | 26px   | Bold 700                    | 404画面の「ページが見つかりません」                                      |
| `text-5xl`       | 30px   | Bold 700                    | Introの見出し（Desktop/Tablet）                                          |
| `text-7xl`       | 72px   | Bold 700                    | 404画面の「404」                                                         |

`text-3xl`（Mobile）と`text-5xl`（Desktop/Tablet）は同じ見出し要素（Intro見出し）のレスポンシブ差分であり、別トークンではなくブレークポイントごとの上書きとして扱う。

---

# 3. 角丸（border-radius）

Material Design 3のコンポーネント形状に合わせ、Figma実測値を用途別の固定トークンとして定義する。

| トークン名（確定） | 値   | Material 3での用途                             | 用途                                                              |
| ------------------ | ---- | ---------------------------------------------- | ----------------------------------------------------------------- |
| `radius-sm`        | 8px  | 小さなコンテナ・スウォッチ                     | カラーピッカーの色見本（スウォッチ）                              |
| `radius-md`        | 10px | テキストフィールド・アウトライン系コントロール | 入力欄・Select・言語切り替え・カラーピッカー・APIエラーバナーの枠 |
| `radius-lg`        | 12px | ボタン・主要アクション                         | ロゴのシンボル、画像生成ボタン、404「トップページへ戻る」ボタン   |
| `radius-xl`        | 18px | カード・大きなコンテナ                         | フォームカード                                                    |

---

# 4. スペーシング

Material 3の4dpグリッドをスペーシングの基準とし、Web実装では1dpを1pxとして扱う。padding・gap・要素の高さは原則として4pxの倍数を使用する。Figmaの実測値が4pxの倍数でない場合は、その値を直接採用せず、視覚的な関係を維持できる最も近い基準値へ調整する。

## 主な値の一覧

`4, 8, 12, 16, 20, 24, 28, 32, 48, 56`

## 主要要素のpadding/gap/height

| 要素                              | padding                                                           | gap                                    | height（Desktop）                      |
| --------------------------------- | ----------------------------------------------------------------- | -------------------------------------- | -------------------------------------- |
| ヘッダー                          | `0px 56px`（Desktop）／`0px 32px`（Tablet）／`0px 20px`（Mobile） | -                                      | 88px（Desktop/Tablet）／72px（Mobile） |
| ロゴ（アイコン+文字）             | -                                                                 | 12px                                   | -                                      |
| メインコンテンツ                  | `48px 0px 0px`（Desktop/Tablet）／`32px 0px 0px`（Mobile）        | 28px（Desktop/Tablet）／24px（Mobile） | -                                      |
| Intro（見出し+説明文）            | -                                                                 | 12px                                   | -                                      |
| フォームカード                    | 32px（Desktop）／`24px 20px`（Mobile）                            | 24px                                   | -                                      |
| フィールド1組（label/input/hint） | -                                                                 | 8px                                    | -                                      |
| TextFieldの入力欄                 | `0px 16px`                                                        | 8px                                    | 48px                                   |
| カラーピッカー                    | `0px 16px 0px 12px`                                               | 12px                                   | 56px                                   |
| カラー見本（スウォッチ）          | -                                                                 | -                                      | 36×36                                  |
| Select                            | `0px 16px`                                                        | -                                      | 48px                                   |
| 画像生成ボタン                    | -                                                                 | -                                      | 56px                                   |
| APIエラーバナー                   | `16px`                                                            | 8px                                    | -                                      |
| フッター                          | -                                                                 | -                                      | 80px                                   |

---

# 5. 効果（Effects）

| トークン名（案） | 値                                     | 用途               |
| ---------------- | -------------------------------------- | ------------------ |
| `shadow-card`    | `0px 8px 24px 0px rgba(0, 0, 0, 0.08)` | フォームカードの影 |

---

# 6. ブレークポイント

| 名称    | 幅     | 対応するFigmaフレーム              |
| ------- | ------ | ---------------------------------- |
| Mobile  | 390px  | `mojica / Mobile / JA`             |
| Tablet  | 768px  | `mojica / Tablet / JA`             |
| Desktop | 1440px | `mojica / Default`、`Desktop / EN` |

Material 3のコンパクト・ミディアム・エクスパンデッドの画面幅区分を参照し、既存のMobile・Tablet・Desktopを対応付ける（ui.md §14参照）。

---

# 7. CSSカスタムプロパティへの反映方針

色・タイポグラフィ・角丸・効果はCSSカスタムプロパティとして`:root`に定義し、Reactコンポーネントから参照する。

```css
/* src/index.css */
:root {
  --md-sys-color-background: oklch(0.978 0.008 225.1);
  --md-sys-color-surface: oklch(1 0 0);
  --md-sys-color-on-surface: oklch(0.229 0.006 56.1);
  --md-sys-color-on-surface-variant: oklch(0.505 0.015 63.7);
  --md-sys-color-outline: oklch(0.872 0.017 79.3);
  --md-sys-color-primary: oklch(0.793 0.088 227.9);
  --md-sys-color-on-primary: oklch(0.33 0.046 228.2);
  --md-sys-color-error: oklch(0.541 0.194 26.7);
  --md-sys-color-error-container: oklch(0.971 0.014 17.4);
  --md-sys-color-on-error-container: oklch(0.541 0.194 26.7);
  --md-sys-color-surface-container: oklch(1 0 0);
  --md-sys-color-outline-variant: oklch(0.881 0.038 224.3);

  --md-sys-typescale-body-small-size: 12px;
  --md-sys-typescale-body-medium-size: 14px;
  --md-sys-typescale-body-large-size: 15px;
  --md-sys-typescale-label-large-size: 16px;
  --md-sys-typescale-title-large-size: 22px;
  --md-sys-typescale-headline-small-size: 24px;
  --md-sys-typescale-display-large-size: 72px;

  --md-sys-shape-corner-small: 8px;
  --md-sys-shape-corner-medium: 10px;
  --md-sys-shape-corner-large: 12px;
  --md-sys-shape-corner-extra-large: 18px;
  --md-sys-elevation-level-1: 0px 8px 24px 0px rgba(0, 0, 0, 0.08);
}

/* 既存コンポーネント向けの別名 */
:root {
  --radius-sm: 8px;
  --radius-md: 10px;
  --radius-lg: 12px;
  --radius-xl: 18px;

  --shadow-card: 0px 8px 24px 0px rgba(0, 0, 0, 0.08);

  --text-xs: 12px;
  --text-sm: 14px;
  --text-base: 15px;
  --text-md: 16px;
  --text-lg: 18px;
  --text-xl: 19px;
  --text-2xl: 22px;
  --text-3xl: 24px;
  --text-4xl: 26px;
  --text-5xl: 30px;
  --text-7xl: 72px;
}

:root {
  --background: oklch(0.978 0.008 225.1);
  --surface: oklch(1 0 0);
  --foreground: oklch(0.229 0.006 56.1);
  --muted-foreground: oklch(0.505 0.015 63.7);
  --helper-foreground: oklch(0.502 0.044 228.7);
  --border: oklch(0.872 0.017 79.3);
  --border-accent: oklch(0.881 0.038 224.3);
  --primary: oklch(0.793 0.088 227.9);
  --primary-foreground: oklch(0.33 0.046 228.2);
  --inverse: oklch(0.24 0.006 78.2);
  --inverse-foreground: oklch(1 0 0);
  --destructive: oklch(0.541 0.194 26.7);
  --destructive-background: oklch(0.971 0.014 17.4);
  --destructive-border: oklch(0.74 0.116 20.2);
}
```

Material 3の役割名を正とし、既存コンポーネントの移行期間は下段の別名を使用する。

---

# 8. 各コンポーネントとの対応

| コンポーネント                                                                                                                                     | 色                                                              | 角丸                                         | 主なタイポグラフィ                                    |
| -------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- | -------------------------------------------- | ----------------------------------------------------- |
| [Layout](./components/Layout.md)（ページ全体）                                                                                                     | `background`                                                    | -                                            | -                                                     |
| [AppHeader](./components/AppHeader.md)・[AppFooter](./components/AppFooter.md)・[ImageGenerationForm](./components/ImageGenerationForm.md)のカード | `surface`                                                       | `radius-xl`（カードのみ）                    | -                                                     |
| [Logo](./components/Logo.md)                                                                                                                       | `inverse` / `inverse-foreground`                                | `radius-lg`（シンボル）                      | `text-xl`（シンボル文字）／`text-2xl`（ワードマーク） |
| [TextField](./components/TextField.md)                                                                                                             | `border` / `border-accent`                                      | `radius-md`                                  | `text-sm`（ラベル）／`text-xs`（ヒント）              |
| [ColorPickerField](./components/ColorPickerField.md)                                                                                               | `border-accent`                                                 | `radius-md`（欄）／`radius-sm`（スウォッチ） | `text-base`（HEX値）                                  |
| [LanguageSwitcher](./components/LanguageSwitcher.md)                                                                                               | `border`                                                        | `radius-md`                                  | `text-xs-medium`                                      |
| [FieldError](./components/FieldError.md)                                                                                                           | `destructive`                                                   | -                                            | `text-xs`                                             |
| [ApiErrorBanner](./components/ApiErrorBanner.md)                                                                                                   | `destructive-background` / `destructive-border` / `destructive` | `radius-md`                                  | `text-sm`（見出し）／`text-xs-medium`（説明文）       |
| [GenerateButton](./components/GenerateButton.md)（通常時）                                                                                         | `primary` / `primary-foreground`                                | `radius-lg`                                  | `text-md`                                             |
| [GenerateButton](./components/GenerateButton.md)（Retryableバリアント）、404の「トップページへ戻る」ボタン                                         | `inverse` / `inverse-foreground`                                | `radius-lg`                                  | `text-md`／`text-base`                                |
| [ImageTypeSelect](./components/ImageTypeSelect.md)                                                                                                 | `border-accent`                                                 | `radius-md`                                  | `text-base`                                           |
| [NotFoundView](./components/NotFoundView.md)                                                                                                       | `foreground`（見出し）／`muted-foreground`（説明文）            | -                                            | `text-7xl`（404）／`text-4xl`（見出し）               |

---

# 9. 残存事項

- `ring`（フォーカスリング色）はFigma上に定義がなく未抽出であり、実装時に決定する
- `radius-sm`/`radius-md`/`radius-xl`はMaterial 3のシェイプ役割へ移行するまで既存コンポーネント向けの別名として維持する（§3参照）
- ダークモードのトークンはFigma上に定義がなく、本書の対象外（MVPではライトモードのみ）
- Material 3のコンパクト・ミディアム・エクスパンデッドとMobile/Tablet/Desktopの境界値は、実装時に画面要件と合わせて確定する

## 参考

- [Material Design 3](https://m3.material.io/)
- [Material 3: States](https://m3.material.io/foundations/interaction/states/overview)
- [Material 3: Canonical layout examples](https://m3.material.io/foundations/layout/canonical-examples/overview)
