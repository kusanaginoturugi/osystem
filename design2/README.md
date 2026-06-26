# osystem design2

`design/base.css` の汎用デザインベースを、**トークン / プリミティブ / アプリ固有** の 3 層に分割した次バージョン。

見た目は `design/sample.html` と等価。値の差し替えは `tokens.css` 1 ファイルで完結する。

## ファイル構成

| ファイル | 役割 |
| --- | --- |
| `tokens.css` | 色・角丸・影・フォントの `:root` 変数。差し替えポイント。 |
| `primitives.css` | アプリ非依存の汎用パーツ (reset, card, form, button, table, flash)。`tokens.css` を参照。 |
| `app.css` | osystem 固有の殻 (app-shell, header-nav, masters-sync 系)。`tokens.css` と `primitives.css` に依存。 |
| `sample.html` | 3 ファイルを読み込んだ表示確認用。 |

依存方向は `tokens.css ← primitives.css ← app.css` の一方向。下から上への参照はしない。

## design/ との関係

- `design/` は freeze。既存採用アプリの参照先として残す。
- 新規アプリ / 既存アプリの再装飾は `design2/` を採用する。
- 他プロジェクトに持ち出すなら `tokens.css` + `primitives.css` の 2 つだけでも成立する。

## 採用方法

各アプリの `application.css` 等の先頭から `@import` するか、リンクタグ 3 本を順に読み込む:

```html
<link rel="stylesheet" href=".../tokens.css">
<link rel="stylesheet" href=".../primitives.css">
<link rel="stylesheet" href=".../app.css">  <!-- osystem 系のみ -->
```

osystem 外で使う場合は `app.css` を外し、自前の layout shell を書く。

## 公開

GitHub Pages (リポジトリ root 公開) で
`https://kusanaginoturugi.github.io/osystem/design2/sample.html` から確認できる。

## 既知の TODO

- spacing scale (`--space-*`) のトークン化は未着手。今回は既存の rem リテラルを維持。
- focus outline 用の `--color-primary-focus` は `--color-primary-strong` の 22% 相当。トークン整理時に統合検討。
- body 背景 gradient のリテラル `rgba(183, 166, 217, 0.18)` 等は `tokens.css` 化していない。
