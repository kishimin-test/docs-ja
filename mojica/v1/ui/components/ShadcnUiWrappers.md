# shadcn/uiラッパー共通設計書

## 対象

`components/ui/`に配置する、shadcn/ui CLIが生成するprimitiveのラッパーに共通する設計を定義する。個々のラッパーのProps、variant、表示状態、利用箇所はshadcn/uiの公開契約または利用側コンポーネントの設計書で扱い、本書には記載しない。

## 配置と所有境界

- 実装とStoryは`components/ui/`に配置する。
- CLI生成物のファイル名はshadcn/ui標準のkebab-caseを維持する。
- `components.json`のaliasはshadcn/uiの既定値（`ui`: `@/components/ui`、`lib`: `@/lib`）を使用する。
- ラッパー本体はCLIが再生成できるvendorレイヤーとして扱い、手動編集しない。
- アプリ固有の状態、文言、業務判断、複数primitiveの合成は利用側の自前コンポーネントが担う。

## カスタマイズ

- 色は[design-tokens.md](../design-tokens.md)のCSS変数で変更する。
- 個別の見た目は利用側から`className`を渡して調整する。
- Tailwindのスケール全体に関わる変更はプロジェクト共通のテーマ設定で行う。
- 構造や振る舞いを追加する必要がある場合は、`components/ui/`を変更せず、自前コンポーネントで合成する。

## Storybookとテスト

- 各ラッパーに`*.stories.tsx`をcolocateし、shadcn/uiが公開するvariantと状態をStoryへ反映する。
- StoryファイルはCLI生成物ではないため、`shadcn add --overwrite`の対象外とする。
- ラッパー専用の`*.small.test.tsx`は作成せず、Storybookのテストと利用側コンポーネントのテストで契約を確認する。
- Radix UIまたはshadcn/uiが保証する内部実装を重複してテストしない。

## アクセシビリティ

Radix UIを基盤とするキーボード操作、フォーカス管理、ARIA属性は標準実装を利用する。利用側で追加するラベル、説明、エラー、ローディング状態との関連付けは、自前コンポーネントの責務として個別の設計書に定義する。
