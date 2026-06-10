# 03_基本觀念 / 11. Sharding｜分片 — 投影片逐字原文

> 來源:`gemini_digest_pdf("03_基本觀念/11. Sharding｜分片")`,2026-06-02。**尚未入庫 KG**(明天討論後蒸餾)。

---

## 什麼是 Sharding?

App 起飛、流量成長、資料變大。一開始靠**垂直擴展**(升級更大 instance、加 CPU/記憶體/儲存)撐一段時間,但終究撞到單台天花板:查詢變慢、寫入成瓶頸、儲存逼近上限。就算 Amazon Aurora 也有約 **256 TB** 硬性限制。單台跟不上時只剩一條路:**把資料拆散到多台機器** = Sharding。

**Partitioning vs Sharding**(常被混用,嚴格說):
- **Partitioning**:在**單一 database instance 內**拆分資料(不增加機器,讓 DB 更有效率管理)。
  - 例:5 億筆、2TB 的訂單表,查上月訂單要掃整張表、索引龐大、vacuum/analyze/重建索引會鎖表。拆成更小分區後查上月只掃對應分區。
  - **水平分區 (Horizontal partitioning)**:把 **row** 拆到不同分區(每年訂單一個分區,同欄位、更少 row)。
  - **垂直分區 (Vertical partitioning)**:把 **column** 拆到不同分區(常用欄位一區、體積大/少存取的另一區,同 row、更少欄位)。
- **Sharding**:把水平分區延伸到**多台機器**。每個 shard 持一部分資料,合起來才是完整資料集;每個 shard 是獨立 DB(自己的 CPU/記憶體/儲存/連線池)。例(用 id 分割訂單):Shard 1→ID 1~100萬、Shard 2→100萬~200萬、Shard 3→200萬~300萬。沒有任何一台持全部資料或承擔全部流量,可隨增加 shard 擴展儲存與讀寫吞吐。
- 實務上多數工程師不太區分,重點是說清楚資料在一台還是多台。

## 如何拆分資料:兩個相互配合的決策

① 用什麼欄位分片(shard key,決定資料怎麼分組)② 怎麼分配這些分組(分佈策略,決定怎麼分散到各機器)。

### 選擇 Shard Key —— 好的 shard key 三條件

- **高基數 (High cardinality)**:key 要有很多不同值。布林欄位最多兩個 shard 沒意義;user ID(幾百萬)有足夠空間分散。
- **均勻分佈 (Even distribution)**:值要能均勻分散。用國家分片但 90% 用戶在美國 → 那個 shard 過大。User ID 通常均勻。
- **契合查詢模式 (Aligns with queries)**:最常見查詢最好只打一台 shard。用 `user_id` 分片,「取用戶個資/訂單」只打一台;跨所有 shard 的查詢很昂貴。

好例子:🟢 用戶導向 App 用 `user_id`(高基數、均勻、多數查詢本就限定單一用戶);🟢 電商訂單表用 `order_id`(高基數、查詢多限定特定訂單、隨時間均勻)。
爛例子:🔴 `is_premium`(布林,只有 2 個 shard,免費用戶多就過載);🔴 成長中的表用 `created_at`(所有新寫入打最新 shard 成寫入熱點,舊 shard 幾乎只讀歷史)。

## Sharding 策略(三種)

- **範圍分片 (Range-Based)**:依連續值範圍分組(Shard 1→User 1~100萬…)。優點:簡單、支援高效範圍掃描(查 user 50萬~60萬只打一個 shard)。缺點:存取模式通常不均勻;用 `created_at` 幾乎所有流量打最新 shard。最適合「不同用戶自然查不同範圍」(多租戶 SaaS:每家公司一段 ID 範圍)。
- **雜湊分片 (Hash-Based)** —— **預設首選**:用雜湊函數均勻分散。`shard = hash(user_id) % 4`(User 42→Shard 2…)。優點:分佈均勻。缺點:增減 shard 時 `% 4` 變 `% 5` 幾乎每筆都換 shard、大量搬遷 → **這就是 Consistent Hashing 的用武之地**(把搬遷量降到最低)。面試中除非特別說明,通常預設雜湊分片。
- **目錄分片 (Directory-Based)**:用查找表決定每筆放哪(`User 15→Shard 1`…)。優點:靈活(熱門用戶可搬到專屬 shard、重平衡只需更新 mapping、可實作複雜邏輯)。缺點:每個請求多一次查找(增延遲)、目錄服務成關鍵依賴(掛了即使所有 shard 健康也全停)。**面試中很少是正解**(引入單點故障 + 增延遲)。

## Sharding 的挑戰

- **熱點與負載不均 (Hot spot)**:就算好的 shard key,某些 shard 仍可能收到遠多流量,過載的 shard 成瓶頸,抵消 sharding 好處。最常見是**名人問題 (celebrity problem)**:用 `user_id` 分片,Taylor Swift 那個 shard 流量可能是普通用戶 1000 倍(圖示:Shard 1 帶名人 = 1M qps,其他 = 1k qps)。時間戳記分片則所有新寫入打最新 shard。偵測:監控各 shard 的查詢延遲、CPU、請求量。
  - 應對:**把熱 key 隔離到專屬 shard**(名人搬到專用 shard,這也是目錄分片偶有用處之處,但不會一開始就這樣設計);**使用複合 shard key**(`hash(user_id + date)` 讓單一用戶資料隨時間分散);**動態 shard 拆分**(MongoDB balancer 自動拆分遷移 chunk;Vitess 支援線上 resharding 但需操作人員發起)。
- **跨 Shard 操作**:任何需從多 shard 取資料的查詢都昂貴(同時查多 shard、等全部回應、自己聚合)。查詢模式和 shard key 不對齊時出現:用 `user_id` 分片,「取 user 12345 個資」只打一台快;但「全站最熱門 10 篇貼文」必須查每一個 shard(64 個 shard = 64 倍網路呼叫+延遲,這叫 scatter-gather)。
  - 最小化方式:**快取結果**(全站熱門 10 篇快取 5 分鐘,後續從快取拿;適合可接受最終一致性的排行榜/熱門/統計);**反正規化讓相關資料放一起**(常一起查貼文+用戶資料就把部分貼文資訊存在用戶 shard 上,資料重複但能從一個 shard 拿齊);**接受罕見查詢的代價**(一天開幾次的管理後台跑慢點 OK)。面試中跨 shard 操作往往是「該重新思考設計」的訊號。
- **維護一致性**:單一 DB 內 transaction 直觀;sharding 打破這點(用戶帳號在 shard 1、交易記錄在 shard 2,不能用單一 DB transaction)。教科書解法 **2PC** 保證一致但慢又脆弱(任一 shard/協調者中途失敗整個卡死),大多數生產系統避免。
  - 怎麼辦:**設計成避免跨 shard transaction**(最好的解法——用 `user_id` 分片就把一個用戶所有資料放他的 shard,所有 transaction 都單 shard);**對多 shard 操作用 Saga 模式**(拆成一連串步驟+補償動作,代價是最終一致性而非 2PC 的脆弱;例:跨 shard 轉帳 ① A 扣款 shard 1 ② B 存入 shard 2 ③ 失敗則退款 A);**接受最終一致性**(粉絲數反正規化存多 shard,幾秒不一致沒問題、最終收斂)。一句話:大多數應用可設計成完全避免跨 shard transaction;若一直需要分散式 transaction,很可能是 shard key 選錯或邊界劃分有問題。

## 現代資料庫中的 Sharding

好消息:你大概不會從頭實作。現代分散式 DB 多自動處理:
- **Cassandra**:partitioner(如 Murmur3Partitioner)+ 虛擬節點,是 consistent hashing 的一種實作。
- **DynamoDB**:對 partition key 雜湊路由到內部分區,隨資料成長自動拆分/合併分區(非用戶可見的經典環形 consistent hashing)。
- **MongoDB**:依 shard key 分成範圍 chunk(選雜湊 shard key 則在雜湊空間劃分);background balancer 自動拆分遷移 chunk。
- **SQL**:Vitess、Citus 是開源 sharding 層(架在 PostgreSQL/MySQL 前,處理路由、跨 shard、resharding);AWS Aurora、Google Cloud Spanner 提供內建 sharding 的分散式 SQL。
- 面試中說「用 DynamoDB 以 `user_id` 為 partition key」或「用 Vitess 以 `user_id` 分片、擴展時由操作人員發起線上 resharding」就夠。

## 面試中怎麼談 Sharding

**小心過早 sharding**——先確立為什麼單一 DB 不夠,再討論。碰到以下限制可帶出:
- **儲存空間**:「5 億用戶 × 5KB = 2.5TB,單台 Postgres 還行,但成長 10 倍就需分片。」
- **寫入吞吐**:「尖峰每秒 5 萬次寫入,單台撐不住,應分片。」
- **讀取吞吐**:「就算有 read replica,服務 1 億 DAU 仍需把讀取分散到多 shard。」

公式:**確認瓶頸 → 解釋為什麼單台無法擴展 → 提出 sharding**。第一大錯誤是還沒證明必要性就引入 sharding。

**逐步說明範例(社群媒體 App)**:① **提出 shard key**(多數查詢以用戶為中心 → 用 `user_id`)② **選分佈策略**(雜湊分片 + consistent hashing)③ **點出取捨**(全域查詢昂貴 → 快取熱門+背景預計算)④ **應對成長**(從 64 個 shard 起步,consistent hashing 讓加 shard 只搬一小部分資料)。

**總結**:Sharding 是單台 DB 無法應付規模時把資料拆散到多台。兩個核心決策:選契合查詢模式的 shard key、選能均勻分散的分佈策略。選錯 → 熱點 + 昂貴跨 shard 查詢。確認瓶頸後再提,別過早分片——一台調校良好的單台 DB 能走的路比多數人預期遠。

## 自我測驗 (Self-test)

**Q1:** Partitioning vs Sharding?
> Partitioning:同一台 DB 內邏輯分割;Sharding:資料分散到不同機器。重點是說清楚在一台還是多台。

**Q2:** shard key 三條件?為什麼 `is_premium` 是爛 key?
> 高基數、均勻分佈、對齊查詢模式。`is_premium` 是布林、基數只有 2、最多 2 個 shard 無法有效分片。

**Q3:** Range-Based vs Hash-Based 優缺點?面試預設?
> Range:支援範圍查詢但易熱點(`created_at` 新寫入全打同一 shard);Hash:分佈均勻但不支援範圍查詢、增減節點需大量搬移(除非 consistent hashing)。預設 **Hash-Based**。

**Q4:** 什麼是 Celebrity Problem (Hot Spot)?兩個解法?
> 某些 shard 因存熱門資料(名人)承受不成比例流量成瓶頸。解:隔離熱 key 到專屬 shard、使用複合 shard key(`hash(user_id+date)`)。

**Q5:** 頻繁需要跨 shard 查詢代表什麼?如何應對?
> 設計訊號:shard key 選錯或邊界劃分有問題。應對:重新評估 shard key、反正規化把相關資料放同一 shard、對罕見查詢用快取或背景預計算。
