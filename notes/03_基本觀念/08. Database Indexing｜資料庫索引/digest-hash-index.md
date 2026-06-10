# 08 索引 — Hash Index(來源:`Bitmap Index.pdf`,檔名貼錯)

> 2026-06-03 蒸餾;2026-06-08 已入庫 KG(節點 `Hash Index` = 55bcfd51,principle;contradicts B-Tree 8d99a971、aligns_to Consistent Hashing)。已重寫成網頁筆記 `web-notes-hash-index.md`。

## 1. 什麼是 Hash Index
利用 Hash 函數把 key 映射到固定範圍的位置,加速查詢。
- 適合**等值查詢 (equality lookup)**,如 `id = 123`。
- **不適合範圍查詢 (`id > 100`) 或排序**,因為雜湊後 key 的順序性消失。

例:`hash(id) = bucket` → 直接定位 bucket → 找到資料。
```
WHERE id = 12 // hash(12)=0 → Bucket 0 → Bohr  O(1)
WHERE id > 10 // 必須掃描所有 bucket       O(n)
```

## 2. 結構
1. **Hash 函數**:把 key 轉成數值 (hash code)。
2. **Hash Table**:多個 bucket,每個指向一組資料記錄。
3. **Collision Handling(碰撞處理)**:兩個不同 key 雜湊到同一 bucket 時:
   - **Chaining(鏈結法)**:bucket 存 linked list/陣列放多筆。
   - **Open Addressing(開放位址法)**:往鄰近空位找。

例:`hash(key)=key%4`,Bucket 0 → [key=8→row#100],[key=12→row#250](key 8、12 碰撞)。

## 3. 查詢流程
查 `id=12`:① 算 `hash(12)=0` ② 去 Bucket 0 找(可能比對鏈結多筆) ③ 回傳 row 指標 → 讀資料頁。

## 4. 特點
- **優點**:等值查詢幾乎 O(1)、空間利用率高(設計得宜)、插入快(算 hash 放 bucket)。
- **缺點**:不支援範圍查詢(順序被打亂,`id>10` 要掃全部)、碰撞需處理、擴容成本高(bucket 太滿要 rehash)。

## 5. Hash Index vs B-Tree Index
| 特性 | Hash Index | B-Tree Index |
|---|---|---|
| 等值查詢 | 很快 O(1) | O(log n) |
| 範圍查詢 | 不支援 | 高效(有序)|
| 排序支援 | 不支援 | 支援 |
| 寫入開銷 | 較低 | 較高(需維持平衡)|
| 空間 | bucket 多時浪費 | 結構較緊湊 |

## 6. 實際應用
MySQL(InnoDB 預設 B+Tree;Memory/HEAP 引擎支援 Hash Index)、Redis(底層 hash 結構就是 Hash Table)、分散式系統 sharding 常用 `hash(key)` 決定資料在哪個節點。

### 自我測驗
- **Q1:**(是非)Hash Index 支援範圍查詢(`WHERE age > 25`)?→ 錯。只支援等值 (O(1)),不支援範圍/排序(hash 破壞順序)。
- **Q2:** Hash vs B-Tree 各自優勢場景?→ Hash:等值查詢極佳 (O(1)),精確匹配;B-Tree:支援等值 (O(log n))+範圍+排序,適用更廣。
- **Q3:** 碰撞兩種處理?→ Chaining(鏈結法)、Open Addressing(開放位址法)。
