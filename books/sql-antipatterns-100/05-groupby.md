---
title: "第5章 集計・GROUP BY"
free: false
---

### No.41 GROUP BY と無関係な列を SELECT する
**❌ 悪い例**
```sql
-- Which name / created_at comes back is undefined per group
SELECT user_id, name, created_at, COUNT(*)
FROM orders o JOIN users u ON u.id = o.user_id
GROUP BY user_id;
```
**✅ 改善例**
```sql
SELECT o.user_id, u.name, COUNT(*) AS cnt
FROM orders o
JOIN users u ON u.id = o.user_id
GROUP BY o.user_id, u.name;
```
解説：グループ化していない列をそのまま出すと、代表値が不定になり、DB によってはエラー（ONLY_FULL_GROUP_BY）になります。主キーでグループ化した場合に機能的従属として許されるケースを除き、出す列は GROUP BY か集計関数に含めるのが原則です。

### No.42 WHERE で絞れる条件を HAVING に書く
**❌ 悪い例**
```sql
SELECT user_id, SUM(total_amount) AS total
FROM orders
GROUP BY user_id
HAVING status = 'paid' AND SUM(total_amount) > 10000;
```
**✅ 改善例**
```sql
SELECT user_id, SUM(total_amount) AS total
FROM orders
WHERE status = 'paid'
GROUP BY user_id
HAVING SUM(total_amount) > 10000;
```
解説：集計前の行条件は WHERE、集計結果への条件は HAVING、という役割分担が基本です。行条件を HAVING に書くと集計対象が無駄に広がり、インデックスも活かせません。可読性の面でも意図が伝わりやすくなります。

### No.43 集計をアプリ側で行う
**❌ 悪い例**
```sql
-- Fetch every row, then sum in application code
SELECT total_amount FROM orders WHERE status = 'paid';
-- app: total = sum(rows)
```
**✅ 改善例**
```sql
SELECT SUM(total_amount) AS total
FROM orders
WHERE status = 'paid';
```
解説：合計や平均は DB の集計関数に任せるのが速くて正確です。全行転送はネットワークとメモリの無駄で、件数が増えるほど破綻します。アプリ側集計が必要なのは、DB で表現しにくい複雑な業務計算に限ります。

### No.44 「上位N件」に ORDER BY + LIMIT がない
**❌ 悪い例**
```sql
-- Intended "top spenders", but no ordering and no limit
SELECT user_id, SUM(total_amount) AS total
FROM orders
GROUP BY user_id;
```
**✅ 改善例**
```sql
SELECT user_id, SUM(total_amount) AS total
FROM orders
GROUP BY user_id
ORDER BY total DESC
LIMIT 10;
```
解説：「上位」「最新」といった要件は ORDER BY と LIMIT で初めて実現します。付け忘れると全件を返してアプリ側で捨てることになり、順位の意味も保証されません。レポート系のクエリでは特に意識したい形です。

### No.45 JOIN 膨張後の SUM（重複集計）
**❌ 悪い例**
```sql
-- o.total_amount is summed once per item row
SELECT SUM(o.total_amount)
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
WHERE o.status = 'paid';
```
**✅ 改善例**
```sql
SELECT SUM(total_amount)
FROM orders
WHERE status = 'paid';
-- If item-level filtering is required, aggregate items first
```
解説：No.24 と同じ「粒度の罠」ですが、集計クエリでは GROUP BY がなくても起きます。明細テーブルを JOIN した瞬間、注文単位の金額が明細行数ぶん重複します。集計するテーブルの粒度を常に意識し、必要なら明細側を先に集計してから結合します。

### No.46 割合計算での整数除算
**❌ 悪い例**
```sql
-- Integer division: 30 / 200 becomes 0
SELECT COUNT(*) / (SELECT COUNT(*) FROM orders) AS ratio
FROM orders
WHERE status = 'paid';
```
**✅ 改善例**
```sql
SELECT COUNT(*) * 1.0 / (SELECT COUNT(*) FROM orders) AS ratio
FROM orders
WHERE status = 'paid';
-- or CAST(COUNT(*) AS DECIMAL(10,4)) / total
```
解説：DB や列の型によっては整数同士の除算が整数のまま切り捨てられ、割合が常に 0 になります。1.0 を掛ける、CAST するなど、小数になることを明示します。割合・率の計算では必ず結果の型を確認しましょう。
補足：MySQL の / は小数除算、DIV が整数除算です。PostgreSQL / SQL Server では整数同士は整数除算になります。

### No.47 集合文字列の連結で順序と上限を無視する
**❌ 悪い例**
```sql
-- Order of concatenated values is unspecified; long results get truncated
SELECT order_id, GROUP_CONCAT(product_id) AS items
FROM order_items
GROUP BY order_id;
```
**✅ 改善例**
```sql
-- MySQL
SELECT order_id,
       GROUP_CONCAT(product_id ORDER BY product_id) AS items
FROM order_items
GROUP BY order_id;
-- PostgreSQL: STRING_AGG(product_id::text, ',' ORDER BY product_id)
```
解説：GROUP_CONCAT / STRING_AGG は順序を指定しないと結果の並びが不定で、DB 側の上限長で静かに切り捨てられることもあります。順序と区切りを明示し、長い結果が想定される場合は行のまま返す設計に戻すことも検討します。

### No.48 累計を自己結合で計算する
**❌ 悪い例**
```sql
-- For each day, re-sum all previous days: O(n^2)
SELECT a.order_date, SUM(b.total_amount) AS running_total
FROM daily_sales a
JOIN daily_sales b ON b.order_date <= a.order_date
GROUP BY a.order_date;
```
**✅ 改善例**
```sql
SELECT order_date,
       SUM(total_amount) OVER (ORDER BY order_date) AS running_total
FROM daily_sales;
```
解説：累計や順位のような「行同士の関係」はウィンドウ関数の守備範囲です。自己結合は行数の二乗オーダーで重くなります。ウィンドウ関数なら1回の走査で書け、可読性も大きく上がります。

### No.49 ORDER BY RAND() でランダム抽出する
**❌ 悪い例**
```sql
-- Sorts the whole table just to pick 5 rows
SELECT id, name FROM products
ORDER BY RAND()
LIMIT 5;
```
**✅ 改善例**
```sql
-- Pick random ids in the app (min..max), then fetch by key
SELECT id, name FROM products
WHERE id IN (12, 458, 902, 1330, 2871);
-- or: WHERE id >= random_start ORDER BY id LIMIT 5
```
解説：ORDER BY RAND()（PostgreSQL では RANDOM()）は全行に乱数を付けてソートするため、テーブルが大きいほど破綻します。ID の範囲から当たりを引く、ランダムな開始点から読むなど、主キーを活かす方法に置き換えます。

### No.50 総件数と明細を別クエリで取ってずれる
**❌ 悪い例**
```sql
-- Count and page can disagree if rows change between the two queries
SELECT COUNT(*) FROM orders WHERE status = 'paid';
SELECT id, total_amount FROM orders
WHERE status = 'paid' ORDER BY id LIMIT 20 OFFSET 0;
```
**✅ 改善例**
```sql
SELECT id, total_amount, COUNT(*) OVER () AS total_count
FROM orders
WHERE status = 'paid'
ORDER BY id
LIMIT 20 OFFSET 0;
```
解説：2回のクエリの間にデータが変わると、総件数と明細の整合が取れなくなります。COUNT(*) OVER () なら同じスナップショットで総件数を同時に得られます。整合がそこまで要らない画面では、そもそも総件数を出さない割り切りも有効です。
