# D1 統合計画

CF Workers 系がアプリごとに別 D1 を持ち、`osystem-masters` から HTTP でマスタをコピーしている構成をやめ、**1 つの D1 を共有する**構成に移す。

ステータス: **計画のみ・未着手** (2026-10-01)

---

## 1. 現状

| Worker | D1 (`database_name`) | テーブル | マスタの持ち方 |
|---|---|---|---|
| `osystem-masters` | `osystem-masters` | `fellowships`, `ceremonies`, `items`, `master_meta` | 正本 |
| `dailytally2` | `dailytally2` | `fellowships`, `ceremonies`, `tally_items`, `tallies`, `fellowship_targets`, `summary_target_overrides`, `report_settings`, `report_history`, `app_settings` | `fellowships` は HTTP pull (`src/services/master-sync.js`)。id は master と一致済み (0005) だが列名が違う (`name`=`short_name`, `tendo_code`=`code`)。`ceremonies` は未同期・独自 id |
| `nimotsu-bango` | `nimotsu-bango` | `fellowships`, `baggage_records` | 同期なし。独自 id 1〜10、master に無い `茨城` を持つ (`active=0`、レコードは 0003 で埼玉へ移動済み) |
| `.old/Dailytally` | `dailytally` | `app_state` | 旧版。本計画の対象外 |

問題:
- 伝道会マスタが D1 だけで 3 重、Rails の SQLite も入れると 6 重。
- 同期ボタンを押すまで古い値のまま。列名の読み替えやアプリ独自の id がアプリごとにバラバラ。
- `DATA.md` §7 の「D1 binding だと cross-DB JOIN できないから HTTP でコピーする」という前提が、DB を分けたことで生まれた制約になっている。DB を 1 つにすればそもそも発生しない。

---

## 2. 決定事項

| 項目 | 決定 | 理由 |
|---|---|---|
| 共有 DB | 既存の `osystem-masters` D1 (`2a5b1ce7-e309-4291-b28e-34fbada65526`) をそのまま使う | master の id・管理 UI・Rails 向け HTTP API を一切動かさずに済む。業務データの移動は 2 アプリ分だけ |
| DB 名 | `osystem-masters` のまま | wrangler 4.92.0 に `d1 rename` が無い。新 DB を作って全部移すコストに見合わない |
| migration の所有者 | `osystem-masters` repo の `migrations/` だけ | D1 は適用済み migration を `d1_migrations` にファイル名で記録する。複数 repo から同じ DB に `migrations_dir` を向けると番号・順序が衝突する |
| 各アプリの `wrangler.toml` | `database_id` を共有 DB に向け、`migrations_dir` を削除 | アプリ側から migration を流させない |
| migration ファイル名 | `00NN_<app>_<内容>.sql` (例: `0004_dailytally_init.sql`)。共有テーブルは `00NN_<内容>.sql` | どのアプリ由来かをファイル名で分かるようにする |
| テーブル名 | 共有マスタは今の名前のまま (`fellowships`, `ceremonies`, `items`, `master_meta`)。アプリのテーブルは `dailytally_` / `nimotsu_` を付ける | 既存の masters / Rails 向け API を壊さない |
| アプリ固有の属性 | master テーブルに列を足さず、アプリごとの side table に分ける | master を汚さない。アプリが master 行を書き換える必要をなくす |
| 書き込み権限 | 共有マスタに書いていいのは `osystem-masters` だけ (運用ルール) | D1 にはテーブル単位の ACL が無く、binding した Worker は全テーブルに書けてしまう |
| Rails 4 本 | 対象外。SQLite + HTTP pull (`lib/tasks/masters.rake`) のまま | D1 に直接つなげないため。`osystem-masters` の `/api/*` は残す |

---

## 3. 統合後のスキーマ (共有 D1)

### 共有マスタ (変更なし)

- `fellowships`, `ceremonies`, `items`, `master_meta`

### dailytally

| 新テーブル | 旧テーブル | 備考 |
|---|---|---|
| `dailytally_fellowship_settings` | `dailytally2.fellowships` の `enabled`, `sort_order` | `fellowship_id INTEGER PRIMARY KEY REFERENCES fellowships(id)`, `enabled`, `sort_order`。`name` / `tendo_code` は捨てて master の `short_name` / `code` を JOIN で読む |
| `dailytally_ceremony_settings` | `dailytally2.ceremonies` の業務列 | `ceremony_id INTEGER PRIMARY KEY REFERENCES ceremonies(id)`, `next_number`, `begin_at`, `end_at`, `seekers_start_at`, `date_preset_key`, `sort_order`。`name` は master から読む |
| `dailytally_tally_items` | `tally_items` | `ceremony_id` を master id にリマップ |
| `dailytally_tallies` | `tallies` | `ceremony_id` をリマップ。`fellowship_id` は master と一致済み |
| `dailytally_fellowship_targets` | `fellowship_targets` | 同上 |
| `dailytally_summary_target_overrides` | `summary_target_overrides` | `ceremony_id` をリマップ |
| `dailytally_report_settings` | `report_settings` | そのまま |
| `dailytally_report_history` | `report_history` | `ceremony_id` をリマップ |
| `dailytally_app_settings` | `app_settings` | 中身に ceremony id (アクティブ護摩供など) が入っていればリマップ (要確認) |

### nimotsu

| 新テーブル | 旧テーブル | 備考 |
|---|---|---|
| `nimotsu_baggage_records` | `baggage_records` | `fellowship_id` を `code` 経由で master id にリマップ |
| (なし) | `fellowships` | 捨てる。master の `fellowships` を直接 JOIN する。`茨城` (code NULL) は 0003 で移動済みなので消すだけ |

---

## 4. id リマップの注意点

- `dailytally2.ceremonies` (id 1〜9) と master `ceremonies` は **位置で対応させず、`name` で突き合わせる**。master 側は seed の順番どおりに id が振られているはずだが、本番の値は Step 0 で確認する。
- `nimotsu-bango.fellowships` → master は `code` で突き合わせる。`code` が NULL なのは `茨城` だけで、`baggage_records` からは参照されていない前提 (Step 0 で確認する)。
- FK に `ON DELETE CASCADE` が付いているので、作り直しの間は `PRAGMA defer_foreign_keys = TRUE;` を使う (`dailytally2/migrations/0005` と同じやり方)。

---

## 5. 手順

### Step 0: 現物の確認 (読み取りのみ)

- `wrangler d1 export <db> --remote` で 3 つの D1 をダンプし、`backups/` に保存する。
- 実際のスキーマが `migrations/` と一致するか確認する (ここまではまだ確認していない)。
- master の `ceremonies` の id ↔ name、`dailytally2.ceremonies` の id ↔ name の対応表を作る。
- `nimotsu-bango.baggage_records` に `茨城` を参照する行が残っていないか確認する。
- `dailytally2.app_settings` の中身を確認する (ceremony id を持っているか)。

### Step 1: パイロット — `nimotsu-bango`

テーブル 2 つ・コード約 900 行・同期処理なし。一番安全なのでここから。

- `osystem-masters/migrations/` に `00NN_nimotsu_init.sql` (`nimotsu_baggage_records` 作成) を追加して、共有 D1 に適用する。
- 旧 DB から `baggage_records` を export → `fellowship_id` を変換 → 共有 D1 に import する。
- コード修正: `src/services/records.js`, `src/services/fellowships.js` のテーブル名を変える。`茨城`→`埼玉` の読み替え (`src/lib/auth.js`) はそのまま残す。
- `wrangler.toml`: `database_name` / `database_id` を共有 DB に変え、`migrations_dir` を削除する。
- deploy して確認する (チェックリストは §6)。

### Step 2: `dailytally2`

テーブル 9 つ、cron (`*/15`)、tendo.net 送信あり。

- 作業は cron が動かない時間帯 (送信時刻 `22:00` 前後は避ける) にやる。
- `osystem-masters/migrations/` に `00NN_dailytally_init.sql` (§3 のテーブル) を追加して適用する。
- 旧 DB を export → ceremony id をリマップ、fellowships を `dailytally_fellowship_settings` に、ceremonies の業務列を `dailytally_ceremony_settings` に分ける → import する。
- コード修正:
  - `src/services/fellowships.js`: `fellowships` JOIN `dailytally_fellowship_settings`。`name`→`short_name`、`tendo_code`→`code`。`enabled` の更新先は settings テーブル。
  - `src/services/ceremonies.js`: `ceremonies` JOIN `dailytally_ceremony_settings`。`UPDATE` は settings テーブルに向ける。
  - その他の業務テーブルは prefix を付けるだけ。
- 削除するもの: `MASTERS_URL` (`wrangler.toml`)、`src/services/master-sync.js`、`/api/sync/masters`、管理画面の「マスタ同期」ボタン。
- 残すもの: `/api/fellowships/all`, `/api/fellowships/enabled` (enabled トグルは settings テーブルを書く形で残す)。
- `wrangler.toml` を差し替えて deploy し、確認する。

### Step 3: 後片付け

- 確認期間 (目安 2 週間) が終わったら、旧 D1 `dailytally2` / `nimotsu-bango` を `wrangler d1 delete` する。
- 各アプリの `migrations/` は履歴として残し、README に「この DB はもう使っていない」と書く。
- `DATA.md` を更新する (§1・§3 の消費アプリ表、§6、§7 のアーキテクチャ)。

---

## 6. 切替時の確認とロールバック

確認:
- ログインでき、一覧・検索の件数が旧 DB と一致する。
- 伝道会名・コードが master の値で表示される。
- `dailytally2`: 集計の入力・PDF 出力・tendo.net への送信 (cron 1 回分) が通る。
- `osystem-masters` の管理画面で伝道会名を変えると、各アプリに**同期なしで**すぐ反映される。

ロールバック:
- 旧 D1 は Step 3 まで消さない。`wrangler.toml` と コードを revert して deploy すれば戻る。
- 切替後に旧 DB へ戻す場合、切替後に入ったデータは旧 DB に無い。差分は共有 D1 から手で戻す。
- 共有 D1 自体が壊れたときは `wrangler d1 time-travel` で戻す。

---

## 7. 対象外

- Rails 4 本 (`bulkpurchase`, `dedications`, `liberation`, `itementry`) の DB 統合。
- `.old/Dailytally` (`dailytally` D1) の削除判断。
- `items` マスタの初期投入 (`DATA.md` §2 の TODO)。

---

## 作業記録

- 2026-10-01: D1 の利用状況を調べて、この計画を作った。コード・DB にはまだ何も変更していない。

## 引き継ぎ

- 次にやるのは Step 0 (export と確認)。リモート D1 の実スキーマはまだ見ていない。
- `茨城` を master にどう扱うか (`DATA.md` §6 の要確認) は、nimotsu 側で埼玉に移動済みなので本計画では捨てる扱いにしている。正式に master へ追加する場合は別タスク。
