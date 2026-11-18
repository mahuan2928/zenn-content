---
title: "第8章 トランザクション・ロック"
free: false
---

### No.71 複数テーブルの更新にトランザクションがない
**❌ 悪い例**
```sql
-- If the second statement fails, the order exists without items
INSERT INTO orders (user_id, status, total_amount)
VALUES (42, 'new', 3200);
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (9001, 101, 2, 1600);  -- fails -> inconsistent state
```
**✅ 改善例**
```sql
BEGIN;
INSERT INTO orders (user_id, status, total_amount)
VALUES (42, 'new', 3200);
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (9001, 101, 2, 1600);
COMMIT;
```
解説：「一緒に成功するか、一緒に失敗するか」を保証するのがトランザクションです。注文と明細、残高と履歴のように不可分な更新は必ず1つのトランザクションにまとめます。フレームワークの宣言的トランザクションでも、境界の置き場所は自分で設計します。

### No.72 トランザクションの中で外部処理を待つ
**❌ 悪い例**
```sql
BEGIN;
UPDATE products SET stock = stock - 1 WHERE id = 101;
-- call payment API here (seconds of waiting, lock held)
-- send email here
COMMIT;
```
**✅ 改善例**
```sql
BEGIN;
UPDATE products SET stock = stock - 1 WHERE id = 101;
INSERT INTO outbox (type, payload) VALUES ('payment', '...');
COMMIT;
-- External calls run after commit, driven by the outbox / queue
```
解説：トランザクション保持中の外部API待ちは、ロックを長時間占有し、全体の詰まりを引き起こします。DB の更新だけを短く確定させ、外部連携はコミット後にキューやアウトボックス経由で行います。トランザクションは「短く」が鉄則です。

### No.73 在庫チェックを読んでから別途更新する
**❌ 悪い例**
```sql
SELECT stock FROM products WHERE id = 101;  -- both requests see 1
-- app checks stock >= 1
UPDATE products SET stock = stock - 1 WHERE id = 101;  -- oversold
```
**✅ 改善例**
```sql
-- Guarded atomic update: 0 rows affected means "sold out"
UPDATE products
SET stock = stock - 1
WHERE id = 101 AND stock >= 1;
-- or lock the row first: SELECT ... FOR UPDATE inside a transaction
```
解説：確認と更新が分かれていると、その隙間に競合が入り、在庫がマイナスになるような過剰販売が起きます。条件付きの原子的更新なら「引けたかどうか」を行数で判定できます。行ロック（FOR UPDATE）も有効ですが、保持範囲の設計が必要です。

### No.74 テーブルごとにバラバラな順序でロックする
**❌ 悪い例**
```sql
-- Tx A updates orders then users; Tx B updates users then orders
-- -> deadlock when they interleave
UPDATE orders SET status = 'x' WHERE id = 1;
UPDATE users  SET point = 0   WHERE id = 42;
```
**✅ 改善例**
```sql
-- Always lock in the same order everywhere (e.g. users -> orders)
UPDATE users  SET point = 0   WHERE id = 42;
UPDATE orders SET status = 'x' WHERE id = 1;
```
解説：デッドロックは「ロックの取得順序の食い違い」で起きます。更新するテーブルや行の順序をアプリ全体で統一すれば、発生率は大きく下がります。複数行を更新する場合も、主キーの昇順に処理するなどの規約が有効です。

### No.75 オートコミット任せで部分的に確定する
**❌ 悪い例**
```sql
-- Each statement commits on its own (autocommit)
UPDATE accounts SET balance = balance - 10000 WHERE id = 1;
-- crash here: money vanished without arriving anywhere
UPDATE accounts SET balance = balance + 10000 WHERE id = 2;
```
**✅ 改善例**
```sql
BEGIN;
UPDATE accounts SET balance = balance - 10000 WHERE id = 1;
UPDATE accounts SET balance = balance + 10000 WHERE id = 2;
COMMIT;
```
解説：送金のような対になる更新は、片方だけ確定するとデータが壊れます。オートコミットの既定動作に任せず、業務的な原子性の単位で明示的にトランザクションを張ります。障害時の状態を想像して境界を決めるのが設計の仕事です。

### No.76 集計の途中で数字が変わる（読み取り一貫性なし）
**❌ 悪い例**
```sql
-- Report runs several queries while orders keep changing:
SELECT SUM(total_amount) FROM orders WHERE status = 'paid';
SELECT COUNT(*) FROM orders WHERE status = 'paid';
-- totals and counts no longer describe the same moment
```
**✅ 改善例**
```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;  -- as needed
SELECT SUM(total_amount) FROM orders WHERE status = 'paid';
SELECT COUNT(*) FROM orders WHERE status = 'paid';
COMMIT;
-- or aggregate in a single query
```
解説：複数クエリからなる集計は、実行中に更新が入ると「同じ時点の数字」になりません。1クエリにまとめるのが第一選択で、無理ならトランザクションと適切な分離レベルでスナップショットを固定します。DB ごとの既定レベル（MySQL は REPEATABLE READ、PostgreSQL は READ COMMITTED）の違いも押さえます。

### No.77 楽観ロックなしの上書き更新
**❌ 悪い例**
```sql
-- Two editors open the same record; the later save silently wins
UPDATE products SET name = 'New name', price = 1500 WHERE id = 101;
```
**✅ 改善例**
```sql
-- Version column: 0 rows affected means someone else saved first
UPDATE products
SET name = 'New name', price = 1500, version = version + 1
WHERE id = 101 AND version = 3;
```
解説：画面編集のように保持時間の長い処理では、行ロックではなくバージョン列による楽観ロックが向きます。更新行数が0なら競合として扱い、再読み込みを促します。「最後に保存した人が無言で勝つ」仕様は、業務事故の温床です。

### No.78 ジョブキューを通常の SELECT で取り合う
**❌ 悪い例**
```sql
-- Every worker picks the same oldest job
SELECT id FROM jobs WHERE status = 'queued' ORDER BY id LIMIT 1;
UPDATE jobs SET status = 'running' WHERE id = ?;
```
**✅ 改善例**
```sql
-- PostgreSQL / MySQL 8: claim a job atomically
UPDATE jobs
SET status = 'running'
WHERE id = (SELECT id FROM jobs
            WHERE status = 'queued'
            ORDER BY id
            LIMIT 1
            FOR UPDATE SKIP LOCKED)
RETURNING id;
```
解説：キューの取り出しは「選ぶ」と「自分のものにする」を原子的に行わないと重複実行になります。FOR UPDATE SKIP LOCKED は、ロック中の行を待たずに飛ばせるためワーカー型処理に最適です。対応していないDBでは、条件付きUPDATEによる先取りでも代替できます。
補足：RETURNING は PostgreSQL 構文です。MySQL では UPDATE 後に接続単位で対象IDを保持するなどの工夫を使います。

### No.79 デッドロックを想定せずリトライしない
**❌ 悪い例**
```sql
-- A deadlock error fails the whole user request immediately
BEGIN;
UPDATE orders SET status = 'paid' WHERE id = 9001;
UPDATE products SET stock = stock - 1 WHERE id = 101;
COMMIT;  -- deadlock -> error surfaced to the user
```
**✅ 改善例**
```sql
-- Retry the whole transaction a few times on deadlock
-- (application-side loop, e.g. up to 3 attempts with a short wait)
BEGIN;
UPDATE orders SET status = 'paid' WHERE id = 9001;
UPDATE products SET stock = stock - 1 WHERE id = 101;
COMMIT;
-- on deadlock / serialization failure: rollback and retry from BEGIN
```
解説：ロック順序を揃えても、デッドロックや直列化失敗は低確率で残ります。これらは「再実行すれば通る」種類のエラーなので、トランザクション全体を数回リトライする設計にします。リトライは文単位ではなくトランザクション単位で行うのがポイントです。

### No.80 巨大な1トランザクションでバッチを回す
**❌ 悪い例**
```sql
BEGIN;
-- update 5,000,000 rows in one transaction
UPDATE orders SET archived = 1 WHERE created_at < '2026-01-01';
COMMIT;  -- long locks, huge undo, replication lag
```
**✅ 改善例**
```sql
-- Chunk by primary key and commit per chunk
UPDATE orders SET archived = 1
WHERE created_at < '2026-01-01' AND id BETWEEN 1 AND 10000;
COMMIT;
UPDATE orders SET archived = 1
WHERE created_at < '2026-01-01' AND id BETWEEN 10001 AND 20000;
COMMIT;
-- ... continue in chunks
```
解説：大きなトランザクションはロック保持・アンドゥ肥大・レプリケーション遅延を招き、失敗時の巻き戻しも長時間化します。主キーの範囲などで刻んでコミットすれば、中断・再開も容易です。バッチは「再開可能性」を設計に含めるのが実務的です。
