# D1 統合計画

CF Workers 系がアプリごとに別 D1 を持ち、`osystem-masters` から HTTP でマスタをコピーしている構成をやめ、**1 つの D1 を共有する**構成に移す。

ステータス: **計画のみ・未着手** (2026-10-01)

実施タイミング: **聖明王院 Cloudflare アカウントへの移管と同時にやる**。移管の全体像は [`ascension/docs/system-migration-assessment.md`](https://github.com/kusanaginoturugi/ascension/blob/main/docs/system-migration-assessment.md)。

---

## 0. 移管と同時にやる理由

- D1 の `database_id` はアカウント間で引き継げないので、移管するときは結局 export → 新規作成 → import になる。その変換のついでに id の振り直しとテーブル名の変更を済ませれば、作業が 1 回で終わる。
- 移管すると Worker の URL も `*.kusanaginoturugi.workers.dev` から `*.<domain>` に変わり、URL・OIDC・リンクを全部直すことになる。統合で変わる箇所もこれと同じタイミングでまとめて直せる。
- 移管と統合を別々にやると、運用中の切替が 2 回になる。切替を 1 回にするのが一番リスクが低い。
- 旧アカウントは移管後もそのまま残るので、ロールバックが簡単 (DNS / 案内を戻すだけ)。

移管までは今の構成を触らない。代わりに次の 2 つを守る:
- 新しく作るアプリは、自分専用の D1 に伝道会などのマスタをコピーしない。
- どうしても HTTP 同期で一時しのぎする場合も、id は master の id に揃える。

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
| 共有 DB | 新アカウントで D1 `osystem` を 1 つだけ作る | 移管でどうせ新規作成になる。DB を 3 つ作り直すより 1 つの方が楽 |
| master データ | 旧 `osystem-masters` D1 から **id を保ったまま** 投入する | 消費アプリ・Rails の FK が master id を前提にしている |
| migration の所有者 | `osystem-masters` repo の `migrations/` だけ。新 DB 用に番号を振り直す (共有テーブル + 各アプリの init) | D1 は適用済み migration を `d1_migrations` にファイル名で記録する。複数 repo から同じ DB に `migrations_dir` を向けると番号・順序が衝突する |
| 各アプリの `wrangler.toml` | `database_name = "osystem"` と新しい `database_id` を設定し、`migrations_dir` を削除 | アプリ側から migration を流させない |
| migration ファイル名 | `00NN_<app>_<内容>.sql` (例: `0010_dailytally_init.sql`)。共有テーブルは `00NN_<内容>.sql` | どのアプリ由来かをファイル名で分かるようにする |
| テーブル名 | 共有マスタは今の名前のまま (`fellowships`, `ceremonies`, `items`, `master_meta`)。アプリのテーブルは `dailytally_` / `nimotsu_` を付ける | `osystem-masters` のコードと Rails 向け API を壊さない |
| アプリ固有の属性 | master テーブルに列を足さず、アプリごとの side table に分ける | master を汚さない。アプリが master 行を書き換える必要をなくす |
| 書き込み権限 | 共有マスタに書いていいのは `osystem-masters` だけ (運用ルール) | D1 にはテーブル単位の ACL が無く、binding した Worker は全テーブルに書けてしまう |
| Rails 4 本 | DB 統合の対象外。SQLite + HTTP pull (`lib/tasks/masters.rake`) のまま。`MASTERS_URL` だけ新ドメインに変える | D1 に直接つなげないため。`osystem-masters` の `/api/*` は残す |

---

## 3. 統合後のスキーマ (共有 D1 `osystem`)

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
- FK に `ON DELETE CASCADE` が付いているので、投入の間は `PRAGMA defer_foreign_keys = TRUE;` を使う (`dailytally2/migrations/0005` と同じやり方)。

---

## 5. 移管で変わる箇所

統合と移管で同時に直すもの。`rg 'workers\.dev|showway\.biz'` で漏れがないか確認する。

| 場所 | 今の値 | 移管後 |
|---|---|---|
| `osystem-masters/wrangler.toml` | D1 `osystem-masters` / `2a5b1ce7-…` | D1 `osystem` / 新 id |
| `dailytally2/wrangler.toml` | D1 `dailytally2` / `ac0f9340-…`, `migrations_dir` | D1 `osystem` / 新 id、`migrations_dir` は削除 |
| `nimotsu-bango/wrangler.toml` | D1 `nimotsu-bango` / `0aa7a802-…`, `migrations_dir` | 同上 |
| 上記 3 つの `AUTHENTIK_ISSUER` / `AUTHENTIK_CLIENT_ID` | `auth.showway.biz` | 新 authentik の値 |
| Worker secrets (3 つ) | `AUTHENTIK_CLIENT_SECRET`, `SESSION_SECRET`, `TENDO_PASSWORD`, `RESEND_API_KEY` など | 新アカウントで入れ直す |
| authentik Redirect URI | `https://<name>.kusanaginoturugi.workers.dev/auth/callback` | `https://<host>.<domain>/auth/callback` |
| `dailytally2/wrangler.toml` `MASTERS_URL` | `osystem-masters.kusanaginoturugi.workers.dev` | **削除** (統合で同期が不要になる) |
| `dailytally2/wrangler.toml` `REPORT_NOTIFY_FROM` | `notify@showway.biz` | ドメインを変えるなら Resend 側の送信ドメイン認証もやり直す (要確認) |
| `bulkpurchase` / `dedications` / `liberation` の `config/initializers/masters.rb` | `osystem-masters.kusanaginoturugi.workers.dev` | `https://masters.<domain>` (ENV `MASTERS_URL` で上書きでも可) |
| `portal/js/app.js`, `WisdomKing/js/app.js` | `*.kusanaginoturugi.workers.dev`, `register-xju.pages.dev` | 新 URL |
| `README.md` のシステム一覧、`nimotsu-bango/README.md` の Redirect URI | 旧 URL | 新 URL |
| `DATA.md` §1・§3・§6・§7 | HTTP 同期前提 | 共有 D1 前提に書き直す |

---

## 6. 手順

### Step 0: 今のうちにやること (読み取りのみ・運用に影響なし)

- `wrangler d1 export <db> --remote` で 3 つの D1 をダンプし、`backups/` に保存する。
- 実際のスキーマが `migrations/` と一致するか確認する (ここまではまだ確認していない)。
- master の `ceremonies` の id ↔ name、`dailytally2.ceremonies` の id ↔ name の対応表を作る。
- `nimotsu-bango.baggage_records` に `茨城` を参照する行が残っていないか確認する。
- `dailytally2.app_settings` の中身を確認する (ceremony id を持っているか)。

### Step 1: 準備 (移管前・ブランチで作業)

- `osystem-masters/migrations/` を新 DB 用に組み直す (共有テーブル + `dailytally_*` + `nimotsu_*`)。
- 変換スクリプトを書く。3 つのダンプを入力にして、共有 D1 用の投入 SQL を出力する (`sh` + `sqlite3`)。
- コードを修正する (main にはまだマージしない):
  - `nimotsu-bango`: `src/services/records.js`, `src/services/fellowships.js` のテーブル名。`茨城`→`埼玉` の読み替え (`src/lib/auth.js`) はそのまま残す。
  - `dailytally2`: `src/services/fellowships.js` は `fellowships` JOIN `dailytally_fellowship_settings` にする (`name`→`short_name`、`tendo_code`→`code`、`enabled` の更新先は settings テーブル)。`src/services/ceremonies.js` は `ceremonies` JOIN `dailytally_ceremony_settings` にし、`UPDATE` は settings に向ける。その他の業務テーブルは prefix を付けるだけ。
  - `dailytally2` から削除するもの: `MASTERS_URL`、`src/services/master-sync.js`、`/api/sync/masters`、管理画面の「マスタ同期」ボタン。
  - `dailytally2` に残すもの: `/api/fellowships/all`, `/api/fellowships/enabled` (enabled トグルは settings テーブルを書く形にする)。
- ローカル D1 (`wrangler d1 execute --local`) に migration と変換結果を流して、3 アプリを `wrangler dev` で動かして確認する。何回でもやり直せるようにしておく。

### Step 2: 移管当日 (新アカウント)

- 作業は `dailytally2` の cron と 22:00 の送信を避け、護摩供の期間外にやる。
- 新アカウントで D1 `osystem` を作り、migration を適用する。
- 旧アカウントの 3 つの D1 から最新のダンプを取り → 変換スクリプトを通して → `osystem` に投入する。
- §5 の値を差し替え、secrets を入れ直してから 3 Worker を deploy する。
- 新ドメインで並行稼働させて確認する (§7)。問題なければ案内を新 URL に切り替える。
- 旧アカウントの Worker は案内を止め、書き込みを止める (読み取り専用にするか停止する)。

### Step 3: 後片付け

- 確認期間 (目安 2 週間) が終わったら、旧アカウントの D1 / Worker を消す。
- 各アプリの旧 `migrations/` は履歴として残し、README に「新 DB では使っていない」と書く。
- `DATA.md` を共有 D1 前提に書き直す。

---

## 7. 切替時の確認とロールバック

確認:
- ログインでき、一覧・検索の件数が旧 DB と一致する。
- 伝道会名・コードが master の値で表示される。
- `dailytally2`: 集計の入力・PDF 出力・tendo.net への送信 (cron 1 回分。本番送信は明示的に許可を取ってから) が通る。
- `osystem-masters` の管理画面で伝道会名を変えると、各アプリに**同期なしで**すぐ反映される。
- Rails 3 本のマスタ同期 (`masters.rake`) が新しい `masters.<domain>` から取れる。

ロールバック:
- 旧アカウントは Step 3 まで消さないので、案内を旧 URL に戻せば元どおりになる。
- ただし切替後に新環境で入力されたデータは旧 DB には無い。戻すときは差分を新 D1 から手で戻す。
- 新 D1 自体が壊れたときは `wrangler d1 time-travel` で戻す。

---

## 8. 対象外

- Rails 4 本 (`bulkpurchase`, `dedications`, `liberation`, `itementry`) の DB 統合。
- `.old/Dailytally` (`dailytally` D1) の削除判断。移管先には持っていかない (バックアップのみ)。
- `items` マスタの初期投入 (`DATA.md` §2 の TODO)。

---

## 作業記録

- 2026-10-01: D1 の利用状況を調べて、この計画を作った。コード・DB にはまだ何も変更していない。
- 2026-10-01: 実施タイミングを「アカウント移管と同時」に変更。共有 DB を「旧 `osystem-masters` D1 を使い回す」から「新アカウントで `osystem` を新規作成」に変え、移管で変わる箇所の一覧 (§5) を追加した。

## 引き継ぎ

- 次にやるのは Step 0 (export と確認)。リモート D1 の実スキーマはまだ見ていない。Step 0 は移管を待たずにやってよい。
- 移管の段取り・見積は `ascension/docs/system-migration-assessment.md` 側にある。D1 の手順はこの文書が優先 (向こうの「D1 を 3 つ作り直す」前提は採らない)。
- `茨城` を master にどう扱うか (`DATA.md` §6 の要確認) は、nimotsu 側で埼玉に移動済みなので本計画では捨てる扱いにしている。正式に master へ追加する場合は別タスク。
