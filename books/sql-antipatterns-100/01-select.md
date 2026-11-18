---
title: "第1章 SELECT・取得系"
free: true
---

### No.1 SELECT * で全カラムを取得する
**❌ 悪い例**
```sql
SELECT * FROM users WHERE id = 42;
```
**✅ 改善例**
```sql
SELECT id, name, email, status
FROM users
WHERE id = 42;
```
解説：SELECT * は不要なカラムまで読み込むため、転送量とメモリを無駄にします。さらにテーブルへカラムを追加しただけでアプリ側の想定が崩れる事故の元です。必要なカラムを明示すれば、意図が明確になり、カバリングインデックスが効く余地も生まれます。

### No.2 絞り込みをアプリ側で行う
**❌ 悪い例**
```sql
-- Fetch all rows, then filter in application code
SELECT id, user_id, status, total_amount, created_at
FROM orders;
-- app: keep only rows where status = 'paid'
```
**✅ 改善例**
```sql
SELECT id, user_id, status, total_amount, created_at
FROM orders
WHERE status = 'paid';
```
解説：全件を取得してアプリ側で捨てるのは、DB・ネットワーク・アプリの三方に無駄な負荷をかけます。絞り込みはデータを持つDB側で行うのが原則です。インデックスも WHERE があってこそ活きます。

### No.3 件数取得のために全行を取得する
**❌ 悪い例**
```sql
-- Fetch all ids, then count them in application code
SELECT id FROM orders WHERE status = 'paid';
-- app: rows.size()
```
**✅ 改善例**
```sql
SELECT COUNT(*) FROM orders WHERE status = 'paid';
```
解説：件数だけ知りたいのに全行を転送するのは典型的な無駄です。COUNT(*) は件数だけを返すため高速で、メモリも消費しません。「存在するか」「何件か」「中身が要るか」を区別して書き分けましょう。

### No.4 LIMIT なしで大量結果を返す
**❌ 悪い例**
```sql
SELECT id, user_id, total_amount, created_at
FROM orders
WHERE created_at >= '2027-01-01'
ORDER BY created_at;
```
**✅ 改善例**
```sql
SELECT id, user_id, total_amount, created_at
FROM orders
WHERE created_at >= '2027-01-01'
ORDER BY created_at
LIMIT 100;
```
解説：条件次第で何万行も返るクエリは、メモリ溢れやタイムアウトの原因になります。一覧表示なら表示件数ぶんだけ、一括処理なら分割取得を前提に LIMIT やページングを必ず付けます。上限がない取得は「事故待ち」の状態です。

### No.5 深いページでの OFFSET ページング
**❌ 悪い例**
```sql
SELECT id, created_at FROM orders
ORDER BY created_at, id
LIMIT 20 OFFSET 200000;
```
**✅ 改善例**
```sql
-- Keyset pagination: pass the last seen (created_at, id)
SELECT id, created_at FROM orders
WHERE (created_at, id) > ('2027-01-10 12:00:00', 98231)
ORDER BY created_at, id
LIMIT 20;
```
解説：OFFSET は「先行する行を全部数えて捨てる」ため、ページが深くなるほど遅くなります。前ページの最終キーを条件に使うキーセットページングなら、インデックスで直接シークできて一定の速さを保てます。無限スクロールや大量一覧では特に効果的です。
補足：行値コンストラクタ (a, b) > (?, ?) は MySQL / PostgreSQL とも利用できます。

### No.6 DISTINCT で重複をごまかす
**❌ 悪い例**
```sql
SELECT DISTINCT u.id, u.name
FROM users u
JOIN orders o ON o.user_id = u.id;
```
**✅ 改善例**
```sql
-- If you only need users who have orders, say so
SELECT u.id, u.name
FROM users u
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```
解説：重複が出るのは JOIN の形が目的と合っていないサインです。DISTINCT はソートや一時テーブルで重複を消すためコストがかかり、原因も隠してしまいます。「注文したことのあるユーザー」が目的なら EXISTS で意図どおりに書くべきです。

### No.7 ORDER BY なしで先頭行に依存する
**❌ 悪い例**
```sql
-- "Latest order" without an explicit order
SELECT id, total_amount FROM orders
WHERE user_id = 42
LIMIT 1;
```
**✅ 改善例**
```sql
SELECT id, total_amount FROM orders
WHERE user_id = 42
ORDER BY created_at DESC, id DESC
LIMIT 1;
```
解説：ORDER BY がなければ返る行の順序は保証されず、実行計画の変化で突然別の行が返ることがあります。「最新」「最大」といった意味は必ず ORDER BY で明示します。同点の揺れを防ぐため、末尾に主キーでのタイブレークを付けるのが安全です。

### No.8 ループ内クエリによる N+1
**❌ 悪い例**
```sql
-- 1 query for the list, then 1 query per user (N times)
SELECT id, name FROM users WHERE status = 'active';
SELECT COUNT(*) FROM orders WHERE user_id = ?;  -- repeated per user
```
**✅ 改善例**
```sql
SELECT u.id, u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.status = 'active'
GROUP BY u.id, u.name;
```
解説：ループのたびにクエリを発行する N+1 は、件数に比例して往復コストが増える代表的な遅さの原因です。関連データは JOIN や IN による一括取得で1回にまとめます。ORM を使う場合も発行SQLを確認する習慣が重要です。

### No.9 存在確認で全カラムを取得する
**❌ 悪い例**
```sql
SELECT id, name, email, status, created_at
FROM users
WHERE email = 'a@example.com';
-- app: check whether any row came back
```
**✅ 改善例**
```sql
SELECT 1 FROM users
WHERE email = 'a@example.com'
LIMIT 1;
-- or: SELECT EXISTS(SELECT 1 FROM users WHERE email = 'a@example.com');
```
解説：存在確認に中身は要りません。SELECT 1 と LIMIT 1 で「見つかった時点で打ち切る」ことを DB に伝えられます。小さな差ですが、高頻度に呼ばれるチェック処理では効いてきます。

### No.10 パラメータ違いのクエリをループで繰り返す
**❌ 悪い例**
```sql
-- Repeated once per id in application loop
SELECT id, name, price FROM products WHERE id = ?;
```
**✅ 改善例**
```sql
SELECT id, name, price
FROM products
WHERE id IN (101, 102, 103, 104);
```
解説：同じ形のクエリを1件ずつ発行すると、往復回数ぶん遅くなります。IN 句でまとめて1回で取得すれば、DB はインデックスを効率よく使えます。件数が非常に多い場合は一時テーブルや VALUES リストとの結合も選択肢です。
