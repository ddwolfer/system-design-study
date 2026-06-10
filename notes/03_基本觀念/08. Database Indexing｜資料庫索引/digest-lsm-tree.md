# 08 索引 — LSM Tree(來源:`Hash Index.pdf`,檔名貼錯)

> 2026-06-03 蒸餾;2026-06-08 已入庫 KG(節點 `LSM Tree (Log-Structured Merge-Tree)` = 82f33e58,principle;contradicts B-Tree 8d99a971)。LSM = Log-Structured Merge-tree 日誌結構合併樹。

## 1. 為什麼需要 LSM-Tree
傳統 B-Tree 適合讀多寫少,但瓶頸:
- 每次寫入(INSERT/UPDATE/DELETE)可能造成**隨機磁碟 I/O**。
- 硬碟/SSD 對隨機寫比順序寫慢得多。
- 高併發、大量寫入下 B-Tree 更新效率不佳。

設計初衷:**把寫入 disk 變成 sequential writes(順序寫),大幅提升寫入效能**。

## 2. 核心概念與寫入流程
多層結構,寫入先進記憶體再批次寫磁碟:
1. **MemTable(記憶體結構,通常是跳表 SkipList 或平衡樹)**:新寫入(PUT/UPDATE/DELETE)先進 MemTable,資料排序好、支援快速查詢。
2. **Write-Ahead Log (WAL,預寫日誌)**:為避免記憶體掉電丟失,每次寫入前先記到 WAL(順序寫檔),當機後可重播恢復。
3. **SSTable(Sorted String Table,磁碟檔案)**:MemTable 滿了 flush 成新的 SSTable(不可變),裡面是排序好的 key-value。

**Compaction(合併壓縮)**:磁碟累積很多 SSTable → 合併多個 SSTable、刪舊版本/刪除標記、保持層級結構(Level 0/1/2…,越下層檔案越大越少)。這就是「Merge」的由來。

## 3. 查詢流程
依序查:① MemTable ② Immutable MemTable(正在 flush) ③ Level 0 SSTables ④ Level 1、2… 直到找到或確定不存在。加速:每個 SSTable 有 **Bloom Filter**(快速判斷 key 是否存在)、index block、block cache。

## 4. 優缺點
- **優點**:寫入效能高(先寫記憶體、之後順序寫磁碟)、磁碟友好(SSD/HDD)、支援大規模高寫入吞吐。
- **缺點**:讀取開銷高(可能查多層 SSTable)、Compaction 成本高(大量 I/O)、**寫放大 (Write Amplification)**(同一筆資料因合併被多次寫入不同層)。

## 5. 使用場景
寫多讀少(time-series DB、log 系統)、分散式 KV-Store(Cassandra、HBase)、嵌入式儲存引擎(LevelDB、RocksDB)、區塊鏈儲存層(Bitcoin Core UTXO DB)。

### 自我測驗
- **Q1:** B-Tree vs LSM Tree 各適合什麼?→ B-Tree 讀為主(O(log n) 直接定位);LSM 寫為主(隨機寫轉順序寫,寫吞吐高,但讀要查多層)。
- **Q2:** LSM 寫入三關鍵步驟?→ MemTable(記憶體排序結構)、WAL(防丟失)、SSTable(flush 成不可變磁碟檔)。
- **Q3:** Write Amplification?Bloom Filter 角色?→ 寫放大:同筆資料因 Compaction 被多次寫入不同層,實際寫入量遠大於原始;Bloom Filter:每個 SSTable 附帶的機率結構,快速判斷 key 是否存在,避免不必要磁碟讀取。
