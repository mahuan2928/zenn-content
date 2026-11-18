---
title: "第2章 WHERE・絞り込み"
free: false
---

### No.11 カラムに関数をかけて絞り込む
**❌ 悪い例**
```sql
SELECT id, total_amount FROM orders
WHERE DATE(created_at) = '2027-01-15';
```
**✅ 改善例**
```sql
SELECT id, total_amount FROM orders
WHERE created_at >= '2027-01-15'
  AND created_at <  '2027-01-16';
```
解説：カラム側に関数を適用すると、通常のインデックスが使えずフルスキャンになりがちです。範囲条件に書き換えればインデックスが効き、月次集計のような重い処理でも効きます。「関数は値側に、カラムは素のまま」が原則です。

### No.12 先頭ワイルドカードの LIKE
**❌ 悪い例**
```sql
SELECT id, name FROM products
WHERE name LIKE '%wireless%';
```
**✅ 改善例**
```sql
-- Prefix match can use an index
SELECT id, name FROM products
WHERE name LIKE 'wireless%';
-- For contains-search, use a full-text index instead
-- MySQL:      FULLTEXT(name) ... MATCH(name) AGAINST ('wireless')
-- PostgreSQL: to_tsvector / pg_trgm
```
解説：LIKE の先頭ワイルドカードは B-tree インデックスで範囲を絞れないため、基本的にフルスキャンです。前方一致で足りるなら前方一致にし、部分一致検索が要件なら全文検索インデックスや検索エンジンの導入を検討します。

### No.13 暗黙の型変換に頼る比較
**❌ 悪い例**
```sql
-- code is VARCHAR, but compared with a number
SELECT id, name FROM users WHERE code = 123;
```
**✅ 改善例**
```sql
SELECT id, name FROM users WHERE code = '123';
```
解説：文字列カラムに数値を渡すと、DB が暗黙変換を行い、インデックスが使えなくなることがあります。さらに '00123' のような値との比較結果が直感とずれる事故も起きます。比較はカラムの型に合わせ、アプリ側で型を揃えてから渡しましょう。

### No.14 OR の羅列でインデックスを活かせない
**❌ 悪い例**
```sql
SELECT id, status FROM orders
WHERE status = 'paid'
   OR status = 'shipped'
   OR status = 'delivered';
```
**✅ 改善例**
```sql
SELECT id, status FROM orders
WHERE status IN ('paid', 'shipped', 'delivered');
```
解説：同一カラムへの OR の羅列は IN で等価に書け、可読性と最適化の両面で有利です。異なるカラム同士の OR はインデックスが効きにくい代表例で、その場合は UNION への分解や設計の見直しを検討します。

### No.15 NOT IN の NULL 罠
**❌ 悪い例**
```sql
-- If the subquery returns even one NULL, the result is empty
SELECT id, name FROM users
WHERE id NOT IN (SELECT user_id FROM banned_users);
```
**✅ 改善例**
```sql
SELECT id, name FROM users u
WHERE NOT EXISTS (
  SELECT 1 FROM banned_users b WHERE b.user_id = u.id
);
```
解説：NOT IN の対象に NULL が1つでも混ざると、結果全体が空（正確には1行も真にならない）になります。意図せぬ全件消失として現れるため非常に気づきにくい罠です。否定の存在条件は NOT EXISTS で書くのが安全です。

### No.16 否定条件（!=、NOT）での絞り込み
**❌ 悪い例**
```sql
SELECT id, name FROM users
WHERE status != 'deleted';
```
**✅ 改善例**
```sql
SELECT id, name FROM users
WHERE status IN ('active', 'suspended');
-- or keep an explicit flag column and index the positive side
```
解説：否定条件は「それ以外すべて」を対象にするためインデックスで絞りにくく、意図しない値（想定外の新ステータスや NULL）まで含めてしまいます。対象を肯定的に列挙したほうが、速くて意図も明確です。なお != では NULL の行は含まれません。

### No.17 常に真の条件と文字列連結での条件組み立て
**❌ 悪い例**
```sql
-- App builds SQL by string concatenation
-- SELECT ... FROM orders WHERE 1=1 AND status = '<user input>'
SELECT id FROM orders WHERE 1 = 1 AND status = 'paid';
```
**✅ 改善例**
```sql
-- Use bind parameters; add conditions only when needed
SELECT id FROM orders WHERE status = ?;
```
解説：WHERE 1=1 を土台に文字列連結で条件を足す書き方は、SQLインジェクションの温床であり、クエリキャッシュやプランの再利用も妨げます。値は必ずバインドパラメータで渡し、条件の有無はアプリ側の分岐で組み立てます。

### No.18 BETWEEN の終端で時刻が落ちる
**❌ 悪い例**
```sql
-- Intended: all of January 2027
SELECT id, created_at FROM orders
WHERE created_at BETWEEN '2027-01-01' AND '2027-01-31';
```
**✅ 改善例**
```sql
SELECT id, created_at FROM orders
WHERE created_at >= '2027-01-01'
  AND created_at <  '2027-02-01';
```
解説：DATETIME 型に日付だけの終端を渡すと、その日の 00:00:00 として解釈され、31日中のデータがほぼ全て漏れます。'23:59:59' と書く応急処置もミリ秒精度で漏れます。期間は「開始以上、終了未満」の半開区間で書くのが確実です。

### No.19 = NULL で NULL 判定する
**❌ 悪い例**
```sql
SELECT id, name FROM users WHERE deleted_at = NULL;
```
**✅ 改善例**
```sql
SELECT id, name FROM users WHERE deleted_at IS NULL;
```
解説：NULL は「値が不明」であり、何とも等しくありません。= NULL は常に真にならず、エラーにもならないため静かにバグります。NULL 判定は IS NULL / IS NOT NULL を使います。これは初学者だけでなく、動的SQLの組み立てでも起きがちなミスです。

### No.20 カラム側で計算してから比較する
**❌ 悪い例**
```sql
SELECT id, name, price FROM products
WHERE price * 1.1 > 10000;
```
**✅ 改善例**
```sql
SELECT id, name, price FROM products
WHERE price > 10000 / 1.1;
```
解説：WHERE 句でカラムに演算を施すとインデックスが使えなくなります。計算は値側へ移すか、どうしても必要なら生成列（generated column）や関数インデックスを用意します。税込・税抜のような換算は、保存時の基準を決めておく設計判断も重要です。
