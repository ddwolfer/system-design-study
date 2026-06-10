# 08 索引 — Inverted Index 倒排索引(來源:`Index Selection.pdf`,檔名貼錯)

> 2026-06-03 蒸餾;2026-06-09 已入庫 KG(節點 `Inverted Index` = 95a9da13,principle;aligns_to LSM Tree 82f33e58)。已重寫成網頁筆記 `web-notes-inverted-index.md`。全文搜尋的核心結構。

## 1. 為什麼需要 Inverted Index
要找「包含單字 apple 的文件」:沒索引就逐一掃描文件(成本 O(n × 文件大小))。Inverted Index 建立「**單字 → 出現在哪些文件**」的映射,查詢就能很快找到相關文件。

## 2. 核心概念
- **正排索引 (Forward Index)**:文件 → 單字
  ```
  Doc1 → {apple, banana, cat}
  Doc2 → {banana, dog}
  ```
- **倒排索引 (Inverted Index)**:單字 → 文件
  ```
  apple → [Doc1, Doc3]
  banana → [Doc1, Doc2]
  dog → [Doc2, Doc3]
  ```
  查 "apple" AND "dog" → posting list 交集 → Doc3。

## 3. 資料結構
1. **Term Dictionary(詞典)**:所有出現過的單字 (term)。
2. **Posting List(倒排表)**:每個 term 對應一個 list,存它在哪些文件出現。每個 entry (posting) 可含:文件 ID (docID)、詞頻 (TF)、詞在文件中的位置 (positions,用於 phrase query)。
   - 例:`apple → [(Doc1, pos=[2,7]), (Doc3, pos=[5])]`

## 4. 查詢流程
- **單詞**:查 apple → 直接取 posting list → [Doc1, Doc3]。
- **多詞 AND**:apple AND dog → 兩個 posting list **交集** → [Doc3]。
- **多詞 OR**:apple OR banana → **聯集**。
- **短語查詢 (phrase query)**:"apple pie" → posting list 有位置資訊,只有當 apple 位置 +1 = pie 位置才匹配。

## 5. 優缺點
- **優點**:查詢非常快(term → posting list 直接跳)、支援布林/短語檢索、適合全文搜索。
- **缺點**:建立成本高(要 tokenize、normalize:斷詞/轉小寫/去 stop words)、儲存成本高(posting list 大,需壓縮如 VarInt、Roaring Bitmap)、更新昂貴(每次增刪文件都要更新倒排表)。

## 6. 實際應用
搜尋引擎(Google、Elasticsearch、Lucene、Solr、OpenSearch)、資料庫全文搜尋(PostgreSQL `to_tsvector()`、MySQL `FULLTEXT` index 的 `MATCH...AGAINST`)、IDE/程式碼搜尋(VSCode、Sourcegraph)。

### 自我測驗
- **Q1:** Forward vs Inverted Index 核心差異?→ Forward:文件→單字;Inverted:單字→文件。後者讓全文搜尋快速找到含特定單字的文件。
- **Q2:** "apple AND dog" 怎麼運作?→ 取 apple 的 posting list [Doc1,Doc3]、dog 的 [Doc2,Doc3],取交集 → [Doc3]。
- **Q3:** 舉一個用 Inverted Index 的知名系統?→ Elasticsearch(底層 Lucene);還有 Solr、OpenSearch、PostgreSQL 全文搜尋、MySQL FULLTEXT。
