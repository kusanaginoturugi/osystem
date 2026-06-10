# design — 共通デザイン / CSS ベース

osystem 配下のアプリ群で「同じ見た目に揃えられるところは揃えたい」というだけのための置き場。
追跡は [`osystem#1`](https://github.com/kusanaginoturugi/osystem/issues/1)。

## 何が source of truth か

- `base.css` がこのベースの正本。
- 元は `dedications/app/assets/stylesheets/application.css` の汎用層。
  dedications のスタイルが好みなので、ここから「アプリ固有じゃない部分」だけ抜き出した。
- `sample.html` は `base.css` を素のまま読み込んで主要部品を一通り並べた確認用ページ。
  ブラウザで開いて崩れていないかを目で見る用途。

## 配布方式

当面はコピー。npm / submodule / CDN は使わない。

理由:
- 対象アプリは 3〜4 個で増減しない。
- 各アプリは独立リポジトリ + 個別デプロイ (Rails 8 / CF Workers)。
- CSS は数百行〜千行台で、自動同期しないと困るほどではない。
- 各アプリで「ここだけは合わない」ということが普通にあるので、上書き前提のほうが現実的。

採用したいアプリは、自分の `app/assets/stylesheets/application.css` を
`base.css` で置き換えるか、必要箇所をマージする。完全に同期する必要はない。

## ベースに入れたもの / 入れないもの

入れた (汎用):

- テーマトークン (`:root` の色)
- リセット (`body`, `a`, form 要素の `font` 統一)
- 骨格 (`.app-shell`, `.app-header`, `.page-head`, `.eyebrow`, `.section-lead`, `.page-actions`)
- ヘッダーメニュー (`.menu-tree*`, `.menu-tree-panel`, `.menu-sub-items`)
- カード (`.card`, `.narrow-card`)
- フォーム部品 (`.stack-form`, `.field`, `.field-label`, `.checkbox-field`, `.toggle-field`, `.toggle-label`, `.derived-value`, `.form-actions`, `.form-two-columns`)
- ボタン / リンク (`.primary-link`, `.secondary-link`, `.primary-button`, `.danger-button`, `.ghost-button`, `.search-result`)
- フラッシュ (`.flash`, `.error-box`)
- テーブル (`.table-wrap`, `table`, `th`, `td`, `.sort-link`, `.clickable-row`, `.text-right`)
- マスタ連携 UI 一式 (`.masters-sync*`, `.fellowship-enabled-list`, `.fellowship-enabled-item`, `.code-mono`)
- 768px 未満のモバイル調整 (骨格 + `.stack-on-mobile`)

入れない (dedications 固有):

- FAX 注文書の升目 (`.sheet-*`)
- 注文フォーム (`.order-form-*`, `.compact-field`, `.fellowship-field`)
- 支払い / 申込種別の選択 (`.payment-*`, `.form-type-*`)
- 集計レポート (`.report-table-compact`, `.dedication-counts-*`, `.counts-sheet-*`, `.summary-*`, `.personal-summary-*`, `.order-group*`)
- 一覧テーブルのカラム幅 (`.orders-*-column`, `.summary-*-column`)
- 申込書専用の表示 (`.serial-gap-warning`, `.status-unpaid`, `.numeric-input`, `.detail-grid`, `.delete-order-form`)
- 印刷用 `@media print` (用紙レイアウト前提なので)

上記は dedications 側に残す。誰も使ってないことが確認できれば削るかもだが、当面は触らない。

## 各アプリの現状

| アプリ | スタイル基盤 | 寄せ方 | 状況 |
|---|---|---|---|
| `dedications` | 手書き `application.css` (1203 行) | ベースの抽出元 | 触らない。汎用層を再切り出しする場合は別判断。 |
| `liberation` | 手書き `application.css` (694 行) | `:root` トークン + ハードコード rgba 置換 | 寄せ済み (2026-06-10)。命名は BEM 風のまま、view は touch していない。 |
| `bulkpurchase` | Tailwind v4 (`tailwindcss-rails`) + Propshaft | `@theme` で stone / amber / emerald / rose / red / blue / gray を上書き | 寄せ済み (2026-06-10)。`current_orders/_form` のみ `.btn`/`.btn-primary`/`.btn-secondary` に @apply 集約済み、他画面の utility 連打は順次。 |
| `itementry` | Bootstrap 5 CDN + 手書き `application.css` | `--bs-*` 変数と `.btn-*` / `.alert-*` / `.table-*` の component 独自変数を `:root` で上書き | 寄せ済み (2026-06-10)。view の Bootstrap クラス名は touch していない。 |
| `dailytally2` | CF Workers + 素 HTML + `public/style.css` (752 行) | `:root` トークン + zinc / red ハードコード置換 | 寄せ済み (2026-06-10)。警告の黄系 (`#fffbe6` 等) は意味があるので維持。 |
| `osystem-masters` | Hono JSX + Layout 内 `<style>{STYLE}</style>` | `STYLE` テンプレ文字列の色値を直接置換 | 寄せ済み (2026-06-10)。inline `style=` は元々無し。本番は未デプロイ。 |
| `register` | スタンドアロン HTML | — | 触らない (osystem#1 のスコープ外、ユーザー指示)。 |

`portal/` は UI なし、`.old/Dailytally/` はアーカイブ。

「寄せ済み」は「`base.css` の色味と一致する見た目に近づけた」という意味で、構造 (HTML/クラス名) を base.css 命名に揃えたわけではない。完全統一は別タスク。

## 試し方

```sh
$ cd osystem/design
$ python3 -m http.server 8000
# http://localhost:8000/sample.html
```

または `firefox sample.html` でも見られる。

## 変更ルール

- `base.css` を変えるときは `sample.html` で崩れていないか目視。
- 「dedications のここを直したい」が動機なら、まず dedications 側の application.css を直す。
  そのうえで「これは汎用層に戻したほうがいいな」と判断したものだけ `base.css` にも反映。
  逆方向 (ベースだけ先に直して dedications には伝播しない) は避ける。同期が崩れる。
- アプリ側で「ベースの定義を上書きする」のは OK。後勝ちにしておけば問題ない。
