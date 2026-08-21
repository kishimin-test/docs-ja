# mojica フロントエンド ブランチ設計書

本書は、mojicaのフロントエンド開発におけるGitブランチの命名、ライフサイクル、Pull Request、CIおよび共有ブランチ保護の設計を定義する。

---

# 1. 目的

- 変更の目的をブランチ名から判別できるようにする。
- 1つのブランチへ無関係な変更が混在することを防ぐ。
- `main`を常にレビュー済みかつFrontend CIを通過した状態に保つ。
- マージ済みの履歴を後から書き換えず、変更の追跡可能性を維持する。

---

# 2. ブランチ構成

## 2.1 共有ブランチ

| ブランチ | 役割 | 直接コミット・直接push |
| --- | --- | --- |
| `main` | リリース可能な統合済みコードを保持する | 禁止 |

`develop`、`release/*`、`frontend/*`のような長期ブランチは設けない。フロントエンドの変更もバックエンドや文書の変更と同じ`main`へ統合する。

## 2.2 作業ブランチ

作業ブランチは`main`から作成し、1つの目的と1つのPull Requestだけに使用する。マージ後は削除し、後続作業へ再利用しない。

```text
main
 └─ <type>/<short-description>
     └─ Pull Request
         └─ main
```

---

# 3. 命名規則

作業ブランチ名は次の形式とする。

```text
{type}/{short-description}
```

`short-description`は英語のkebab-caseで記述し、変更対象ではなく変更の目的を簡潔に表す。

| type | 用途 | 例 |
| --- | --- | --- |
| `feat` | ユーザーへ新しい振る舞いを提供する | `feat/add-language-switcher` |
| `fix` | 不具合を修正する | `fix/prevent-form-double-submit` |
| `refactor` | 外部の振る舞いを変えず構造を改善する | `refactor/extract-image-form` |
| `docs` | 設計書や利用手順だけを変更する | `docs/define-frontend-branch-strategy` |
| `test` | テストだけを追加・修正する | `test/add-image-form-cases` |
| `chore` | 依存関係、ツール、設定などを保守する | `chore/update-vite` |

`feature/*`は使用せず、Conventional Commitsのtypeと揃えた`feat/*`を使用する。`frontend/feat/*`のように技術領域を先頭へ追加しない。フロントエンドであることはPull Requestの変更ファイル、タイトル、ラベルから判別する。

Issueと関連付ける場合も、ブランチ名へIssue番号を必須とはしない。追跡情報はPull Request本文に記録する。

---

# 4. 作業フロー

## 4.1 作成

作業開始時に`main`を基点として新しいブランチを作成する。

```bash
git switch main
git switch -c feat/add-language-switcher
```

作業ブランチのupstreamに`main`を設定してはならない。push時は同名のリモート作業ブランチを指定する。

```bash
git push -u origin feat/add-language-switcher
```

## 4.2 変更範囲

1ブランチでは1つの目的だけを扱う。作業中に独立した変更が必要になった場合は、別の作業ブランチとPull Requestへ分離する。

次の変更を同じブランチへ混在させない。

- 機能追加と無関係なリファクタリング
- 不具合修正と依存関係の一括更新
- フロントエンド変更と無関係なバックエンド変更
- 設計書の更新と、その設計に関係しない文書修正

仕様、設計書、実装、テストが同じ目的を表す場合は、同一ブランチに含めてよい。

## 4.3 Pull Request

作業ブランチから`main`へPull Requestを作成し、次を確認してからマージする。

- 変更目的と範囲がPull Request本文に記載されている。
- 関連する仕様、設計書、Issue、ADRが明記されている。
- レビューが完了している。
- 必須のFrontend CIがすべて成功している。
- unresolvedなレビュー指摘が残っていない。

## 4.4 マージ後

- リモートとローカルの作業ブランチを削除する。
- マージ済みブランチへ追加コミットをpushしない。
- マージ後に修正が必要になった場合は、更新後の`main`から新しい作業ブランチを作成し、新しいPull Requestで反映する。

---

# 5. Pull Requestとコミットの単位

ブランチ、Pull Request、変更目的は1対1に対応させる。

```text
1つの変更目的
  = 1つの作業ブランチ
  = 1つのPull Request
```

コミットはレビュー可能な単位に分ける。コミットメッセージはConventional Commits形式を使い、Summaryで変更内容、Bodyで変更理由を英語で説明する。

マージ方法はリポジトリで設定された方式に従う。共有済みのコミットに対する`rebase`、`amend`、`squash`は行わない。Pull Request作成前の未共有な作業ブランチでは、履歴を整理するために実行してよい。

---

# 6. CI品質ゲート

`main`へマージするPull Requestでは、path filterを持たず常に生成されるFrontend CIの次のチェックを必須とする。

| 必須チェック | 検証対象 |
| --- | --- |
| `install` | lockfileに基づいて依存関係を再現できること |
| `check` | 型チェック、Lintなどの静的検査 |
| `unit-test` | フロントエンドの自動テスト |

必須チェック名を推測で追加しない。GitHub Actionsで実際に生成されるcheck名を確認し、ワークフロー側で名前を変更した場合はRulesetとの一致を再確認する。

path filterによりPull Requestの変更内容次第で生成されないチェックは、fallback jobなどで常時生成を保証しない限り必須チェックに設定しない。

---

# 7. 共有ブランチの保護

`main`はGitHubのActiveなRulesetを最終防壁として、次の規則を強制する。

- Pull Requestを必須にする。
- `install`、`check`、`unit-test`の成功を必須にする。
- ブランチ削除を禁止する。
- non-fast-forward更新とforce pushを禁止する。
- 通常運用のbypass actorを設定しない。

ローカルの`pre-push` hookは、`main`宛ての誤pushを送信前に検出する補助防壁として使用する。hookはcloneごとに設定が必要であり、サーバー側Rulesetの代替にはしない。

---

# 8. 禁止事項

- `main`へ直接commitまたはpushする。
- `HEAD:main`などの明示refspecでPull Requestを迂回する。
- `main`または共有済みブランチへforce pushする。
- 作業ブランチのupstreamを`origin/main`にする。
- 1つの作業ブランチを複数のPull Requestで再利用する。
- マージ済みの作業ブランチへ後続修正を追加する。
- CIを通す目的でテストを無効化または削除する。
- 通常作業を早める目的でRulesetのbypassを使用する。

---

# 9. 例外と緊急対応

通常の開発では例外を設けない。障害復旧などでRulesetの一時変更が必要になった場合も、担当者が単独でbypassせず、変更理由、承認者、実施内容、復旧確認を追跡可能な記録へ残す。

緊急時の権限者、承認経路、Rulesetの復旧手順は現時点で未定義である。これらが決定するまでは、通常のPull Request経路を使用する。

---

# 10. 運用確認

ブランチ運用またはCI設定を変更した場合は、次を確認する。

- 現在の作業ブランチが`main`ではない。
- 作業ブランチのupstreamが同名のリモートブランチを指している。
- `main`を対象とするRulesetがActiveである。
- 必須チェックが対象Pull Requestで実際に生成される。
- `main`宛ての直接pushがサーバーで拒否される。
- ローカルhookが`main`宛てを拒否し、作業ブランチ宛てを許可する。

---

# 11. 関連資料

- [UI設計書](./ui.md)
- [フロントエンドアーキテクチャ設計書](./frontend-architecture.md)
- ADR-0021「共有ブランチはPull Request経由で更新する」
- ADR-0030「共有ブランチの直接pushをサーバー側Rulesetとローカルhookで防止する」
- ADR-0035「マージ済みPRへのレビュー指摘は新しいブランチとPRで反映する」
- ADR-0042「必須status checkはpath filterを持たないCIワークフローに限定する」
