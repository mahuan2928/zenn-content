---
title: "第10章 NULL・型・日付の扱い"
free: false
---

### No.91 NULL と空文字を混在させる
**❌ 悪い例**
```sql
-- Some rows have NULL, others '' for "no value"
SELECT COUNT(*) FROM users WHERE email = '';   -- misses NULL rows
SELECT COUNT(*) FROM users WHERE email IS NULL; -- misses '' rows
```
**✅ 改善例**
```sql
-- Pick one representation (e.g. NULL) and enforce it
UPDATE users SET email = NULL WHERE email = '';
-- Queries then use a single form:
SELECT COUNT(*) FROM users WHERE email IS NULL;
```
解説：「値なし」の表現が混ざると、あらゆる検索が2条件必要になり、漏れの温床になります。NULL か空文字か、どちらかに統一し、登録経路で矯正します。移行時は既存データの寄せ（UPDATE で統一）を忘れずに行います。

### No.92 集計で NULL を意図せず除外・伝播させる
**❌ 悪い例**
```sql
-- AVG ignores NULLs; SUM of all-NULL returns NULL, not 0
SELECT AVG(discount), SUM(discount) FROM orders;
-- point + NULL = NULL in row expressions:
SELECT point + bonus AS total_point FROM users;
```
**✅ 改善例**
```sql
SELECT AVG(COALESCE(discount, 0)) AS avg_discount,
       COALESCE(SUM(discount), 0) AS sum_discount
FROM orders;
SELECT point + COALESCE(bonus, 0) AS total_point FROM users;
```
解説：集計関数は NULL を除外し、式の中の NULL は結果全体を NULL にします。「0 として扱いたい」のか「対象外にしたい」のかを決め、COALESCE で意図を明示します。平均の分母が変わる点は特に、レポートの数字合わせで問題化します。

### No.93 UNIQUE 制約と NULL の組み合わせの誤解
**❌ 悪い例**
```sql
-- Expecting "one row per coupon per user", but NULL user_id
-- rows can be inserted any number of times
CREATE TABLE coupon_uses (
  id BIGINT PRIMARY KEY,
  user_id BIGINT NULL,
  coupon_code VARCHAR(20) NOT NULL,
  UNIQUE KEY (user_id, coupon_code)
);
```
**✅ 改善例**
```sql
-- Make the column NOT NULL if it must be unique,
-- or use a generated key that maps NULL to a sentinel
ALTER TABLE coupon_uses MODIFY user_id BIGINT NOT NULL;
-- PostgreSQL 15+: UNIQUE NULLS NOT DISTINCT (user_id, coupon_code)
```
解説：標準SQLでは UNIQUE 制約は NULL 同士を「異なるもの」として扱うため、NULL を含む組み合わせは重複し放題になります。一意性が要件なら NOT NULL 化するか、DB の拡張（NULLS NOT DISTINCT 等）を使います。仕様として知らないと静かに破綻するポイントです。

### No.94 タイムゾーンなしで日時を保存・比較する
**❌ 悪い例**
```sql
-- Server local time, app local time, and UTC get mixed
INSERT INTO orders (created_at) VALUES (NOW());
SELECT * FROM orders WHERE created_at >= '2027-01-01 00:00:00';
-- Whose midnight is this?
```
**✅ 改善例**
```sql
-- Store in UTC, convert at display time
-- PostgreSQL: created_at TIMESTAMPTZ NOT NULL DEFAULT now()
-- MySQL: store DATETIME in UTC and fix the session time_zone = '+00:00'
SELECT * FROM orders
WHERE created_at >= '2027-01-01 00:00:00';  -- interpreted as UTC by rule
```
解説：タイムゾーンの混在は「日付がずれる」「締め集計が合わない」といった静かな事故を生みます。保存は UTC に統一し、表示時に利用者のタイムゾーンへ変換するのが定石です。DB の型（TIMESTAMP / TIMESTAMPTZ）の変換仕様も合わせて確認します。

### No.95 日付計算を文字列組み立てで行う
**❌ 悪い例**
```sql
-- App builds '2027-02-31' or breaks at month boundaries
SELECT * FROM orders
WHERE created_at >= '2027-01-31' + 1;  -- string arithmetic, DB-dependent
```
**✅ 改善例**
```sql
-- MySQL
SELECT * FROM orders
WHERE created_at >= DATE_ADD('2027-01-31', INTERVAL 1 DAY);
-- PostgreSQL
-- WHERE created_at >= DATE '2027-01-31' + INTERVAL '1 day'
-- "One month later" is also safe: + INTERVAL 1 MONTH
```
解説：日付の加減算は月末・うるう年・月跨ぎの規則を含むため、DB の日付関数に任せます。文字列や数値での自前計算は、特定の日付でだけ壊れる厄介なバグになります。「来月の同日」のような表現も INTERVAL の月加算なら安全です。

### No.96 年齢・経過年数を年の差だけで計算する
**❌ 悪い例**
```sql
-- Wrong for everyone whose birthday has not come yet this year
SELECT id, YEAR(CURDATE()) - YEAR(birth_date) AS age
FROM users;
```
**✅ 改善例**
```sql
-- MySQL
SELECT id, TIMESTAMPDIFF(YEAR, birth_date, CURDATE()) AS age
FROM users;
-- PostgreSQL: EXTRACT(YEAR FROM AGE(birth_date))
```
解説：年の差は誕生日を迎えたかどうかを考慮しないため、最大で1歳ずれます。年齢や勤続年数のような「経過期間」は専用関数で計算します。条件判定（成人確認など）に使う場合は特に正確さが求められます。

### No.97 数値を文字列で持ってソートが崩れる
**❌ 悪い例**
```sql
-- Stored as text: '10' sorts before '9'
SELECT code FROM items ORDER BY code;
-- '1', '10', '2', '9' ...
```
**✅ 改善例**
```sql
-- Store numbers as numbers
ALTER TABLE items MODIFY code INT NOT NULL;
SELECT code FROM items ORDER BY code;
-- If text is unavoidable: ORDER BY CAST(code AS UNSIGNED) (MySQL)
-- or zero-pad at write time: '0009', '0010'
```
解説：文字列の並びは辞書順であり、数値の大小とは一致しません。並べ替えや範囲条件が必要な値は数値型で持ちます。表示上の桁揃えはアプリ側で行うか、ゼロ埋めした文字列として書き込み時に統一します。No.86 の「識別子は文字列」と対になる原則です。

### No.98 真偽値を曖昧な表現で持つ
**❌ 悪い例**
```sql
-- 'Y' / 'N' / '1' / '0' / NULL all mean something, somewhere
CREATE TABLE users (
  id BIGINT PRIMARY KEY,
  is_active CHAR(1) NULL  -- NULL = "unknown"? or "inactive"?
);
SELECT * FROM users WHERE is_active <> 'N';  -- NULL rows are excluded
```
**✅ 改善例**
```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY,
  is_active BOOLEAN NOT NULL DEFAULT TRUE
);
SELECT * FROM users WHERE is_active;
```
解説：真偽は2値であるべきなのに、NULL を含めた3値や文字表現が混ざると、条件のたびに解釈が必要になります。BOOLEAN（または TINYINT(1)）＋NOT NULL で「真か偽か」だけに閉じます。「不明」という状態が本当に必要なら、ステータス列として別に表現します。

### No.99 照合順序（コレーション）依存の比較で検索漏れする
**❌ 悪い例**
```sql
-- Collation decides whether this matches 'ABC' or full-width variants
SELECT id FROM users WHERE name = 'tanaka';
-- Results differ between utf8mb4_bin and *_ci collations
```
**✅ 改善例**
```sql
-- Decide matching rules explicitly:
-- case-insensitive: use a *_ci collation column or LOWER() + functional index
-- exact match:     COLLATE utf8mb4_bin
SELECT id FROM users WHERE email = 'a@example.com';  -- stored normalized (lowercase)
```
解説：大文字小文字・全角半角・濁点の扱いはカラムの照合順序で決まります。「環境によって結果が違う」事故を防ぐには、登録時の正規化（小文字化など）と照合順序の明示が有効です。メールアドレスのような識別子は正規化して保存するのが確実です。

### No.100 税・割引の丸めをバラバラに行う
**❌ 悪い例**
```sql
-- Rounding at different points produces mismatched totals
SELECT SUM(ROUND(unit_price * quantity * 1.1)) AS total_a,  -- per row
       ROUND(SUM(unit_price * quantity) * 1.1) AS total_b   -- per order
FROM order_items;
-- total_a != total_b
```
**✅ 改善例**
```sql
-- Fix one rounding rule (e.g. tax per order, rounded down) and
-- apply it in exactly one place, with DECIMAL arithmetic
SELECT ROUND(SUM(unit_price * quantity) * 1.1, 0) AS total
FROM order_items
WHERE order_id = 9001;
```
解説：丸めは「どの単位で・いつ・どう丸めるか」を1つに決めて初めて数字が合います。明細ごとと合計後では結果が変わるため、業務ルールとして固定し、計算は DECIMAL で行います。アプリと DB の両方で計算する場合は、どちらを正とするかも決めておきます。
