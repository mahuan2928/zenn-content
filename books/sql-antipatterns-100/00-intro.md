---
title: "はじめに"
free: true
---

# SQLアンチパターン100＋改善クエリ例集

遅い・壊れる・読みにくいSQLを直すための実務リファレンス

## はじめに

本書は、現場でよく見かけるSQLのアンチパターン100件を、悪い例と改善例のペアで整理したリファレンスです。扱うスキーマは、ECサイトを想定した users（ユーザー）、orders（注文）、order_items（注文明細）、products（商品）、categories（カテゴリ）を中心に統一しています。SQLは標準SQLをベースに書き、MySQL / PostgreSQL で考え方の異なる箇所には補足を付けました。実行計画の読み方ではなく「書き方のクセ」を直すことが目的です。自分のコードレビューや設計レビューのチェック材料としてご活用ください。

前提スキーマの要点：users(id, name, email, status, created_at, deleted_at)、orders(id, user_id, status, total_amount, created_at)、order_items(id, order_id, product_id, quantity, unit_price)、products(id, name, category_id, price, stock)、categories(id, name)。主キーは各テーブルの id です。

## 目次

- 第1章 SELECT・取得系（No.1〜10）
- 第2章 WHERE・絞り込み（No.11〜20）
- 第3章 JOIN（No.21〜30）
- 第4章 インデックス（No.31〜40）
- 第5章 集計・GROUP BY（No.41〜50）
- 第6章 サブクエリ（No.51〜60）
- 第7章 INSERT / UPDATE / DELETE（No.61〜70）
- 第8章 トランザクション・ロック（No.71〜80）
- 第9章 テーブル設計（No.81〜90）
- 第10章 NULL・型・日付の扱い（No.91〜100）
- 巻末チェックリスト
