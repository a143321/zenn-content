---
title: "SELECT文まとめ：基礎構文からウィンドウ関数・CTE まで"
emoji: "📊"
type: "tech"
topics: ["sql", "データエンジニア", "ウィンドウ関数", "初心者"]
published: true
---

## はじめに

SQL の SELECT 文で使う構文を、基礎から応用まで整理したまとめ記事です。
特に**ウィンドウ関数**と **CTE（WITH句）** は理解しづらいポイントが多いので、図解やつまずきポイントを交えて解説しています。

### SQL の3分類

| 分類 | 正式名 | 主なコマンド | 役割 |
|------|--------|-------------|------|
| **DQL** | Data Query Language | `SELECT` | データの**取得・分析** |
| **DML** | Data Manipulation Language | `INSERT` / `UPDATE` / `DELETE` | データの**追加・変更・削除** |
| **DDL** | Data Definition Language | `CREATE TABLE` / `DROP` / `ALTER` | テーブル構造の**定義・変更** |

この記事は **DQL（SELECT文）** に特化しています。DML/DDL については別記事で扱う予定です。

:::message
記事中のサンプルは DuckDB で動作確認していますが、構文自体は標準 SQL に準拠しているので他の DB でもほぼそのまま使えます。
:::

---

## SQL 実行順序（最重要！）

まずこれを暗記する。ウィンドウ関数や HAVING の理解に直結します。

```
① FROM / JOIN     — テーブルを結合
② WHERE           — 行をフィルタ（個別の行）
③ GROUP BY        — グループ化
④ HAVING          — グループをフィルタ（集計結果で絞る）
⑤ SELECT          — 列を選択 ＋ ウィンドウ関数を実行
⑥ ORDER BY        — 並び替え
⑦ LIMIT           — 件数制限
```

:::message alert
**ウィンドウ関数は⑤で実行される** → `WHERE`（②）や `HAVING`（④）では使えない。
ウィンドウ関数の結果でフィルタしたい場合は **CTE で先に計算 → 外側で WHERE** する。
:::

---

## 基礎構文の整理

まず基本的な構文を確認しておく。

### IN / NOT IN — 複数条件の指定

```sql
-- OR の短縮形
WHERE category IN ('food', 'drink')

-- 除外
WHERE category NOT IN ('stationery')
```

⚠ **NOT IN の罠**: サブクエリの結果に NULL が含まれると全行が除外される → `NOT EXISTS` の方が安全。

### BETWEEN — 範囲指定（両端を含む）

```sql
WHERE sale_date BETWEEN '2025-03-03' AND '2025-03-06'
WHERE price BETWEEN 300 AND 900
```

### LIKE — パターンマッチ

| パターン | 意味 |
|---------|------|
| `'%Tea%'` | Tea を含む |
| `'%Tea'` | Tea で終わる |
| `'Tea%'` | Tea で始まる |
| `'Store-_'` | Store- + 任意の1文字 |

`%` = 0文字以上、`_` = ちょうど1文字。

### DISTINCT — 重複排除

```sql
-- 1列
SELECT DISTINCT category FROM products;

-- 複数列（組み合わせで一意化）
SELECT DISTINCT store_id, time_slot FROM sales;
```

### WHERE vs HAVING

```sql
-- WHERE: GROUP BY の前（行レベルのフィルタ）
WHERE sale_date >= '2025-03-01'

-- HAVING: GROUP BY の後（集計結果のフィルタ）
HAVING SUM(quantity) >= 1000
```

### EXISTS vs IN

```sql
-- EXISTS: 行が存在するかだけ判定（1行見つけたら即終了 → 高速）
WHERE EXISTS (SELECT 1 FROM sales s WHERE p.product_id = s.product_id)

-- NOT EXISTS: NULL安全、大規模データ向き → 推奨
WHERE NOT EXISTS (SELECT 1 FROM ... WHERE ...)
```

`SELECT 1` は慣例。EXISTS は「行が存在するか」だけを見るので、何を返すかは関係ない。

### COALESCE — NULL 置換

```sql
-- 最初の非NULL値を返す
COALESCE(LAG(quantity) OVER (...), 0)  -- NULL なら 0 に置換
```

語源: coalesce =「合体する・1つにまとまる」（ラテン語）。

### != と <>

```sql
WHERE category <> 'stationery'   -- SQL標準（こちら推奨）
WHERE category != 'stationery'   -- 非標準だがほぼ全DBで動く
```

### CASE WHEN — 条件分岐

```sql
SELECT
    product_name, price,
    CASE
        WHEN price < 500 THEN 'Low'
        WHEN price < 1000 THEN 'Mid'
        ELSE 'High'
    END AS price_rank
FROM products;
```

上から順に評価され、最初にマッチした `THEN` の値を返す。どれにもマッチしなければ `ELSE`（省略時は NULL）。
`SELECT` だけでなく `ORDER BY` や `GROUP BY` の中でも使える。

### JOIN の種類

```sql
-- INNER JOIN: 両方にあるデータだけ
SELECT * FROM orders o
INNER JOIN products p ON o.product_id = p.product_id;

-- LEFT JOIN: 左テーブルは全行残す（右に一致がなければ NULL）
SELECT * FROM products p
LEFT JOIN orders o ON p.product_id = o.product_id;

-- CROSS JOIN: 全行 × 全行（1行のテーブルを全行に付与する時に便利）
SELECT * FROM sales
CROSS JOIN (SELECT AVG(quantity) AS avg_qty FROM sales) overall;
```

| 種類 | マッチなし時 | 主な用途 |
|------|-------------|---------|
| INNER JOIN | 行が消える | 関連データの結合 |
| LEFT JOIN | NULL で残る | 「存在しない」データの検出 |
| CROSS JOIN | 全組み合わせ | 全体平均を各行に付与 |

### UNION / UNION ALL — 縦結合

```sql
-- UNION ALL: 重複行もそのまま結合（高速）
SELECT product_name, price FROM products WHERE category = 'food'
UNION ALL
SELECT product_name, price FROM products WHERE category = 'drink';

-- UNION: 重複行を除去（遅い）
```

⚠ 両方の `SELECT` の**列数と型が一致**する必要がある。重複がないとわかっていれば `UNION ALL` を使う。

---

## ウィンドウ関数の基本

### GROUP BY との違い

| | GROUP BY | ウィンドウ関数 |
|---|----------|--------------|
| 行の扱い | グループごとに1行に**潰す** | 元の行を**保ったまま**集計値を追加 |
| 結果 | 行数が減る | 行数は変わらない |

### 基本構文

```sql
関数名() OVER (
    PARTITION BY グループ列   -- ① 壁を作る（どのグループ内で計算するか）
    ORDER BY 並び順           -- ② 方向を決める（窓がスライドする順序）
    ROWS BETWEEN ... AND ...  -- ③ 窓の大きさ
)
```

### 3つの句の役割 — 「覗き窓」のイメージ

#### ① PARTITION BY — 絶対に混ざらない壁

データ全体を独立したグループに分割する。別グループのデータは**絶対に混ざらない**。

```
┌── store_id=1 ──┐  ┌── store_id=2 ──┐  ┌── store_id=3 ──┐
│ 3/1  500        │  │ 3/1  300        │  │ 3/3  800        │
│ 3/2  510        │  │ 3/2  320        │  │ 3/4  600        │
│ ...             │  │ ...             │  │ ...             │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

#### ② ORDER BY — 窓がスライドする方向

:::message alert
**最大の落とし穴**: OVER 内の `ORDER BY` は表示順ではなく**計算ロジックそのもの**。

| 場所 | `ORDER BY` の役割 |
|------|------------------|
| クエリ末尾 | 表示順を決めるだけ。計算結果に影響しない |
| `OVER()` の中 | **計算ロジックそのもの**。窓がスライドする方向を決める |
:::

#### ③ ROWS BETWEEN — 窓の大きさ

```
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW = 直近3行

行1: 3/1  500  ← 窓 [500]           → AVG = 500.0
行2: 3/2  480  ← 窓 [500, 480]      → AVG = 490.0
行3: 3/3  510  ← 窓 [500, 480, 510] → AVG = 496.67
行4: 3/4  200  ← 窓 [480, 510, 200] → AVG = 396.67  ← 窓がスライド
```

### よく使うフレーム範囲

| フレーム | 範囲 | 用途 |
|---------|------|------|
| `UNBOUNDED PRECEDING ... CURRENT ROW` | 先頭〜自分 | 累積合計 |
| `2 PRECEDING ... CURRENT ROW` | 2行前〜自分 | 移動平均 |
| `1 PRECEDING ... 1 FOLLOWING` | 前後1行+自分 | 中央移動平均 |
| 省略 | 関数が自動決定 | RANK, LAG 等 |

---

## ウィンドウ関数の種類

### 集計系（引数あり）

```sql
SUM(quantity) OVER (...)   -- 累積合計
AVG(quantity) OVER (...)   -- 移動平均
```

### 順位系（引数なし、ORDER BY が実質的な引数）

```sql
RANK()       OVER (ORDER BY price DESC)  -- 同率同順位、次を飛ばす (1,1,3)
DENSE_RANK() OVER (ORDER BY price DESC)  -- 同率同順位、飛ばさない (1,1,2)
ROW_NUMBER() OVER (ORDER BY price DESC)  -- 必ず一意の連番 (1,2,3)
```

### 行参照系（引数あり）

```sql
LAG(col)              -- 1行前の値
LAG(col, 2)           -- 2行前の値
LEAD(col)             -- 1行後の値
LAG(col IGNORE NULLS) -- NULLをスキップして前の非NULL値を取得
```

---

## CTE（Common Table Expression）

### 基本構文

```sql
WITH cte_name AS (
    SELECT ... FROM ...
)
SELECT * FROM cte_name;
```

### 2段 CTE（カンマで繋ぐ）

```sql
WITH step1 AS (
    SELECT ... FROM ...
),                          -- ← カンマで繋ぐ（WITH は1回だけ）
step2 AS (
    SELECT ... FROM step1   -- ← 前の CTE を参照可能
)
SELECT * FROM step2;
```

### なぜ CTE が必要か

SQL の実行順序上、ウィンドウ関数の結果は `WHERE` で使えない。

```sql
-- ❌ これはエラー
SELECT *, RANK() OVER (...) AS rnk
FROM sales
WHERE rnk = 1;

-- ✅ CTE で先に計算 → 外側でフィルタ
WITH ranked AS (
    SELECT *, RANK() OVER (...) AS rnk
    FROM sales
)
SELECT * FROM ranked WHERE rnk = 1;
```

---

## 実践パターン集

### パターン1: 累積合計

```sql
SUM(quantity) OVER (
    PARTITION BY store_id
    ORDER BY sale_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
) AS cumulative_qty
```

### パターン2: 3日移動平均

```sql
ROUND(AVG(quantity) OVER (
    PARTITION BY store_id
    ORDER BY sale_date
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
), 2) AS ma_3day
```

### パターン3: CTE + RANK で TOP N 抽出

よく使うパターン。GROUP BY で集計 → RANK で順位 → WHERE でフィルタ。

```sql
WITH ranked AS (
    SELECT
        store_id, sale_date,
        ROUND(100.0 * SUM(return_qty) / SUM(quantity), 2) AS return_rate_pct,
        RANK() OVER (
            PARTITION BY store_id
            ORDER BY 1.0 * SUM(return_qty) / SUM(quantity) DESC
        ) AS rnk
    FROM sales
    GROUP BY store_id, sale_date
)
SELECT store_id, sale_date, return_rate_pct
FROM ranked
WHERE rnk = 1
ORDER BY store_id;
```

:::message
**注意**: 同じ SELECT 内で定義した別名は他の列で参照できない（標準SQL）。  
`ORDER BY return_rate_pct DESC` ではなく、計算式を再記述する必要がある。
:::

### パターン4: LAG IGNORE NULLS（2段CTE）

異常値の直前の「正常値」を取得したい場合。3つの罠がある。

| 罠 | 問題 | 解法 |
|----|------|------|
| WHERE先行 | 正常行が消えて LAG が参照できない | CTE内でフィルタなしに LAG → 外側でフィルタ |
| 連続異常 | 普通の LAG は直前の異常値を返す | `IGNORE NULLS` で正常値まで遡る |
| 構文エラー | `LAG(CASE WHEN ... IGNORE NULLS)` は不可 | 先に CASE WHEN で列を作り、次の CTE で LAG |

```sql
WITH prepared AS (
    SELECT *,
           CASE WHEN NOT is_irregular THEN amount END AS normal_amount
    FROM transactions
),
lagged AS (
    SELECT *,
           LAG(normal_amount IGNORE NULLS) OVER (
               PARTITION BY store_id, payment_type
               ORDER BY transaction_at
           ) AS prev_normal_amount
    FROM prepared
)
SELECT transaction_at, store_id, payment_type, amount, prev_normal_amount
FROM lagged
WHERE is_irregular = true
ORDER BY store_id, transaction_at;
```

### パターン5: SUM() OVER() でシェア計算

```sql
-- PARTITION BY なし + ORDER BY なし = テーブル全体の合計
ROUND(100.0 * total_sales / SUM(total_sales) OVER(), 2) AS share_pct
```

---

## つまずきポイント集

### 1. OVER 内の ORDER BY は「ロジック」

クエリ末尾の `ORDER BY` は表示順だけだが、`OVER()` 内の `ORDER BY` は計算結果を左右する。移動平均で `ORDER BY quantity` にしたら、時系列の移動平均にならなかった。

### 2. GROUP BY を使ったら集約関数が必要

`GROUP BY store_id, sale_date` の中で素の `return_qty` は使えない。`SUM(return_qty)` にする必要がある。

### 3. NOT IN は NULL に弱い

サブクエリが NULL を含むと全行が除外される。`NOT EXISTS` の方が安全。

### 4. CASE WHEN で「行を消さずに列を追加」

WHERE でフィルタすると行が消える。CASE WHEN なら行を残したまま条件付きの列を追加できる。

### 5. CTE の WITH は1回だけ

```sql
-- ❌ WITH を2回書くとエラー
WITH a AS (...) WITH b AS (...) SELECT ...

-- ✅ カンマで繋ぐ
WITH a AS (...), b AS (...) SELECT ...
```

---

## まとめ

| レベル | テーマ | キーワード |
|--------|--------|-----------|
| 基礎 | フィルタ・集計 | IN, BETWEEN, LIKE, DISTINCT, HAVING, EXISTS |
| 中級 | ウィンドウ関数 | SUM/AVG OVER, RANK, LAG, ROWS BETWEEN |
| 応用 | CTE + 複合 | WITH句, CROSS JOIN, IGNORE NULLS, 2段CTE |

ウィンドウ関数は「**覗き窓を通して集計する**」イメージで理解する。

- **PARTITION BY** = 絶対に混ざらない壁
- **ORDER BY** = 窓がスライドする方向（ロジック！）
- **ROWS BETWEEN** = 窓の大きさ

手を動かして覚えるのが一番。DuckDB ならインストール不要でローカルですぐ試せるのでおすすめです。
