# 08 索引 — B-Tree / B+Tree(來源:`B+ Tree.pdf`)

> 2026-06-03。已入庫 KG。

## 1. 什麼是 B-Tree Index
**B-Tree(Balanced Tree,平衡樹)**:資料庫常用的索引結構,加速查找、插入、刪除。特點:
- **平衡**:根到所有葉節點路徑等長,不會退化成鏈表。
- **多叉樹**:不是二元樹,每個節點可有多個 key 和子節點。
- **磁碟友好**:設計考慮磁碟頁 (page/block) 大小,減少磁碟 I/O。

MySQL/PostgreSQL/Oracle 的主鍵、唯一、普通索引大部分用 B-Tree 或變體 B+Tree。

## 2. 結構
節點含多個排序好的 Key + 子節點指標。例:`[20 | 40]` —— 小於 20 走左、20~40 走中、大於 40 走右。查找從根逐層往下到葉節點。
(示意:root `[20|40]` → 左葉 `[5,10]`、中葉 `[25,30,35]`、右葉 `[50,60,70]`。)

## 3. 特點
- **搜尋快**:O(log n)。找 30:`[20|40]`(介於走中)→ `[25|30|35]` 找到 ✓。
- **範圍查詢**:key 有序 → `BETWEEN`,`>`,`<`。
- **排序**:索引本身有序,`ORDER BY` 直接利用,免額外排序。
- **動態更新**:插入/刪除後自動調整保持平衡。

## 4. B-Tree vs B+Tree
- **B-Tree**:每個節點都可存資料 (data pointer)。
- **B+Tree**:只有葉節點存資料,內部節點只存索引 (key);葉節點間有**鏈結 (linked list)**,更適合範圍查詢。MySQL InnoDB 用 B+Tree。

## 5. 使用場景
PostgreSQL 幾乎都用 B-Tree(primary key、unique、regular index)。建表時 PRIMARY KEY(`product_id`)、UNIQUE(`sku`)會**自動建立 B-Tree 索引**,對唯一性檢查與範圍查詢(`product_id BETWEEN 100 AND 200`)都至關重要。DynamoDB sort key 是 B-Tree 變體;MongoDB 用 B+Tree(`db.products.createIndex({"sku":1})`)。

### 自我測驗
- **Q1:** B-Tree vs B+Tree?MySQL InnoDB 用哪種?→ B-Tree 所有節點存資料;B+Tree 只葉節點存資料、葉節點有鏈結更適合範圍掃描。InnoDB 用 **B+Tree**。
- **Q2:** B-Tree 支援哪些查詢?→ 等值 (`WHERE id=123`)、範圍 (`BETWEEN`/`>`/`<`)、排序 (`ORDER BY`)、前綴 (`LIKE 'abc%'`)。
- **Q3:** PostgreSQL 建 PRIMARY KEY / UNIQUE 自動建什麼索引?→ B-Tree。
