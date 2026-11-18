---
title: "第4章 インデックス"
free: false
---

### No.31 検索・ソート条件にインデックスがない
**❌ 悪い例**
```sql
-- No index on orders.user_id: full scan for every lookup
SELECT id, total_amount, created_at
FROM orders
WHERE user_id = 42
ORDER BY created_at DESC;
```
**✅ 改善例**
```sql
CREATE INDEX idx_orders_user_created
  ON orders (user_id, created_at);

SELECT id, total_amount, created_at
FROM orders
WHERE user_id = 42
ORDER BY created_at DESC;
```
解説：WHERE と ORDER BY の両方を1つの複合インデックスで賄えると、絞り込みと並び替えが同時に速くなります。まず実行頻度と遅さの両面から、効く場所に絞って張ることが大切です。EXPLAIN で確認してから本番適用しましょう。

### No.32 複合インデックスの列順を無視する
**❌ 悪い例**
```sql
-- Index is (status, created_at), but the query filters only created_at
CREATE INDEX idx_orders_status_created ON orders (status, created_at);

SELECT id FROM orders
WHERE created_at >= '2027-01-01';
```
**✅ 改善例**
```sql
-- Put the equality / most-used column first, range column after
CREATE INDEX idx_orders_created ON orders (created_at);
-- or, if status is always present:
SELECT id FROM orders
WHERE status = 'paid' AND created_at >= '2027-01-01';
```
解説：複合インデックスは左端の列から順にしか効きません（左端原則）。等値条件の列を前に、範囲条件の列を後ろに置くのが基本形です。よくある検索パターンごとに必要な複合インデックスを設計します。

### No.33 単独インデックスの乱立（過剰インデックス）
**❌ 悪い例**
```sql
CREATE INDEX i1 ON orders (user_id);
CREATE INDEX i2 ON orders (status);
CREATE INDEX i3 ON orders (created_at);
CREATE INDEX i4 ON orders (user_id, status);
CREATE INDEX i5 ON orders (user_id, created_at);
-- ... every column ends up indexed
```
**✅ 改善例**
```sql
-- Cover the real query patterns with a few composite indexes
CREATE INDEX idx_orders_user_status ON orders (user_id, status);
CREATE INDEX idx_orders_user_created ON orders (user_id, created_at);
```
解説：インデックスは読み取りを速くする代わりに、書き込みとストレージを確実に遅く・重くします。単独インデックスを足し続けるより、実際の検索パターンを賄う複合インデックスに寄せるほうが全体最適です。重複するインデックスは統合します。

### No.34 低カーディナリティ列だけのインデックス
**❌ 悪い例**
```sql
-- status has only 3 distinct values: the index barely narrows rows
CREATE INDEX idx_orders_status ON orders (status);

SELECT id FROM orders WHERE status = 'paid';
```
**✅ 改善例**
```sql
CREATE INDEX idx_orders_status_created
  ON orders (status, created_at);

SELECT id FROM orders
WHERE status = 'paid' AND created_at >= '2027-01-01';
```
解説：値の種類が少ない列単独では、インデックスを使っても対象行がほとんど減らず、オプティマイザに無視されることもあります。他の条件列と複合にすることで実用的な絞り込みになります。ただし分布が偏っている場合は例外もあるため、実行計画で確認します。

### No.35 関数をかける前提の通常インデックス
**❌ 悪い例**
```sql
-- Plain index on email cannot serve LOWER(email) lookups
SELECT id FROM users WHERE LOWER(email) = 'a@example.com';
```
**✅ 改善例**
```sql
-- PostgreSQL: functional index
CREATE INDEX idx_users_email_lower ON users (LOWER(email));
-- MySQL 8: functional index or a generated column + index
-- ALTER TABLE users ADD email_lower VARCHAR(255)
--   GENERATED ALWAYS AS (LOWER(email)) STORED,
--   ADD INDEX (email_lower);
```
解説：検索条件の形に合わせてインデックス側も用意するのが関数インデックスです。そもそも登録時に正規化（小文字化）して保存する設計にすれば、通常インデックスで済みます。検索側で毎回関数をかける設計は、保存側の正規化で解消できないか先に検討します。

### No.36 カバリングインデックスを知らず本体参照が多発する
**❌ 悪い例**
```sql
-- Index on user_id only: each hit still fetches the full row
CREATE INDEX idx_orders_user ON orders (user_id);

SELECT id, total_amount, created_at
FROM orders WHERE user_id = 42;
```
**✅ 改善例**
```sql
-- Include the selected columns so the index alone answers the query
CREATE INDEX idx_orders_user_cover
  ON orders (user_id, created_at, total_amount);

SELECT id, total_amount, created_at
FROM orders WHERE user_id = 42;
```
解説：必要な列がすべてインデックス内にあれば、テーブル本体へのランダムアクセスを省略できます。一覧取得のような定番クエリでは劇的に効きます。ただし列を増やしすぎるとインデックスが肥大するため、頻出クエリに絞って使います。
補足：PostgreSQL では INCLUDE 句でキー外の列を付加できます。MySQL では複合インデックスの列として含めます。

### No.37 ランダムな UUID 文字列を主キーにする
**❌ 悪い例**
```sql
-- Random UUID as PK: inserts scatter across the whole B-tree
CREATE TABLE orders (
  id CHAR(36) PRIMARY KEY,  -- UUID v4, fully random
  user_id BIGINT NOT NULL,
  total_amount DECIMAL(12,2) NOT NULL
);
```
**✅ 改善例**
```sql
-- Prefer sequential ids, or time-ordered UUID v7 if UUIDs are required
CREATE TABLE orders (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT NOT NULL,
  total_amount DECIMAL(12,2) NOT NULL
);
```
解説：完全ランダムな主キーは挿入位置が散らばり、ページ分割と断片化を招いて書き込みが劣化します。連番が無理でも、時刻順に並ぶ UUID v7 のような時間順IDなら局所性を保てます。IDの生成方式はインデックス性能に直結する設計判断です。
補足：PostgreSQL では AUTO_INCREMENT の代わりに GENERATED ALWAYS AS IDENTITY を使います。

### No.38 ソート専用の filesort が毎回発生する
**❌ 悪い例**
```sql
-- Index on (user_id) only: rows are sorted after fetching
SELECT id, created_at FROM orders
WHERE user_id = 42
ORDER BY created_at DESC
LIMIT 20;
```
**✅ 改善例**
```sql
CREATE INDEX idx_orders_user_created_desc
  ON orders (user_id, created_at DESC);

SELECT id, created_at FROM orders
WHERE user_id = 42
ORDER BY created_at DESC
LIMIT 20;
```
解説：インデックスの並びと ORDER BY が一致すれば、ソート処理そのものを省略して先頭から読むだけで済みます。LIMIT と組み合わさると差はさらに大きくなります。「WHERE の列＋ORDER BY の列」の複合が一覧クエリの定番形です。

### No.39 重複チェックをアプリ側だけで行う
**❌ 悪い例**
```sql
-- Race condition: two requests can both pass this check
SELECT 1 FROM users WHERE email = 'a@example.com';
-- app: if no row, then INSERT
INSERT INTO users (name, email) VALUES ('Ann', 'a@example.com');
```
**✅ 改善例**
```sql
ALTER TABLE users ADD UNIQUE KEY uq_users_email (email);

INSERT INTO users (name, email) VALUES ('Ann', 'a@example.com');
-- duplicate -> the database raises a unique violation
```
解説：アプリ側の事前チェックと登録の間には隙間があり、同時リクエストで重複がすり抜けます。一意性は DB の UNIQUE 制約で保証し、違反エラーを受けて処理するのが確実です。制約はドキュメント代わりにもなります。

### No.40 使われていないインデックスを放置する
**❌ 悪い例**
```sql
-- Legacy indexes from old features stay forever,
-- slowing every INSERT / UPDATE without helping any query
SHOW INDEX FROM orders;  -- 15 indexes, half of them unused
```
**✅ 改善例**
```sql
-- Periodically review usage and drop what is not used
-- MySQL:      sys.schema_unused_indexes
-- PostgreSQL: pg_stat_user_indexes (idx_scan = 0)
DROP INDEX idx_orders_legacy_flag ON orders;
```
解説：使われないインデックスは「ただ書き込みを遅くする荷物」です。使用状況を定期的に棚卸しし、不要なものは影響を確認してから落とします。ただしユニーク制約を兼ねるインデックスは使用頻度が低くても落としてはいけません。
