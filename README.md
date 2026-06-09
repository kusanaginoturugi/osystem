# osystem

聖明王院向けに作った業務システム群を置く作業ディレクトリ。各サブディレクトリは独立した git リポジトリ (`github.com/kusanaginoturugi/<name>`)。

## システム一覧

| ディレクトリ | 用途 | スタック | 本番 URL | GitHub repo | 最終コミット |
|---|---|---|---|---|---|
| [`portal/`](./portal) | 全システムへの入口 (ポータル) | 静的 HTML/JS | — | `WisdomKing` | 2026-05-11 |
| [`osystem-masters/`](./osystem-masters) | 共通マスタ (伝道会 / 護摩供 / 商品) の正本 + 管理 UI + 読み取り API | CF Workers + D1 + Hono / authentik OIDC | (未デプロイ) | `osystem-masters` | 2026-06-09 |
| [`dailytally2/`](./dailytally2) | 八大明王護摩供 毎日集計 (現行) | CF Workers + D1 / authentik OIDC | https://dailytally.kusanaginoturugi.workers.dev/ | `dailytally2` | 2026-06-05 |
| [`dedications/`](./dedications) | 八大明王護摩供 代理奉納 (FAX 申込書入力) | Rails 8.1 / Ruby 3.4.8 | https://dedications.showway.biz/ | `dedications` | 2026-05-19 |
| [`liberation/`](./liberation) | 超抜式 挙行登録 (現在は単一聖院モード) | Rails 8.1 / Ruby 3.4.8 | https://liberation.showway.biz/ | `liberation` | 2026-05-08 |
| [`itementry/`](./itementry) | 道具販売登録 (レシート / 帳票 / 商品マスタ) | Rails 8.1 / Ruby 3.4.7 | https://itementry.showway.biz/ | `itementry` | 2026-04-20 |
| [`bulkpurchase/`](./bulkpurchase) | 道具一括注文 (伝道会の月次注文集約) | Rails 8.1 / Ruby 3.4.8 | https://bulkpurchase.showway.biz/ | `bulkpurchase` | 2026-04-30 |
| [`register/`](./register) | テンキーレジ (Web 版 / Python CLI 版) | 静的 HTML / Python 3 | https://register-xju.pages.dev/ | `register` | 2026-05-17 |

`portal/` のカード定義 (`portal/js/app.js`) と上の表は対応している。新システム追加時は両方更新する。

各システムが保持しているマスタデータの所在と源流は [`DATA.md`](./DATA.md) を参照。新システムを作る前に必ず読む。

## 命名規則

- ディレクトリ名は lowercase。
- GitHub リポジトリ名と異なる場合は上の表「GitHub repo」列で対応を示す。
  - `portal/` は GitHub 上では `WisdomKing` リポジトリ。

## 旧版

- `.old/Dailytally/` — `dailytally2/` の旧版 (構造を正規化して作り直す前のもの)。アーカイブとしてのみ残す。

## ローカル運用メモ

- 各リポジトリは個別に clone されている。横断的な作業はここで `for d in */; do ...; done` する。
- Rails 系 4 つは全部 `mise` で Ruby を入れる構成 (`.mise.toml` または `.ruby-version`)。
- CF Workers 系 (`dailytally2`) は `node_modules` で 200M+ あるので du に注意。

## 関連する外部システム

- 認証: `dailytally2` は authentik (OIDC) または SSO ヘッダで権限制御。
- 連携: `dailytally2` は tendo.net への送信、Resend でのメール通知あり。
