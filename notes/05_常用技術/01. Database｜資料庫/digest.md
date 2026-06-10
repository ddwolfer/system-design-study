# 05_常用技術 / 01. Database｜資料庫 — 總覽 (INDEX MAP)

> **⚠️ 課程素材包損壞警告(2026-06-08 確認)**
> 此資料夾原有 14 個 PDF,但經 Gemini 逐一實讀驗證:**檔名與內容嚴重不符,14 個檔其實只含 6 個真實主題**(各以繁/簡體 + 重複檔名出現)。使用者已確認「就這些了」——課程方並未提供其餘獨立 DB 主題。
>
> **檔名有、但內容完全找不到的 8 個 DB(課程缺檔):** SQL 本身、NoSQL 總論、Cassandra、MongoDB、NewSQL、BigTable、MySQL、Redis。日後若課程平台修好素材,再補讀。

## 6 個真實主題(已清理快取)

| # | 真實主題 | digest 檔 | 原始(亂貼)PDF 來源 | 投影片數 |
|---|---|---|---|---|
| 01 | 資料庫選型總論(各 DB 設計哲學/適用場景/面試選庫) | `digest-01-db-overview.md` | `SQL.pdf` / `Cassandra.pdf` / `Database.pdf` | 7 |
| 02 | **PostgreSQL**(索引/JSONB/PostGIS、WAL、複製、ACID 隔離層級) | `digest-02-postgresql.md` | `NoSQL.pdf` / `BigTable.pdf` | 15 |
| 03 | **DynamoDB**(分區鍵/排序鍵、GSI/LSI、一致性、DAX/Streams) | `digest-03-dynamodb.md` | `DynamoDB.pdf` / `NewSQL.pdf` | 13 |
| 04 | **OLTP vs OLAP**(列式/行式、星型 Schema、ETL/CDC/ELT、HTAP) | `digest-04-oltp-vs-olap.md` | `MongoDB.pdf` / `OLTP vs OLAP.pdf` | 11 |
| 05 | **Elasticsearch**(document/index/mapping、Lucene segment、inverted index) | `digest-05-elasticsearch.md` | `PostgreSQL.pdf` / `Redis.pdf` | 23 |
| 06 | **Vector Database**(embedding、KNN/ANN、HNSW/IVF/LSH、pgvector/Pinecone) | `digest-06-vector-database.md` | `MySQL.pdf` / `Elasticsearch.pdf` / `Vector Database.pdf` | 17 |

> 6 個 digest 均為 Gemini 逐字原文 + 圖描述(繁體為主;DynamoDB 來源檔為簡體)。**尚未入庫 KG**——日後在場上課時才依信任規則蒸餾。
> 清理紀錄(2026-06-08):原 11 個亂名 digest → 保留 6 個完整內容並正名為上表編號檔,刪除 5 個重複/截斷副本(cassandra/bigtable/newsql/redis/elasticsearch)。

## 呼叫 Gemini 重讀時的 file 對照(若需重抓某主題逐字原文)

因課程 PDF 檔名是亂的,要重讀「真實主題」時 `file` 要帶**亂貼的原始檔名**:
- 資料庫選型總論 → `file="Database"`(正名相符,最乾淨)
- PostgreSQL → `file="NoSQL"`
- DynamoDB → `file="DynamoDB"`(正名相符)
- OLTP vs OLAP → `file="OLTP vs OLAP"`(正名相符)
- Elasticsearch → `file="PostgreSQL"`
- Vector Database → `file="Vector Database"`(正名相符)
