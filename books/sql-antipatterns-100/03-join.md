---
title: "第3章 JOIN"
free: false
---

### No.21 カンマ結合による暗黙的 JOIN
**❌ 悪い例**
```sql
SELECT o.id, u.name
FROM orders o, users u
WHERE o.user_id = u.id;
```
**✅ 改善例**
```sql
SELECT o.id, u.name
FROM orders o
JOIN users u ON u.id = o.user_id;
```
解説：カンマ結合は結合条件と絞り込み条件が WHERE に混ざり、条件を1つ書き忘れただけで直積（クロス結合）になります。JOIN ... ON で結合の意図を明示すると、読み手にもオプティマイザにも意図が伝わります。

### No.22 結合条件の書き忘れによる直積
**❌ 悪い例**
```sql
-- Missing join condition: every order pairs with every product
SELECT o.id, p.name
FROM orders o
JOIN products p ON p.category_id = o.user_id;  -- wrong columns, still a cross-like result
```
**✅ 改善例**
```sql
SELECT o.id, p.name
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id;
```
解説：多対多の関係は中間テーブルを経由するのが正しい形です。無関係なカラム同士を結ぶとエラーにはならず、件数だけが静かに膨らみます。JOIN を書いたら「行数が想定どおりか」を必ず確認しましょう。

### No.23 LEFT JOIN が WHERE のせいで INNER JOIN 化する
**❌ 悪い例**
```sql
-- Users without paid orders disappear anyway
SELECT u.id, u.name, o.id
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.status = 'paid';
```
**✅ 改善例**
```sql
SELECT u.id, u.name, o.id
FROM users u
LEFT JOIN orders o
  ON o.user_id = u.id
 AND o.status = 'paid';
```
解説：LEFT JOIN の右表への条件を WHERE に書くと、結合できなかった行（右表が NULL）も除外され、実質 INNER JOIN になります。「存在しない行も残したい」なら条件は ON 側へ書きます。逆に除外したいなら素直に INNER JOIN を使いましょう。

### No.24 JOIN の膨張をそのまま集計する
**❌ 悪い例**
```sql
-- total_amount is counted once per item row
SELECT u.id, SUM(o.total_amount) AS total
FROM users u
JOIN orders o ON o.user_id = u.id
JOIN order_items oi ON oi.order_id = o.id
GROUP BY u.id;
```
**✅ 改善例**
```sql
SELECT u.id, SUM(o.total_amount) AS total
FROM users u
JOIN orders o ON o.user_id = u.id
GROUP BY u.id;
-- or pre-aggregate each table before joining
```
解説：1対多の JOIN を重ねると親側の値が明細行の数だけ重複し、SUM が過大になります。集計前に対象テーブルの粒度を確認し、不要な JOIN を外すか、サブクエリで事前に集計してから結合します。「金額が合わない」系の障害で最も多い原因の1つです。

### No.25 結合キーの型・桁の不一致
**❌ 悪い例**
```sql
-- orders.user_id is INT, users.id is BIGINT (or VARCHAR in another table)
SELECT o.id
FROM orders o
JOIN users u ON u.id = o.user_id;
```
**✅ 改善例**
```sql
-- Align types at design time: both BIGINT
-- ALTER TABLE orders MODIFY user_id BIGINT NOT NULL;
SELECT o.id
FROM orders o
JOIN users u ON u.id = o.user_id;
```
解説：結合キーの型が違うと、暗黙変換のためにインデックスが使えず、巨大なテーブル同士の全件結合になり得ます。外部キーを張る際は参照先と型・符号・照合順序まで揃えるのが原則です。後からの変更は高くつくため、設計時に統一します。

### No.26 使わないテーブルまで JOIN する
**❌ 悪い例**
```sql
SELECT o.id, o.total_amount
FROM orders o
JOIN users u ON u.id = o.user_id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id;
```
**✅ 改善例**
```sql
SELECT o.id, o.total_amount
FROM orders o;
```
解説：SELECT 句にも WHERE 句にも出てこない JOIN は、コストを増やし、行の重複リスクだけを残します。コピペで育ったクエリに多いパターンで、消すだけで速くなることも珍しくありません。JOIN は「その列が要る」「絞り込みに要る」場合だけ書きます。

### No.27 JOIN で書ける処理を相関サブクエリで書く
**❌ 悪い例**
```sql
SELECT u.id, u.name,
       (SELECT MAX(o.created_at) FROM orders o
         WHERE o.user_id = u.id) AS last_order_at
FROM users u;
```
**✅ 改善例**
```sql
SELECT u.id, u.name, MAX(o.created_at) AS last_order_at
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name;
```
解説：SELECT 句の相関サブクエリは行ごとに実行されるため、件数が増えると直撃します。集計＋JOIN で書けば1回の処理にまとめられます。もちろん相関サブクエリが有利な場面もありますが、まずは JOIN 形を検討するのが順序です。

### No.28 NATURAL JOIN / USING への依存
**❌ 悪い例**
```sql
-- Silently changes meaning if a same-named column is added later
SELECT o.id, total_amount
FROM orders o
NATURAL JOIN users;
```
**✅ 改善例**
```sql
SELECT o.id, o.total_amount
FROM orders o
JOIN users u ON u.id = o.user_id;
```
解説：NATURAL JOIN は「同名カラム全部」で結合するため、後から同名カラム（例：status、created_at）を追加しただけで意味が変わります。結合条件は ON で明示し、将来のスキーマ変更に強いクエリを書きましょう。

### No.29 結合キーにインデックスがない巨大テーブル同士の JOIN
**❌ 悪い例**
```sql
-- order_items.product_id has no index: full scan on every join
SELECT p.name, SUM(oi.quantity)
FROM order_items oi
JOIN products p ON p.id = oi.product_id
GROUP BY p.name;
```
**✅ 改善例**
```sql
CREATE INDEX idx_items_product ON order_items (product_id);

SELECT p.name, SUM(oi.quantity)
FROM order_items oi
JOIN products p ON p.id = oi.product_id
GROUP BY p.name;
```
解説：外部キー側のカラムにインデックスがないと、結合のたびに全件走査やハッシュ結合の構築が発生します。参照される側（主キー）だけでなく、参照する側の外部キーにもインデックスを張るのが基本です。外部キー制約を張ると自動で作られるDBもありますが、過信せず確認します。

### No.30 LEFT JOIN 先の件数を COUNT(*) で数える
**❌ 悪い例**
```sql
-- Users with no orders are counted as 1
SELECT u.id, COUNT(*) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id;
```
**✅ 改善例**
```sql
SELECT u.id, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id;
```
解説：LEFT JOIN では相手がいない行も1行として残るため、COUNT(*) では「0件」が「1件」になります。右表の件数を数えたいときは COUNT(右表の主キー) を使います。COUNT(*) は行数、COUNT(列) は非NULL数という違いを意識しましょう。
