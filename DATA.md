# DATA — マスタデータの所在と源流

`osystem/` 配下の各システムが持っているマスタデータと、その源流（source of truth）を整理したもの。新システム作成時はこれを見て、自前で持つか共有先を参照するかを判断する。

更新ルール：マスタの内容を変えたときは、まず **源流** を更新し、続いて各システムへ反映する。

---

## 1. 伝道会マスタ

同じ 9 つを 4 システムが別スキーマで保持。コード体系 (`tendo_code` / `code`) は全システムで共通 (`31303` 大江戸、`32204` 富士山 …)。

| システム | テーブル / ファイル | 主なカラム |
|---|---|---|
| `dedications` | `congregations` / `資料/伝道会番号.csv` | `code`, `name`, `old_code` |
| `dailytally2` | `fellowships` | `name`, `tendo_code`, `sort_order` |
| `liberation` | `evangelism_meetings` (seed: `config/meetings.yml`) | `name`, `color_code`, `region_id`, `display_order`, `active` |
| `bulkpurchase` | `organizations` (seed: `db/seeds.rb` 内ハードコード) | `name`, `code`, `active` |

**源流**: `dedications/資料/伝道会番号.csv` (4 列: `新番号,旧番号,名称,短縮名`)

- コード: `新番号` (5 桁文字列)。`dailytally2.fellowships.tendo_code` / `bulkpurchase.organizations.code` と一致。
- 正式名: `名称` (例: `大江戸準総壇`)。`dedications.congregations.name` で使う。
- 表示名: `短縮名` (例: `大江戸`)。`dailytally2.fellowships.name` / `bulkpurchase.organizations.name` / `liberation.evangelism_meetings.name` で使う。
- `旧番号` は CSV にあるものだけ持つ (`31407 かながわ準総壇` は不明のため空欄)。

**更新フロー**:
1. `dedications/資料/伝道会番号.csv` を更新。
2. `dedications` 側で取り込み。
3. `bulkpurchase/db/seeds.rb`、`liberation/config/meetings.yml`、`dailytally2/migrations/0002_seed.sql` の `fellowships` を手動で反映。
4. 各システムをデプロイ。

**既知の差分**:
- `dailytally2.fellowships` で山梨の `tendo_code` が `NULL` になっている → `31901` を入れて修正する必要あり。

---

## 2. 商品マスタ

| システム | テーブル / ファイル | 用途 | カラム |
|---|---|---|---|
| `itementry` | `items` (Rake: `db/items3.csv`) | 窓口販売 | `item_code`, `name`, `value`, `refund`, `item_type`, `is_variable_value` |
| `bulkpurchase` | `items` | 一括注文 | `code`, `name`, `value`, `refund`, `unit`, `center_category`, `special_handling_type` |
| `register` | ブラウザ localStorage | 窓口レジ | code, name, price |

価格 (`value`) と還付額 (`refund`) は同じ概念で重複。`code` 体系の互換性は要確認。

**源流（未定）**: 候補は `itementry/db/items3.csv`。要確認。

要確認：
- `itementry` と `bulkpurchase` の `code` は同じ商品で同じ値か。
- `register` の商品マスタを Rails 側と同期する仕組みを作るか、独立運用のままにするか。

---

## 3. 護摩供マスタ

| システム | テーブル | 範囲 |
|---|---|---|
| `dailytally2` | `ceremonies` (`migrations/0002_seed.sql`) | 9 種類フル |
| `dedications` | `events` | 八大明王のみ（代理奉納対象） |

**源流（提案）**: `dailytally2/migrations/0002_seed.sql` の `ceremonies` セクション。
- 理由: 9 種類フルで持っているのはここだけ。

---

## 4. 外部コード体系（既に統一済み）

- 聖明王院本部: `99300`
- 各伝道会: `3xxxx` (`dailytally2.fellowships.tendo_code` = `bulkpurchase.organizations.code` = `dedications.congregations.code`)
- tendo.net への送信 (`dailytally2`) もこのコードを使用。

新規でコードが必要な場面では同じ体系を踏襲する。

---

## 5. 認証

| システム | 方式 |
|---|---|
| `dailytally2` | authentik OIDC / SSO ヘッダ |
| `dedications` | Rails 個別 users |
| `liberation` | Rails 個別 users |
| `itementry` | (要確認) |
| `bulkpurchase` | Rails 個別 users |

新規 CF Workers 系は authentik OIDC に揃える方針。Rails 4 つの統合は別途検討。

---

## 今後の方向性（CF Workers + D1 が増える前提）

短期：このドキュメントで源流を明確化し、手動同期で運用する。

中期：`osystem-masters` D1 を新設し、伝道会・護摩供・商品の正本を集約する。

- CF Workers 系は同アカウント内で D1 binding して直接読み取り（HTTP オーバーヘッドなし）。
- 書き込みは管理用 Worker 1 つに集約。
- Rails 系は当面それぞれ自前で持ち、API 経由で読み取る形に段階移行。

詳細設計は次の CF Workers + D1 案件着手前に別途詰める。
