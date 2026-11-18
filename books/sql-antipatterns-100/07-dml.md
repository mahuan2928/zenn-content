---
title: "第7章 INSERT / UPDATE / DELETE"
free: false
---

### No.61 1行ずつ INSERT するループ
**❌ 悪い例**
```sql
-- One round trip per row
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (9001, 101, 2, 1200);
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (9001, 102, 1, 800);
-- ... repeated hundreds of times
```
**✅ 改善例**
```sql
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (9001, 101, 2, 1200),
       (9001, 102, 1, 800),
       (9001, 103, 3, 450);
```
解説：複数行 INSERT なら1回の往復で済み、トランザクション内での処理も一括化できます。大量の場合は数百〜数千行ごとの分割が目安です。ORM のバルク挿入機能も同様の効果があります。

### No.62 アプリに読み出してから別テーブルへ書き戻す
**❌ 悪い例**
```sql
-- App reads paid orders, then inserts them one by one into the archive
SELECT id, user_id, total_amount FROM orders WHERE status = 'paid';
INSERT INTO orders_archive (id, user_id, total_amount)
VALUES (?, ?, ?);  -- repeated per row
```
**✅ 改善例**
```sql
INSERT INTO orders_archive (id, user_id, total_amount)
SELECT id, user_id, total_amount
FROM orders
WHERE status = 'paid';
```
解説：DB 内のデータ移動は INSERT INTO ... SELECT で完結させます。アプリを経由すると転送と往復が増え、途中失敗時の不整合も扱いにくくなります。「読み出して加工する」必要がない移動は DB 内で閉じましょう。

### No.63 対象確認なしに UPDATE / DELETE を実行する
**❌ 悪い例**
```sql
-- Typo in the condition can update far more rows than intended
UPDATE users SET status = 'suspended' WHERE id = 42;  -- actually meant user 420?
```
**✅ 改善例**
```sql
-- 1) Preview with the same condition
SELECT id, name, status FROM users WHERE id = 42;
-- 2) Run inside a transaction and check the affected row count
BEGIN;
UPDATE users SET status = 'suspended' WHERE id = 42;
-- affected rows: 1  -> as expected
COMMIT;
```
解説：更新・削除は取り消しが効きません。実行前に同条件の SELECT で対象を確認し、影響行数が想定どおりかを見てからコミットする手順を習慣にします。本番作業では特に、WHERE なし実行の防止策（ツール側の警告設定など）も併用します。

### No.64 大量 DELETE を1回で実行する
**❌ 悪い例**
```sql
-- Millions of rows at once: long lock, huge transaction log
DELETE FROM access_logs WHERE created_at < '2026-01-01';
```
**✅ 改善例**
```sql
-- Delete in chunks, committing between batches
DELETE FROM access_logs
WHERE created_at < '2026-01-01'
ORDER BY id
LIMIT 10000;
-- repeat until 0 rows are affected
```
解説：一括の大量削除はロック時間とトランザクションログを膨らませ、レプリケーション遅延の原因にもなります。小さく刻んでコミットすれば、障害時のやり直しも容易です。テーブルごと不要なら TRUNCATE やパーティションの DROP が桁違いに速いこともあります。
補足：LIMIT 付き DELETE は MySQL 構文です。PostgreSQL では id IN (SELECT id ... LIMIT ...) の形を使います。

### No.65 自分自身を参照するサブクエリでの UPDATE
**❌ 悪い例**
```sql
-- MySQL rejects this: "You can't specify target table for update"
UPDATE orders
SET status = 'cancelled'
WHERE id IN (SELECT id FROM orders WHERE status = 'expired');
```
**✅ 改善例**
```sql
-- MySQL: wrap in a derived table, or use a JOIN update
UPDATE orders
JOIN (SELECT id FROM orders WHERE status = 'expired') t
  ON t.id = orders.id
SET orders.status = 'cancelled';
```
解説：更新対象テーブルをそのままサブクエリで参照する書き方は、DB によってはエラーになります。派生テーブルで包む、JOIN 更新に変えるなど、DB ごとの作法に合わせます。意図せず全件スキャンになることもあるため、実行計画の確認も忘れずに。
補足：PostgreSQL では UPDATE ... WHERE id IN (SELECT ...) がそのまま動作します。

### No.66 カウンタの読み取り→加算→書き戻し
**❌ 悪い例**
```sql
-- Two requests read 10, both write 11: one increment is lost
SELECT stock FROM products WHERE id = 101;   -- app reads 10
UPDATE products SET stock = 11 WHERE id = 101;  -- app writes back
```
**✅ 改善例**
```sql
-- Atomic update inside the database
UPDATE products
SET stock = stock + 1
WHERE id = 101;
-- Decrement with a guard: ... SET stock = stock - 1
-- WHERE id = 101 AND stock > 0
```
解説：読み取りと書き戻しの間に他の更新が割り込むと、加算が失われる競合（ロストアップデート）が起きます。DB 内の原子的な加減算にすれば隙間がなくなります。在庫のように下限がある値は、条件付き更新で不足を検出できます。

### No.67 UPSERT を知らず SELECT→分岐で書く
**❌ 悪い例**
```sql
SELECT 1 FROM user_settings WHERE user_id = 42 AND key_name = 'theme';
-- app: INSERT if missing, UPDATE if present (race-prone)
```
**✅ 改善例**
```sql
-- MySQL
INSERT INTO user_settings (user_id, key_name, value)
VALUES (42, 'theme', 'dark')
ON DUPLICATE KEY UPDATE value = VALUES(value);
-- PostgreSQL
-- INSERT ... ON CONFLICT (user_id, key_name)
-- DO UPDATE SET value = EXCLUDED.value;
```
解説：「あれば更新、なければ挿入」は UPSERT 構文で1文・原子的に書けます。前提として対象カラムに UNIQUE 制約が必要です。アプリ側の分岐は競合に弱く、コードも増えるため、UPSERT が使える場面ではこちらを選びます。

### No.68 保持期間の過ぎたデータを DELETE で消し続ける
**❌ 悪い例**
```sql
-- Nightly DELETE of old partitions' worth of rows, forever
DELETE FROM access_logs WHERE created_at < NOW() - INTERVAL 90 DAY;
```
**✅ 改善例**
```sql
-- Partition by time and drop old partitions instead
-- MySQL:      ALTER TABLE access_logs DROP PARTITION p2026_10;
-- PostgreSQL: DROP TABLE access_logs_2026_10;  (partitioned table)
-- No partitioning? At least chunk the DELETE (see No.64)
```
解説：時系列データの保持期間管理は、行削除ではなく期間パーティションの切り離しが本命です。DROP は一瞬で終わり、断片化も残りません。導入が難しい場合でも、チャンク分割した削除（No.64）に切り替えるだけで負荷は大きく下がります。

### No.69 更新しながら OFFSET でページングする
**❌ 悪い例**
```sql
-- Rows shift as they are updated, so some rows are skipped
SELECT id FROM orders WHERE status = 'pending'
ORDER BY id LIMIT 100 OFFSET 0;    -- process, set status = 'done'
SELECT id FROM orders WHERE status = 'pending'
ORDER BY id LIMIT 100 OFFSET 100;  -- the set already changed
```
**✅ 改善例**
```sql
-- Collect target ids first, or page by key
SELECT id FROM orders WHERE status = 'pending' ORDER BY id;
-- then update in chunks by id list,
-- or loop: WHERE status = 'pending' AND id > ? ORDER BY id LIMIT 100
```
解説：処理済みの行が集合から消えていく状況で OFFSET を使うと、行がずれて未処理が発生します。対象IDを先に確定させるか、主キーで前進するキーセット方式にします。バッチ処理の「一部だけ処理されない」系の不具合でよくある原因です。

### No.70 変更していない列まで全部 SET する
**❌ 悪い例**
```sql
-- ORM default: write back every column, changed or not
UPDATE users
SET name = 'Ann', email = 'a@example.com', status = 'active',
    created_at = '2026-04-01 10:00:00', point = 120
WHERE id = 42;
```
**✅ 改善例**
```sql
UPDATE users
SET point = 120, updated_at = NOW()
WHERE id = 42;
```
解説：全列更新は、同時編集の上書き事故や、トリガー・監査ログの無駄な発火を招きます。変更した列だけを SET するのが基本です。楽観ロック用のバージョン列を使う設計なら、更新対象と検証条件も明確になります。
