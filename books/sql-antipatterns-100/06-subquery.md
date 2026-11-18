---
title: "第6章 サブクエリ"
free: false
---

### No.51 SELECT 句の相関サブクエリを全行で実行する
**❌ 悪い例**
```sql
SELECT p.id, p.name,
       (SELECT COUNT(*) FROM order_items oi
         WHERE oi.product_id = p.id) AS sold_count
FROM products p;
```
**✅ 改善例**
```sql
SELECT p.id, p.name, COUNT(oi.id) AS sold_count
FROM products p
LEFT JOIN order_items oi ON oi.product_id = p.id
GROUP BY p.id, p.name;
```
解説：相関サブクエリは外側の行ごとに実行されるため、行数に比例してコストが増えます。JOIN＋GROUP BY に書き換えれば一括処理になります。ウィンドウ関数が向くケースもあるので、「行ごとに繰り返していないか」を基準に点検します。

### No.52 IN（サブクエリ）の巨大化と NULL リスク
**❌ 悪い例**
```sql
SELECT id, name FROM users
WHERE id IN (SELECT user_id FROM orders);  -- huge, may include NULLs
```
**✅ 改善例**
```sql
SELECT id, name FROM users u
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```
解説：存在確認の目的なら EXISTS が意図どおりで、相手が見つかった時点で打ち切れます。IN は「値リストとの一致」には適しますが、巨大なサブクエリや NULL の混入には弱い面があります。用途に応じて使い分けましょう。

### No.53 派生テーブルの内側で絞り込まない
**❌ 悪い例**
```sql
-- Inner query materializes all orders before filtering
SELECT *
FROM (SELECT id, user_id, status, total_amount, created_at
      FROM orders) t
WHERE t.status = 'paid';
```
**✅ 改善例**
```sql
SELECT id, user_id, status, total_amount, created_at
FROM orders
WHERE status = 'paid';
-- If a derived table is needed, push the filter inside it
```
解説：絞り込みを外側に書くと、内側で全件を実体化してから捨てることになりがちです。条件は可能な限り内側（データに近い側）へ寄せます。オプティマイザが自動で寄せてくれる場合もありますが、頼り切らず自分で内側に書くのが確実です。

### No.54 最大値の行をサブクエリの二度打ちで取る
**❌ 悪い例**
```sql
SELECT id, user_id, total_amount FROM orders
WHERE total_amount = (SELECT MAX(total_amount) FROM orders);
```
**✅ 改善例**
```sql
SELECT id, user_id, total_amount FROM orders
ORDER BY total_amount DESC
LIMIT 1;
-- Keep ties: use RANK() OVER (ORDER BY total_amount DESC)
```
解説：最大の「行」が欲しいのか最大の「値」が欲しいのかで書き方は変わります。行が欲しいなら ORDER BY + LIMIT が簡潔で、同点の扱いも制御できます。MAX との等値比較は同点行が全部返る点も意識しましょう。

### No.55 同じサブクエリを何度もコピーする
**❌ 悪い例**
```sql
SELECT (SELECT SUM(total_amount) FROM orders WHERE status = 'paid') AS paid_total,
       (SELECT SUM(total_amount) FROM orders WHERE status = 'paid'
         AND created_at >= '2027-01-01') AS paid_total_this_year,
       (SELECT COUNT(*) FROM orders WHERE status = 'paid') AS paid_count;
```
**✅ 改善例**
```sql
WITH paid AS (
  SELECT total_amount, created_at FROM orders WHERE status = 'paid'
)
SELECT SUM(total_amount) AS paid_total,
       SUM(CASE WHEN created_at >= '2027-01-01'
                THEN total_amount ELSE 0 END) AS paid_total_this_year,
       COUNT(*) AS paid_count
FROM paid;
```
解説：同じ走査を何度も書くと遅いだけでなく、条件の修正漏れで数字同士が矛盾します。CTE（WITH 句）で共通部分を定義し、条件付き集計で1回にまとめます。保守性の差がそのまま障害の少なさにつながります。

### No.56 ID 一覧をアプリに往復させて再投入する
**❌ 悪い例**
```sql
-- Step 1 (app fetches ids), Step 2 (app sends them back)
SELECT id FROM orders WHERE status = 'paid';
SELECT * FROM order_items WHERE order_id IN (/* thousands of ids */);
```
**✅ 改善例**
```sql
SELECT oi.*
FROM order_items oi
JOIN orders o ON o.id = oi.order_id
WHERE o.status = 'paid';
```
解説：中間結果をアプリに持ち出して戻すと、転送量・往復回数・SQL の長さがすべて膨らみます。DB 内で完結できる処理は JOIN やサブクエリで1本化します。段階を分けたい場合も、一時テーブルや CTE が候補です。

### No.57 存在確認に IN を使う（EXISTS の出番）
**❌ 悪い例**
```sql
SELECT p.id, p.name FROM products p
WHERE p.id IN (SELECT product_id FROM order_items);
```
**✅ 改善例**
```sql
SELECT p.id, p.name FROM products p
WHERE EXISTS (
  SELECT 1 FROM order_items oi WHERE oi.product_id = p.id
);
```
解説：No.52 と同じ方向ですが、こちらは可読性の問題です。「売れたことのある商品」という意図は EXISTS の形が最も素直に伝わります。重複行も出ません。最近のオプティマイザは両者を等価に最適化することも多いので、意図の伝わりやすさで選んで構いません。

### No.58 UPDATE / DELETE の相関サブクエリ連打
**❌ 悪い例**
```sql
UPDATE orders o
SET status = 'archived'
WHERE (SELECT COUNT(*) FROM order_items oi
        WHERE oi.order_id = o.id) = 0
  AND o.created_at < '2026-01-01';
```
**✅ 改善例**
```sql
UPDATE orders o
LEFT JOIN order_items oi ON oi.order_id = o.id
SET o.status = 'archived'
WHERE oi.id IS NULL
  AND o.created_at < '2026-01-01';
-- PostgreSQL: UPDATE orders o SET ... WHERE NOT EXISTS (...)
```
解説：更新対象の行ごとにサブクエリを回す書き方は、対象が多いほど遅くなります。JOIN 更新や NOT EXISTS で集合として処理します。更新系は特に、事前に同条件の SELECT で対象件数を確認してから実行するのが安全です。
補足：UPDATE ... LEFT JOIN の形は MySQL 構文です。PostgreSQL では UPDATE ... FROM を使います。

### No.59 ビューの上にビューを重ねる
**❌ 悪い例**
```sql
-- v_order_summary -> v_order_summary_paid -> v_monthly_paid ...
CREATE VIEW v_monthly_paid AS
SELECT * FROM v_order_summary_paid;  -- which itself wraps another view
```
**✅ 改善例**
```sql
-- Define reporting views directly on base tables,
-- or keep the chain shallow and documented
CREATE VIEW v_monthly_paid AS
SELECT DATE_FORMAT(created_at, '%Y-%m') AS ym,
       SUM(total_amount) AS total
FROM orders
WHERE status = 'paid'
GROUP BY ym;
```
解説：ビューの多段ネストは、実行時に展開される実クエリが見えなくなり、最適化も効きにくくなります。段数は浅く保ち、最終的にどんなSQLになるかを把握できる範囲に留めます。再利用したい集計は、実テーブルへの定期集計（サマリテーブル）も選択肢です。
補足：DATE_FORMAT は MySQL 構文です。PostgreSQL では to_char(created_at, 'YYYY-MM') を使います。

### No.60 順位付けを相関 COUNT で行う
**❌ 悪い例**
```sql
-- Rank by counting "rows greater than me" for every row
SELECT o1.id,
       (SELECT COUNT(*) FROM orders o2
         WHERE o2.total_amount > o1.total_amount) + 1 AS rnk
FROM orders o1;
```
**✅ 改善例**
```sql
SELECT id,
       RANK() OVER (ORDER BY total_amount DESC) AS rnk
FROM orders;
```
解説：順位・連番・前行との差分といった処理は、ウィンドウ関数（RANK、ROW_NUMBER、LAG など）のためにあります。相関サブクエリで書くと二乗オーダーになりやすく、意図も伝わりません。分析系クエリではまずウィンドウ関数を検討します。
