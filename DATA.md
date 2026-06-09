# DATA — マスタデータの所在と源流

`osystem/` 配下の各システムが持っているマスタデータと、その源流（source of truth）を整理したもの。新システム作成時はこれを見て、自前で持つか共有先を参照するかを判断する。

**源流**: [`osystem-masters`](https://github.com/kusanaginoturugi/osystem-masters) (Cloudflare Workers + D1)

更新ルール：
1. `osystem-masters` の管理画面 (`/admin`) で更新する (authentik OIDC、admin グループのみ)。
2. 各消費アプリの「マスタ同期」ボタンで自分の D1 / DB に取り込む (pull 型)。

`osystem-masters` の管理 UI で編集すると、その場で読み取り API (`GET /api/...`) にも反映される。消費アプリ側は同期ボタンを押されるまで古い値のまま運用される。

---

## 1. 伝道会マスタ

**源流**: `osystem-masters.fellowships`

`id INTEGER PK AUTOINCREMENT` + `code TEXT UNIQUE NOT NULL`。`id` は不変、`code` はユーザー編集可。論理削除のみ (`active = 0`)。

| カラム | 内容 |
|---|---|
| `id` | 不変の内部キー。消費アプリ側の FK もこれを参照する想定。 |
| `code` | 5 桁文字列。例: `31303` 大江戸、`32204` 富士山。tendo.net への送信にも使う。 |
| `old_code` | 旧コード (4 桁)。`dedications` で歴史データとの突合に使う。 |
| `name` | 正式名。例: `大江戸準総壇`。 |
| `short_name` | 短縮名。例: `大江戸`。一覧表示や入力 UI で使う。 |
| `active` | `1`=有効、`0`=論理削除。 |
| `sort_order` | 一覧表示順。 |

初期投入は 94 件。元データは `dedications/資料/伝道会番号.csv` (4 列: `新番号,旧番号,名称,短縮名`)。CSV は履歴と初期投入用としてリポジトリに残すが、以後は `osystem-masters` 側で編集する。

### 消費アプリの状態

| システム | テーブル | 同期実装 | 備考 |
|---|---|---|---|
| `dailytally2` | `fellowships` | 済 | id (1〜9) → master id (15〜28) リマップ + `enabled` 列で集計対象トグル |
| `bulkpurchase` | `fellowships` (旧 `organizations`) | 済 | id 統合 + `enabled` 列 + `organizations` → `fellowships` rename 完了 |
| `liberation` | `fellowships` (旧 `evangelism_meetings`) | 済 | id 統合 + `enabled` 列。`color_code` は master に集約、`region_id` / `display_order` は liberation 側に保持 |
| `dedications` | `fellowships` (旧 `congregations`) | 済 | id 統合 (89 件すべて master と一致) + `enabled` 列。検索 API は JSON のまま、管理 UI は HTML として `/fellowships` に追加 |

---

## 2. 商品マスタ

**源流**: `osystem-masters.items` (初期投入は未実施。`itementry` と `bulkpurchase` のコード突合が完了したら seed migration を投入する別タスク)

| カラム | 内容 |
|---|---|
| `id` | 不変の内部キー |
| `code` | 商品コード |
| `name` | 名称 |
| `value` | 価格 |
| `refund` | 還付額 |
| `unit` | 単位 (任意) |
| `category` | カテゴリ (任意) |
| `active` | 論理削除フラグ |
| `sort_order` | 表示順 |

### 既存の商品マスタ

| システム | テーブル / ファイル | 同期 | 備考 |
|---|---|---|---|
| `itementry` | `items` (Rake: `db/items3.csv`) | TODO | `item_code` の体系を `bulkpurchase` と突合してから集約 |
| `bulkpurchase` | `items` | TODO | `code` 体系を `itementry` と突合してから集約 |
| `register` | ブラウザ localStorage | 当面ノータッチ | 完全スタンドアロン運用 |

---

## 3. 護摩供マスタ

**源流**: `osystem-masters.ceremonies` (9 件、ローマ字 code 付与)

| code | name |
|---|---|
| `hachidai` | 八大明王護摩供 |
| `daigen` | 大元地空護摩供 |
| `jizo` | 地蔵尊王護摩供 |
| `segaki` | 施餓鬼供養護摩供 |
| `hokuto` | 北斗鎮圧護摩供 |
| `rokuzon` | 禄存宝珠護摩供 |
| `chosei` | 長生南十字星護摩供 |
| `myozen` | 妙善閻魔天王護摩供 |
| `chinkon` | 鎮魂四海龍王護摩供 |

### 消費アプリの状態

| システム | テーブル | 同期 | 備考 |
|---|---|---|---|
| `dailytally2` | `ceremonies` | TODO | 既存 id (1〜9) を master の id (1〜9) にリマップ。`tally_items.ceremony_id` の参照も更新 |
| `dedications` | `events` | TODO | 八大明王のみ使用（代理奉納対象） |

---

## 4. 外部コード体系（既に統一済み）

- 聖明王院本部: `99300`
- 各伝道会: `3xxxx` (`osystem-masters.fellowships.code` を正本とする)
- tendo.net への送信 (`dailytally2`) もこのコードを使用

新規でコードが必要な場面では同じ体系を踏襲する。

---

## 5. 認証

| システム | 方式 |
|---|---|
| `osystem-masters` | authentik OIDC (admin グループのみ編集可) |
| `dailytally2` | authentik OIDC / SSO ヘッダ |
| `dedications` | Rails 個別 users |
| `liberation` | Rails 個別 users |
| `itementry` | (要確認) |
| `bulkpurchase` | Rails 個別 users |

新規 CF Workers 系は authentik OIDC に揃える方針。Rails 4 つの統合は別途検討。

---

## 6. アーキテクチャ

- `osystem-masters` は読み取り API (`GET /api/fellowships` 他) と管理 UI を持つ。
- 消費アプリは pull 型で同期する。管理画面に「マスタ同期」ボタンを置き、`INSERT OR REPLACE` で自分の DB に取り込む。
- CF Workers 系も同アカウントの D1 binding ではなく HTTP 経由で同期する。理由: D1 binding だと cross-DB JOIN ができないため、結局アプリ側にコピーが必要になる。
- 物理削除は禁止。`active = 0` の論理削除のみ。同期で削除済みレコードを誤って復活させないため。
- 同期キーは `id` (master 側の不変 id)。`code` ではない。`code` は人間の都合で変更されうるため。

詳細は [`osystem-masters/README.md`](https://github.com/kusanaginoturugi/osystem-masters)。
