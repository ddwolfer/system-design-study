# 03_基本觀念 / 08. Database Indexing｜資料庫索引 — 總覽 (INDEX MAP)

> 2026-06-03 重新確認:這課資料夾有 **6 個 PDF**,但**檔名與內容對不上**(貼錯)。實際主題對照:

| 來源檔名 | 實際主題 | digest 檔 |
|---|---|---|
| `Database Indexing.pdf` | 索引總論 | 本檔(下方)|
| `B+ Tree.pdf` | B-Tree / B+Tree | `digest-btree.md` |
| `Hash Index.pdf` | LSM Tree | `digest-lsm-tree.md` |
| `Bitmap Index.pdf` | Hash Index | `digest-hash-index.md` |
| `Covering Index.pdf` | Geospatial Index | `digest-geospatial-index.md` |
| `Index Selection.pdf` | Inverted Index | `digest-inverted-index.md` |

呼叫:`gemini_digest_pdf(lesson, file="<來源檔名去掉.pdf>")`(server 已支援 `file` 參數)。

---

## 索引總論(來源:`Database Indexing.pdf`)

**什麼是 Indexing(索引)?** 在資料庫中,索引就像書本的目錄,讓你快速找到資料,而不用從頭到尾翻整張表。

**為什麼需要索引?**
- 加快查詢速度:`SELECT ... WHERE` 或 `JOIN` 時快速定位。
- 減少掃描整張表的成本(避免 Full Table Scan)。
- 提升排序與搜尋效率,例如 `ORDER BY` 或前綴搜尋 `LIKE 'abc%'`。

**沒有索引**:對 `WHERE email = '...'`,沒索引就得 Full Table Scan 逐筆比對,效率隨筆數增加而下降(**O(n)**)。
**建立索引後**(`CREATE INDEX idx_email ON users(email);`):透過索引快速定位(像查目錄找頁碼),查詢時間接近 **O(log n)**。

**索引的缺點**:
- 佔用額外儲存空間。
- 寫入(INSERT/UPDATE/DELETE)變慢,因為每次寫入都要更新索引。
- 建太多索引反而拖累效能。

**常見索引種類**:B-Tree Index、LSM Tree、Hash Index、Geospatial Index、Inverted Index。

### 自我測驗
- **Q1:** 沒索引 vs 有索引怎麼查?→ 沒索引:Full Table Scan 逐筆比對,O(n);有索引:快速定位,接近 O(log n)。
- **Q2:** 索引缺點?→ 佔空間、寫入變慢(要更新索引)、太多索引拖累效能。
- **Q3:** 索引能加速哪些操作?→ `WHERE` 條件、`JOIN`、`ORDER BY`、前綴搜尋 `LIKE 'abc%'`。
