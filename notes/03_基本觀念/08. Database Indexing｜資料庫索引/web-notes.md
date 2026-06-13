# Database Indexing｜資料庫索引

> [[indexing|索引 (Indexing)]] 就像書本的目錄:讓你快速找到資料,而不用從頭翻整張表。代價是多佔空間、寫入變慢——所以索引建得「剛剛好」才好。不同資料形狀(等值、範圍、地理、文字)需要**不同的索引結構**,這課後半就是在比較這五種。

## 1. 為什麼需要索引

- 加快查詢速度:`SELECT ... WHERE` 或 [[join|JOIN]] 時快速定位。
- 減少掃描整張表的成本,避免 [[full-table-scan|Full Table Scan]]。
- 提升排序與搜尋效率,例如 `ORDER BY` 或前綴搜尋 `LIKE 'abc%'`。

**沒有索引時**:對 `WHERE email = '...'`,得做 [[full-table-scan|Full Table Scan]] 逐筆比對,效率隨筆數增加而下降,複雜度 [[big-o-n|O(n)]]。

**建立索引後**(`CREATE INDEX idx_email ON users(email);`):透過索引快速定位(像查目錄找頁碼),查詢時間接近 [[big-o-logn|O(log n)]]。

## 2. 索引的取捨 (Trade-offs)

索引不是免費的:

- 佔用額外儲存空間。
- 寫入([[write-amplification|INSERT/UPDATE/DELETE]])變慢,因為每次寫入都要同步更新索引。
- 建太多索引反而拖累效能。

> 一句話心法:**索引拿「寫入 + 空間」換「讀取速度」**。讀多寫少 → 多建;寫多讀少 → 謹慎。

---

## 3. 五種常見索引結構

| 結構 | 最擅長 | 不擅長 | 代表系統 |
|---|---|---|---|
| [[btree-index]] | 等值 + 範圍 + 排序(通用) | 超高寫入吞吐 | PostgreSQL / MySQL InnoDB 預設 |
| [[lsm-tree]] | 海量寫入 | 讀取(要查多層) | Cassandra / RocksDB / LevelDB |
| [[hash-index]] | 等值查詢 O(1) | 範圍、排序 | Redis、MySQL Memory 引擎 |
| [[geospatial-index]] | 二維地理鄰近查詢 | 一維資料 | PostGIS、Redis GEO、Uber H3 |
| [[inverted-index]] | 全文關鍵字搜尋 | 一般欄位查詢 | Elasticsearch / Lucene |

### 3-1. [[btree-index|B-Tree / B+Tree]] —— 通用王者

[[btree-index|B-Tree]](Balanced Tree,平衡樹)是關聯式資料庫**最常見**的索引,主鍵、唯一、普通索引大多用它或變體 [[bplus-tree|B+Tree]]。

- **平衡多叉樹**:根到所有葉節點路徑等長(不退化成鏈表),每個節點存多個排序好的 key,例 `[20 | 40]` → 小於 20 走左、20~40 走中、大於 40 走右。
- **磁碟友好**:節點大小對齊磁碟頁 (page),減少 I/O。
- **三種查詢全包**:等值 [[big-o-logn|O(log n)]]、範圍(`BETWEEN`/`>`/`<`,因 key 有序)、排序(`ORDER BY` 直接用,免再排)、前綴(`LIKE 'abc%'`)。

**B-Tree vs [[bplus-tree|B+Tree]]**:B-Tree 每個節點都能存資料;B+Tree **只有葉節點存資料**,內部節點只放索引 key,且**葉節點間用鏈結串起來** → 範圍掃描更快。MySQL InnoDB 用 B+Tree。

> PostgreSQL 建 `PRIMARY KEY` / `UNIQUE` 會**自動建 B-Tree 索引**,對唯一性檢查與 `product_id BETWEEN 100 AND 200` 範圍查詢都關鍵。

### 3-2. [[lsm-tree|LSM Tree]] —— 為寫入而生

[[btree-index|B-Tree]] 讀多寫少很棒,但每次寫入可能造成**隨機磁碟 I/O**,高併發大量寫入下吃力。[[lsm-tree|LSM Tree]](Log-Structured Merge-Tree)的設計初衷:**把隨機寫變成順序寫 (sequential write)**,大幅拉高寫入吞吐。

寫入三步:

1. **[[memtable|MemTable]]**:新寫入先進記憶體中的排序結構(通常是 SkipList)。
2. **[[wal|WAL 預寫日誌]]**:寫入前先順序記到 log,當機可重播恢復,防記憶體掉電丟資料。
3. **[[sstable|SSTable]]**:MemTable 滿了 flush 成不可變的磁碟檔(排序好的 key-value)。

之後 **[[compaction|Compaction 合併壓縮]]** 把多個 SSTable 合併、清掉舊版本,維持 Level 0/1/2… 層級——這就是「Merge」的由來。查詢時依序查 MemTable → 各層 SSTable,並靠每個 SSTable 附帶的 **[[bloom-filter|Bloom Filter]]** 快速判斷「key 在不在這層」避免無謂磁碟讀。

- **優點**:寫入吞吐極高、磁碟友好。
- **缺點**:讀取較慢(可能查多層)、Compaction 成本高、有 [[write-amplification|寫放大]]。
- **場景**:time-series、log 系統、Cassandra / HBase、RocksDB / LevelDB。

### 3-3. [[hash-index|Hash Index]] —— 等值查詢之王

用 [[hash-function|Hash 函數]] 把 key 映射到固定的 bucket,等值查詢幾乎 [[big-o-1|O(1)]]。

```
WHERE id = 12   // hash(12)=0 → 直接到 Bucket 0 → 找到  O(1)
WHERE id > 10   // 順序被打亂,得掃所有 bucket        O(n)
```

- **結構**:Hash 函數 + Hash Table(多個 bucket)+ [[collision-handling|碰撞處理]]。兩個 key 落同一 bucket 時用 **Chaining(鏈結法)** 或 **Open Addressing(開放位址法)** 解決。
- **致命限制**:**不支援範圍查詢與排序**(雜湊把 key 的順序性破壞了)。
- **場景**:Redis 底層 hash 結構、MySQL Memory/HEAP 引擎、分散式系統用 `hash(key)` 決定 sharding 節點。

對照 [[btree-index|B-Tree]]:等值 Hash 贏(O(1) vs O(log n)),但範圍 / 排序只有 B-Tree 能做,所以**通用性 B-Tree 完勝**。

### 3-4. [[geospatial-index|Geospatial Index]] —— 二維鄰近搜尋

「附近 5 公里的餐廳 / 司機」這類題目。一般 [[btree-index|B-Tree]] / [[hash-index|Hash]] 針對**一維** key,但地理是**二維**(經度、緯度)。

**為什麼不能用兩個一維 B-Tree(各建在 lat、lng)?** 只用緯度 → 找到一條橫跨全球的長條帶;兩個索引做交集 → 形成一個**矩形**,比真正要的「圓形半徑」大得多,還要再過濾。所以要專門的空間索引:

- **[[geohash|Geohash]]**:把經緯度壓成一維 Base32 字串,前綴越長越精確(`dr` → `dr5` → `dr5ru`)。相近地點通常共享前綴 → **直接用現成 B-Tree 做前綴/範圍查詢**。缺點:邊界效應(馬路兩側可能前綴完全不同)。Redis GEO、MongoDB 用它。
- **[[quadtree|QuadTree]]**:空間遞迴切四象限(NE/NW/SE/SW),密集區切更細。概念是 R-tree 基礎,生產少見。
- **[[r-tree|R-Tree]]**:用**彈性、可重疊的矩形 (MBR)** 依實際資料分布組織,**同一索引能同時處理點與多邊形**,磁碟友好。**現代生產標準**(PostGIS、SQLite)。
- **[[s2-h3|S2 / H3]]**:Google S2 把地球投影到立方體切四邊形;Uber H3 投影到二十面體切**六邊形**(分布均勻,適合鄰近搜尋與聚合)。最後都再用 Haversine 精確距離過濾。

### 3-5. [[inverted-index|Inverted Index]] —— 全文搜尋核心

要找「包含單字 apple 的文件」,逐一掃描太慢。[[inverted-index|倒排索引]] 建立「**單字 → 出現在哪些文件**」的映射(相對於 [[forward-index|正排索引]] 的「文件 → 單字」)。

```
apple  → [Doc1, Doc3]
banana → [Doc1, Doc2]
dog    → [Doc2, Doc3]
查 "apple" AND "dog" → posting list 交集 → Doc3
```

- **結構**:[[term-dictionary|Term Dictionary 詞典]](所有單字)+ [[posting-list|Posting List 倒排表]](每個 term 對應它出現的文件,可含詞頻 TF、位置 positions)。
- **查詢**:單詞直接取 list;AND = 交集;OR = 聯集;短語 `"apple pie"` 靠位置資訊(apple 位置 +1 = pie)。
- **缺點**:建立要 tokenize / 正規化(斷詞、轉小寫、去 stop words),儲存大(需壓縮),更新昂貴。
- **場景**:Elasticsearch / Lucene / Solr、PostgreSQL `to_tsvector()`、MySQL `FULLTEXT`。

---

## 4. 收尾小考

1. 沒索引 vs 有索引,查詢方式與複雜度各是什麼?
2. 索引有哪些缺點?(至少兩個)
3. [[btree-index|B-Tree]] 和 [[bplus-tree|B+Tree]] 差在哪?MySQL InnoDB 用哪種?
4. 為什麼 [[hash-index|Hash Index]] 不能做 `WHERE age > 25`?
5. 寫入吞吐極高的場景該選 B-Tree 還是 [[lsm-tree|LSM Tree]]?為什麼?
6. 「附近的餐廳」為什麼不能用兩個一維 B-Tree(lat、lng)解決?
7. 全文搜尋 `MATCH 'apple'` 背後是哪種索引?它的核心映射是什麼?

```glossary
{
  "indexing": { "term": "索引 (Indexing)", "short": "資料庫中像書本目錄的結構,讓查詢快速定位資料而不用掃整張表;代價是佔空間、拖慢寫入。不同資料形狀需不同索引結構。" },
  "join": { "term": "JOIN｜表格關聯", "short": "把多張表依關聯欄位組合的查詢;關聯欄位有 [[indexing|索引]] 時快很多。" },
  "full-table-scan": { "term": "Full Table Scan｜全表掃描", "short": "沒有適用索引時逐筆比對整張表,複雜度 [[big-o-n|O(n)]],筆數越多越慢。" },
  "big-o-n": { "term": "O(n)｜線性複雜度", "short": "成本隨資料筆數 n 成正比;[[full-table-scan|全表掃描]]就是 O(n)。" },
  "big-o-logn": { "term": "O(log n)｜對數複雜度", "short": "成本隨資料量呈對數成長,遠快於 [[big-o-n|O(n)]];[[btree-index|B-Tree]] 查詢可接近此速度。" },
  "big-o-1": { "term": "O(1)｜常數複雜度", "short": "不論資料多大,查詢成本固定;[[hash-index|Hash Index]] 的等值查詢接近 O(1)。" },
  "write-amplification": { "term": "寫放大｜Write Amplification", "short": "實際寫入磁碟的量遠大於原始資料量。索引越多寫入越慢;[[lsm-tree|LSM Tree]] 因 [[compaction|Compaction]] 把同筆資料多次寫入不同層,寫放大尤其明顯。" },
  "btree-index": { "term": "B-Tree Index｜B 樹索引", "short": "最通用的平衡多叉樹索引,擅長等值 [[big-o-logn|O(log n)]]、範圍、排序;PostgreSQL/MySQL 預設。磁碟頁對齊減少 I/O。" },
  "bplus-tree": { "term": "B+Tree｜B 加樹", "short": "B-Tree 變體:只有葉節點存資料,葉節點間有鏈結 → 範圍掃描更快。MySQL InnoDB 採用。" },
  "lsm-tree": { "term": "LSM Tree｜日誌結構合併樹", "short": "Log-Structured Merge-Tree:把隨機寫轉成順序寫以拉高寫入吞吐。先寫 [[memtable|MemTable]]+[[wal|WAL]],滿了 flush 成 [[sstable|SSTable]],再靠 [[compaction|Compaction]] 合併。寫多讀少首選。" },
  "memtable": { "term": "MemTable｜記憶體表", "short": "[[lsm-tree|LSM Tree]] 中接收新寫入的記憶體排序結構(常為 SkipList),滿了 flush 成 [[sstable|SSTable]]。" },
  "wal": { "term": "WAL｜Write-Ahead Log 預寫日誌", "short": "寫入前先順序記到 log,當機後可重播恢復,防止記憶體中的資料掉電遺失。" },
  "sstable": { "term": "SSTable｜Sorted String Table", "short": "[[lsm-tree|LSM Tree]] flush 到磁碟的不可變檔案,內含排序好的 key-value。" },
  "compaction": { "term": "Compaction｜合併壓縮", "short": "把多個 [[sstable|SSTable]] 合併、清掉舊版本、維持層級結構;是 LSM「Merge」的由來,但帶來 [[write-amplification|寫放大]]。" },
  "bloom-filter": { "term": "Bloom Filter｜布隆過濾器", "short": "機率性結構,快速判斷「key 一定不存在 / 可能存在」;每個 [[sstable|SSTable]] 附帶一個,避免無謂磁碟讀取。" },
  "hash-index": { "term": "Hash Index｜雜湊索引", "short": "用 [[hash-function|Hash 函數]] 把 key 映射到 bucket,等值查詢 [[big-o-1|O(1)]];但破壞順序,不支援範圍與排序。" },
  "hash-function": { "term": "Hash Function｜雜湊函數", "short": "把任意 key 轉成固定範圍數值 (hash code) 的函數,用來決定資料落在哪個 bucket。" },
  "collision-handling": { "term": "Collision Handling｜碰撞處理", "short": "兩個不同 key 雜湊到同一 bucket 時的解法:Chaining(鏈結法,bucket 存 list)或 Open Addressing(開放位址法,找鄰近空位)。" },
  "geospatial-index": { "term": "Geospatial Index｜地理空間索引", "short": "針對二維/多維地理座標(經緯度)的索引,解決「附近 N 公里」這類鄰近查詢;一維 B-Tree 無法有效處理。" },
  "geohash": { "term": "Geohash｜地理雜湊", "short": "把經緯度壓成一維 Base32 字串,前綴越長越精確、相近點共享前綴 → 可直接用 [[btree-index|B-Tree]] 範圍查;缺點是邊界效應。" },
  "quadtree": { "term": "QuadTree｜四分樹", "short": "空間遞迴切四象限(NE/NW/SE/SW),密集區切更細;是 [[r-tree|R-Tree]] 概念基礎,生產較少見。" },
  "r-tree": { "term": "R-Tree｜矩形樹", "short": "用彈性可重疊的最小邊界矩形 (MBR) 依資料分布組織,能同時處理點與多邊形,磁碟友好;現代空間索引生產標準(PostGIS、SQLite)。" },
  "s2-h3": { "term": "S2 / H3｜球面網格索引", "short": "Google S2 投影到立方體切四邊形;Uber H3 投影到二十面體切六邊形(分布均勻,適合鄰近搜尋與聚合)。最後都用 Haversine 精確距離過濾。" },
  "inverted-index": { "term": "Inverted Index｜倒排索引", "short": "全文搜尋核心,建立「單字 → 出現在哪些文件」映射;查詢取 [[posting-list|posting list]] 並做交集/聯集。Elasticsearch/Lucene 底層。" },
  "forward-index": { "term": "Forward Index｜正排索引", "short": "「文件 → 單字」的映射,與 [[inverted-index|倒排索引]] 方向相反。" },
  "term-dictionary": { "term": "Term Dictionary｜詞典", "short": "[[inverted-index|倒排索引]] 中所有出現過的單字 (term) 的集合。" },
  "posting-list": { "term": "Posting List｜倒排表", "short": "每個 term 對應一個 list,記它出現在哪些文件,可含詞頻 (TF) 與位置 (positions,供短語查詢)。" }
}
```
