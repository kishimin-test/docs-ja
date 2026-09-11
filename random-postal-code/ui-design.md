# Zipnami UI設計書

## 1. 適用範囲と責任

この文書はZipnami Web・Android MVP UIの正本である。[design.md](./design.md)、Issue [#5](https://github.com/kishimin/random-postal-code/issues/5)〜[#10](https://github.com/kishimin/random-postal-code/issues/10)、[#12](https://github.com/kishimin/random-postal-code/issues/12)〜[#17](https://github.com/kishimin/random-postal-code/issues/17)を具体化する。

UIは表示、client state、local履歴、地図操作、任意の広告領域、Consent表示、アクセシブルなfeedbackを所有する。[api-design.md](./api-design.md) の契約を利用するが再定義しない。

## 2. 体験原則

- 1つの主要操作で郵便番号を生成し、初期表示では自動生成しない。
- 郵便番号と対応する全住所を主要contentとする。
- 地図、広告、Consentは任意機能とし、主要結果を妨げない。
- WebとAndroidは振る舞いを共有し、navigationとcontrolは各platformに合わせる。
- keyboard、screen reader、zoom、大きな文字、reduced motion、第三者サービス障害時も利用できる。
- MVPの表示言語は日本語とする。製品表示labelをAPI型名から生成しない。

## 3. 情報Architecture

### 3.1 Web Routes

| Route | Page | 目的 |
| --- | --- | --- |
| `/` | Generator | 生成、確認、copy、地図、履歴 |
| `/privacy` | Privacy | 自サービス保存、第三者処理、出典、問い合わせの説明 |

PagesのSPA fallbackにより、両routeへの直接accessとreloadを解決する。

### 3.2 Android Routes

| Route | Screen | 目的 |
| --- | --- | --- |
| `/` | Generator | 生成、確認、外部地図、履歴 |
| `/information` | Information | Privacy link、出典、広告開示、アプリ情報 |

Androidの戻る操作でInformationからGeneratorへ戻り、成功済み結果をresetしない。

## 4. 主要状態Model

Generator featureはloading、data、errorの独立booleanではなく排他的状態を所有する。

```ts
type GeneratorState =
  | { status: "idle" }
  | { status: "loading"; previousResult?: PostalCode }
  | { status: "success"; result: PostalCode }
  | { status: "error"; error: UiError; previousResult?: PostalCode };

type UiError =
  | { kind: "offline" }
  | { kind: "service-unavailable"; requestId?: string }
  | { kind: "invalid-response"; requestId?: string }
  | { kind: "unexpected"; requestId?: string };
```

- Idleは説明と生成操作、空の結果領域を表示する。
- Loadingは重複生成だけを無効にし、navigation、直前結果、履歴を利用可能に保つ。
- Successは現在結果を置換し、local履歴へ正確に1回だけ追加する。
- Errorは理解可能な再試行操作を表示し、直前結果があれば保持して履歴へ再追加しない。
- 失敗したrequestは履歴を作らない。

## 5. Web Page設計

### 5.1 Generator Layout

```text
+------------------------------------------------------+
| Header: Zipnami                         Privacy       |
+------------------------------------------------------+
| 説明                                                 |
| [ 郵便番号を生成 ]                                   |
| 状態 / error / retry                                 |
+-----------------------------+------------------------+
| 現在結果                    | 地図                   |
| 100-0001  [Copy]            | 選択住所               |
| 住所一覧                    | 埋込地図 / fallback    |
+-----------------------------+------------------------+
| 最近の履歴（最大20件）                               |
+------------------------------------------------------+
| 広告専用領域                                         |
+------------------------------------------------------+
| 日本郵便出典 | Privacy | Contact                    |
+------------------------------------------------------+
```

`1024px`未満では結果と地図を1 columnに積み、`1024px`以上では2 columnsとする。DOM・読み上げ順では結果を先にする。content containerはfluidで最大`1200px`とし、`320px` viewportで主要contentや操作に横scrollを要求しない。履歴は現在結果より後に置き、過去結果が主要操作と競合しないようにする。

### 5.2 Web Componentsと所有境界

```text
src/
  app/
    routes/                    # Generator・Privacy route composition
  api/                         # API clientとruntime response validation
  features/postal-generator/
    components/                # GenerateAction、CurrentResult、AddressList
    hooks/                     # request lifecycleとfeature orchestration
  features/history/
    components/                # HistoryList、HistoryItem
    storage/                   # version付きbrowser persistence adapter
  features/maps/
    components/                # AddressMap、MapFallback
  features/advertising/        # AdSense slotとConsent境界
  components/ui/               # 表示専用の再利用control
```

共有`ui` componentはnative HTML propsを拡張し、data fetch、navigation、global stateへ依存しない。Feature componentはdomain表示を、route componentはfeature compositionとpage-level navigationを所有する。2つ以上の実利用が同じ表示契約を必要とするまでcomponentを共通化しない。

### 5.3 現在結果

- API・storageでは7桁の正規値を保持し、画面では`NNN-NNNN`と表示する。
- Issue #6で実装前に契約を変更しない限り、copy操作はhyphenなし7桁をcopyする。
- 住所を元データ順ですべて表示し、複数住所を要約して欠落させない。
- 最初の住所を初期map選択とする。
- 他住所の選択はmap対象だけを変え、現在結果や履歴を変えない。
- 全住所へ、完全な住所queryをencodeした明確な名前のGoogle Maps外部linkを設ける。

copy成功はpoliteなstatus regionで通知し、focusを移動しない。copy失敗でも結果は利用可能に保ち、操作付近へ短いerrorを表示する。

### 5.4 Web履歴

各履歴は正規`PostalCode`と全住所を保存し、server IDを持たない。成功結果を先頭へ追加し、重複を保持し、20件を超えた分を末尾から削除する。破損または未対応versionの保存データは、生成を妨げず破棄または移行する。

狭い画面では履歴を郵便番号と代表住所まで折り畳めるが、明示的な展開controlから全住所へ到達できなければならない。展開だけでは現在のmap選択を変えず、その履歴から利用者が地図操作を選んだ場合だけ変更する。

## 6. Android Screen設計

### 6.1 Generator Layout

```text
+----------------------------------+
| App bar: Zipnami      Information|
+----------------------------------+
| Scroll可能content               |
| 説明                             |
| [ 郵便番号を生成 ]               |
| 状態 / error / retry             |
| 現在の郵便番号                   |
| 住所一覧 + 外部地図              |
| 最近の履歴（最大20件）           |
+----------------------------------+
| AdMob banner専用領域             |
+----------------------------------+
```

contentは予約済みbanner領域と独立してscrollする。bannerはcontrolや結果へ重ならない。大画面でも読みやすい行長に制限した1 columnを維持し、Android埋込地図を追加しない。

### 6.2 Androidの振る舞い

- Webと同じGenerator状態遷移と履歴規則を使う。
- 完全な住所queryをplatformの外部URL・intent機構で開く。
- 対応handlerがない場合、結果を失わずerrorを表示する。
- 位置情報、camera、microphone、連絡先権限を要求しない。
- 履歴を端末内だけに保存し、WebやBackendと同期しない。
- 同一session内でInformationへ移動して戻った場合、Generator状態を保持する。

## 7. 地図・広告・Consent

Web地図は結果取得後にlabel付き領域として表示する。生成前は空iframeを出さず領域自体を省略できる。Loading・失敗時はlayout shiftを抑える領域を確保する。地図失敗時は選択住所と外部linkを含むfallbackを表示する。

広告は必要に応じて広告と識別できる専用領域を使う。広告が利用不能、block、unfilledの場合は、SDK契約に従い安全にcollapseするか非interactiveな予約領域を保ち、アプリerrorとして扱わない。Consent失敗時は利用可能な最もprivacy保護的な広告modeを使い、生成を妨げない。

## 8. Accessibility契約

- 1 pageに1つの`h1`を使い、結果、地図、履歴を階層的headingで構成する。
- ARIA代替よりnativeの`button`、`a`、list semanticsを優先する。
- 可視focusを提供し、DOM順と一致する論理的focus順を保つ。
- layoutが許す限り操作targetを縦横`44px`以上とし、WCAG最小spacingを下回らない。
- loading、成功結果、copy feedback、request errorを、未変更contentの反復なしで通知する。
- 生成中は結果領域へ`aria-busy`を設定し、通常の成功contentへ自動focus移動しない。
- 即時操作が必要なerrorだけsummaryへfocusを移し、それ以外は緊急度に応じたlive regionで通知する。
- 各住所の地図操作へ住所contextを含む一意なaccessible nameを付ける。
- browser zoom 200%、文字拡大、画面回転、reduced motionへ対応する。
- 選択、loading、成功、error、広告状態を色だけで伝えない。

Android controlもReact Native accessibility propsを通じて同等のlabel、role、disabled、selected状態を公開する。

## 9. Visual・Content契約

- 既存Zipnami logo・faviconを使い、装飾画像へ重複する代替textを付けない。
- 郵便番号を視覚的に強調し、fontが対応する場合はtabular numeralを使う。
- `4px`単位のspacing scaleを使い、主要操作、結果、履歴、広告の間を明確に分ける。
- 対応する全themeでtextとcontrolの十分なcontrastを保つ。
- 意味をanimationだけへ依存させず、装飾motionはreduced-motion設定に従う。
- 日本語表示文字列はAPI modelではなくUI所有resourceに置き、MVPでは言語切替を表示しない。

正確なcolor、typography、elevation tokenはUI基盤実装時に固定し、この文書の情報階層を変えずに記録する。

## 10. Responsive検証

- 狭いphone: `320px`、`390px`
- Tablet: `768px`
- Desktop: `1024px`、`1440px`
- browser zoom 200%
- Android phoneと代表的な大画面emulator
- 対応するportrait・landscape

テストは正確なpixelではなくcontent利用可能性と状態の振る舞いをassertする。visual token固定後、安定した代表状態をvisual regression baselineと比較する。

## 11. テスト契約

- Small: 純粋なformat、state reducer、履歴保持、storage validation、URL生成、実serviceを使わないcomponent behavior
- Medium: route navigation、API client境界、browser persistence、制御済み代替を使うmap・ad adapter、Android外部link adapter
- Large: 代表的なPagesからWorkers、AndroidからWorkersのjourneyと、実広告操作を伴わないproduction相当SDK設定
- 利用者操作テストは孤立したDOM event発火ではなくuser-level interactionを使う。
- route遷移テストはtest専用routeではなくproduction route構成を使う。
- 安定したWeb状態を自動accessibility検査し、keyboard、screen reader、zoom、Android手動確認で補う。

Component、Integration、E2Eという名称ではなく実際の依存範囲でtest sizeを分類する。coverage commandが存在する場合、全体summaryの全指標を80%以上とする。

## 12. 未確定の実装詳細

- 正確なdesign tokenとtheme対応
- component libraryを使用するか
- browser・Android persistence adapterとversion key
- 正確な日本語文言と問い合わせ先
- copy値へ表示用hyphenを含めるか
- SDK固有のempty-ad・Consent表示

これらの判断も、上記の状態model、情報順、任意service分離、Accessibility契約を維持する。
