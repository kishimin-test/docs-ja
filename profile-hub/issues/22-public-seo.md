# 公開サイトの SEO 対応 (prerender / メタ情報 / OGP / sitemap) を実装する

| 項目 | 値 |
| ---- | -- |
| Issue | [#22](https://github.com/kishimin-ai-create/profile-hub/issues/22) |
| 英語タイトル | Implement SEO for the public site (prerendering, metadata, OGP, sitemap) |
| マイルストーン | v1 |
| ラベル | enhancement,area:public-web |

---

## 解決したい課題

公開サイトは Vite + React の SPA として構築するため、素の状態では初期 HTML に本文が含まれず `<div id="root"></div>` のみが返る。この状態では以下が起きる。

- クローラーやリンクプレビューが本文を取得できず、検索結果や SNS 共有時に内容が表示されない
- 全ページで `title` と `description` が同一になり、検索結果で記事が区別されない
- 記事を公開しても検索エンジンに発見されるまでの経路がない

自己紹介サイトは検索と SNS 共有からの流入が主な到達経路であるため、これは公開の目的そのものに影響する。

## 提案する解決策

ビルド時の prerender と、ページ単位のメタ情報出力によって SEO 要件を満たす。

### prerender

- ビルド時に各ルートを静的 HTML として出力し、初期 HTML に本文を含める
- 対象 — `/`, `/about`, `/hobby`, `/hobby/:slug` (全公開記事), `/engineering`, `/engineering/:slug` (全公開記事), `/contact` と各言語版
- 記事の追加・公開時に静的 HTML を再生成する手順を決める (ビルド実行のタイミングと自動化の要否)
- prerender した HTML が、JavaScript 有効時のクライアント側描画と矛盾しないこと

### メタ情報

- ページ単位の `title` と `description`
- OGP (`og:title`, `og:description`, `og:type`, `og:url`, `og:image`) と Twitter Card
- 正規 URL (`canonical`)
- i18n Issue と連携した `hreflang`
- 記事詳細ページの構造化データ (JSON-LD)

### 発見性

- `sitemap.xml` の生成。公開済み記事のみを含め、下書きを含めない
- `robots.txt`

## 代替案

- **Next.js の SSG/ISR に移行する** — SEO は最も有利だが、mojica と同一の Vite 前提 Lint / テスト構成を維持できない。SPA + prerender を採用して不採用。
- **サーバー側でクローラーのみに静的 HTML を返す** — クローラー判定の維持コストが高く、判定を誤ると内容が取得されない。全訪問者へ prerender した HTML を返す方式を採用。
- **prerender を行わずメタタグのみ動的に設定する** — 一部のクローラーは JavaScript を実行するが、リンクプレビューの多くは実行しない。不採用。

## 受入条件

- [ ] JavaScript を無効にした状態で各ページを開き、本文が HTML に含まれている
- [ ] ビルド成果物の HTML に、各ページ固有の `title` と `description` が含まれている
- [ ] 記事詳細ページの OGP に、その記事のタイトルと説明が含まれている
- [ ] 各ページに正規 URL が出力されている
- [ ] 日本語版と英語版が `hreflang` で相互に関連付けられている
- [ ] 記事詳細ページに構造化データが出力され、構文エラーがない
- [ ] `sitemap.xml` が生成され、公開済み記事のみが含まれ、下書きが含まれない
- [ ] `robots.txt` が配信され、管理画面がクロール対象外になっている
- [ ] 記事を公開してから静的 HTML に反映されるまでの手順が文書化されている

## 対象外

- 検索順位そのものの改善施策
- アクセス解析の導入

## 補足

- 依存: public-web 基盤Issue、記事APIIssue、i18n Issue
- prerender の手段 (Vite プラグイン / ビルド後のクロール生成など) は本Issue内で選定する
