---
title: "第9章 テーブル設計"
free: false
---

### No.81 EAV（キー・バリュー行）で属性を表現する
**❌ 悪い例**
```sql
-- Every attribute is a row: no types, no constraints, painful queries
CREATE TABLE product_attributes (
  product_id BIGINT NOT NULL,
  attr_name  VARCHAR(50) NOT NULL,
  attr_value VARCHAR(255),
  PRIMARY KEY (product_id, attr_name)
);
SELECT * FROM product_attributes
WHERE product_id = 101 AND attr_name IN ('color', 'size', 'weight');
```
**✅ 改善例**
```sql
-- Stable attributes get real columns; only truly variable
-- extras go to JSON
CREATE TABLE products (
  id BIGINT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  color VARCHAR(30),
  size VARCHAR(10),
  weight_kg DECIMAL(6,2),
  extra JSON
);
```
解説：EAV は柔軟に見えて、型・制約・インデックス・集計のすべてを失います。安定した属性は通常のカラムにし、本当に可変な部分だけ JSON に逃がすハイブリッドが現実解です。「何でも入る」は「何も保証しない」と同義だと心得ます。

### No.82 複数値をカンマ区切り文字列で持つ
**❌ 悪い例**
```sql
CREATE TABLE products (
  id BIGINT PRIMARY KEY,
  name VARCHAR(100),
  tag_ids VARCHAR(255)  -- '3,7,15'
);
SELECT * FROM products WHERE tag_ids LIKE '%7%';  -- matches 17, 70 too
```
**✅ 改善例**
```sql
CREATE TABLE product_tags (
  product_id BIGINT NOT NULL,
  tag_id BIGINT NOT NULL,
  PRIMARY KEY (product_id, tag_id),
  KEY (tag_id, product_id)
);
SELECT p.* FROM products p
JOIN product_tags pt ON pt.product_id = p.id
WHERE pt.tag_id = 7;
```
解説：カンマ区切りは検索・結合・制約のすべてを不可能にし、LIKE 検索では誤検出もします。多対多は中間テーブルが正解です。第1正規形（繰り返しを持たない）は、後からの変更コストを下げる基本原則です。

### No.83 フラグ列の乱立で状態を表現する
**❌ 悪い例**
```sql
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  is_paid TINYINT, is_shipped TINYINT,
  is_delivered TINYINT, is_cancelled TINYINT
  -- contradictory combinations are possible
);
```
**✅ 改善例**
```sql
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  status VARCHAR(20) NOT NULL DEFAULT 'new'
  -- 'new' / 'paid' / 'shipped' / 'delivered' / 'cancelled'
);
```
解説：フラグの組み合わせは指数的に増え、「発送済みなのに未払い」のような矛盾した状態を許してしまいます。状態は単一のステータス列で排他的に表現し、遷移をアプリ側で制御します。履歴が要るならステータス変更履歴テーブルを別に持ちます。

### No.84 日時を文字列で保存する
**❌ 悪い例**
```sql
CREATE TABLE events (
  id BIGINT PRIMARY KEY,
  occurred_at VARCHAR(19)  -- '2027/01/15 10:00:00' as text
);
SELECT * FROM events
WHERE occurred_at BETWEEN '2027/01/01' AND '2027/01/31';  -- format-dependent
```
**✅ 改善例**
```sql
CREATE TABLE events (
  id BIGINT PRIMARY KEY,
  occurred_at DATETIME NOT NULL
);
SELECT * FROM events
WHERE occurred_at >= '2027-01-01' AND occurred_at < '2027-02-01';
```
解説：文字列の日時は並び順・比較・期間計算・妥当性検証のすべてが書式依存になります。日時型なら日付関数や範囲条件が正しく使え、不正な値も入りません。表示形式はアプリ側の責務であり、保存形式と混ぜないことが大切です。

### No.85 金額を浮動小数点（FLOAT / DOUBLE）で持つ
**❌ 悪い例**
```sql
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  total_amount DOUBLE  -- 0.1 + 0.2 != 0.3 in binary floating point
);
SELECT SUM(total_amount) FROM orders;  -- tiny errors accumulate
```
**✅ 改善例**
```sql
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  total_amount DECIMAL(12,2) NOT NULL
);
-- or store the minor unit as integer (e.g. cents / yen)
```
解説：二進浮動小数点は10進の金額を正確に表せず、集計や突合で「1円合わない」事故を生みます。金額は DECIMAL か、最小単位の整数で保持します。これは DB だけでなくアプリ側の言語の型選びにも共通する原則です。

### No.86 電話番号・郵便番号を数値型で持つ
**❌ 悪い例**
```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY,
  postal_code INT,      -- 0600001 loses nothing, but 060-0001 format is lost;
                        -- leading zeros vanish for codes like 00123 areas
  phone BIGINT          -- '+', '-', and leading 0 cannot be represented
);
```
**✅ 改善例**
```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY,
  postal_code CHAR(7),        -- digits only, as text
  phone VARCHAR(20)           -- normalized digits, as text
);
```
解説：「数字だけの文字列」と「数値」は別物です。先行ゼロや区切りを含む識別子は文字列で持ち、必要ならチェック制約で桁や文字種を検証します。計算しない値に数値型を使う理由はありません。

### No.87 主キーがない、または巨大な自然キー
**❌ 悪い例**
```sql
-- No primary key: duplicates and slow updates
CREATE TABLE order_items (
  order_id BIGINT, product_id BIGINT,
  quantity INT, unit_price DECIMAL(10,2)
);
-- Or: PK = (email, company_code, branch_code, ...) long composite
```
**✅ 改善例**
```sql
CREATE TABLE order_items (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  order_id BIGINT NOT NULL,
  product_id BIGINT NOT NULL,
  quantity INT NOT NULL,
  unit_price DECIMAL(10,2) NOT NULL,
  UNIQUE KEY (order_id, product_id)
);
```
解説：主キーがないテーブルは重複行の排除も行の特定も困難です。複合自然キーを主キーにすると、参照側すべてにその列群が伝播します。単一のサロゲートキーを主キーにし、自然キーの一意性は UNIQUE 制約で担保するのが扱いやすい形です。

### No.88 ステータスを自由入力の文字列にする
**❌ 悪い例**
```sql
-- 'paid', 'Paid', 'PAID', 'paid ' all coexist
SELECT COUNT(*) FROM orders WHERE status = 'paid';  -- undercounts
```
**✅ 改善例**
```sql
ALTER TABLE orders
  ADD CONSTRAINT chk_orders_status
  CHECK (status IN ('new', 'paid', 'shipped', 'delivered', 'cancelled'));
-- plus a code table / master if statuses carry attributes
```
解説：値の表記ゆれは集計漏れの直接原因です。CHECK 制約や区分マスタで取りうる値を閉じ、アプリ側も定数・列挙型で扱います。「文字列だから自由」という設計は、データ品質の問題を未来に先送りします。
補足：MySQL の CHECK 制約は 8.0.16 以降で実際に強制されます。それ以前は ENUM やトリガーで代替していました。

### No.89 NULL 許可だらけでデフォルトもない
**❌ 悪い例**
```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY,
  name VARCHAR(100),          -- NULL names appear later
  status VARCHAR(20),         -- NULL status breaks every filter
  point INT                   -- NULL arithmetic yields NULL
);
```
**✅ 改善例**
```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'active',
  point INT NOT NULL DEFAULT 0
);
```
解説：NULL は「未設定なのか、設定できないのか」を曖昧にし、あらゆる比較・集計に例外処理を強います。原則は NOT NULL＋意味のあるデフォルトで、「本当に値が存在しないこと自体が意味を持つ」列だけを NULL 許可にします。設計時のこの判断が、後段の全クエリを単純にします。

### No.90 更新で履歴を消してしまう
**❌ 悪い例**
```sql
-- Price changes overwrite the past; disputes cannot be verified
UPDATE products SET price = 1800 WHERE id = 101;
-- orders placed yesterday no longer match any record
```
**✅ 改善例**
```sql
-- Snapshot what matters on the transaction itself,
-- and keep history for master data
CREATE TABLE order_items (
  id BIGINT PRIMARY KEY,
  order_id BIGINT NOT NULL,
  product_id BIGINT NOT NULL,
  unit_price DECIMAL(10,2) NOT NULL  -- price at purchase time
);
CREATE TABLE product_price_history (
  product_id BIGINT NOT NULL,
  price DECIMAL(10,2) NOT NULL,
  effective_from DATETIME NOT NULL
);
```
解説：マスタの現在値だけでは過去の取引を再現できません。取引時点の値を明細側にスナップショットとして持ち、重要な変更は履歴テーブルに残します。「なぜこの金額なのか」を後から説明できることが、データの信頼性です。
