# 03_基本觀念 / 10. Caching｜快取 — 投影片逐字原文

> 來源:`gemini_digest_pdf("03_基本觀念/10. Caching｜快取")`,2026-06-02。**尚未入庫 KG**(明天討論後蒸餾)。

---

## 什麼是 Caching?

高讀取流量幾乎必出現快取。從 Postgres 讀用戶資料可能 50ms,從 Redis 這類記憶體快取只需 1ms,快 50 倍——資料庫資料在磁碟,每次查詢付磁碟存取代價;記憶體離 CPU 近,繞過瓶頸。快取**降低 DB 負載、削減延遲**,但帶來快取失效與故障處理的新挑戰。

## 在哪裡快取(快取的多個層次)

快取存在系統多層:瀏覽器、CDN、應用程式、資料庫本身都有快取。

- **外部快取 (External Caching)**:獨立快取服務,application 透過網路溝通(Redis / Memcached)。所有 application server 可共用同一快取,擴展性好;支援 LRU 淘汰 + TTL 控制記憶體。**面試預設答案就是用 Redis 做外部快取**(效能、共享性、操作成熟度平衡最好)。先從這出發,再按需加 CDN 或客戶端快取。
- **CDN (Content Delivery Network)**:地理分散的伺服器網路,把內容快取在靠近用戶處(edge server)。運作:① 用戶請求圖片 ② 路由到最近 edge ③ 有快取直接回 ④ 沒有則從 origin 取、存起來、回傳 ⑤ 同區未來用戶幾乎瞬間拿到。無 CDN:伺服器在維吉尼亞、用戶在印度,延遲增加 250-300ms;有 CDN 從附近 edge 傳只需 20-40ms。**面試引入 CDN 最穩妥理由 = 大規模傳遞靜態媒體(.png/.jpg/.svg)**。
- **客戶端快取 (Client-Side Caching)**:把資料存靠近請求方(瀏覽器 HTTP cache/localStorage、行動 App 本地儲存)。也可指 client library 內部快取(如 Redis client 快取 cluster metadata,直接路由到正確節點)。後端能控制程度有限、資料易過時。例:Strava 離線存跑步資料稍後同步;瀏覽器重用已下載圖片。
- **行程內快取 (In-Process Caching)**:伺服器有大量記憶體,可直接在 application 行程內快取,比 Redis 更快(零網路呼叫)。適合頻繁請求的小塊資料:設定值、feature flag、小型參考資料集、熱門 key、rate limiting 計數器、預先計算結果。**限制**:每個 instance 有自己的快取、不跨伺服器共享,一個 instance 更新/失效其他不知道。用在很少變、頻繁存取的小資料;**不是 Redis 的替代品**,面試中只在介紹完外部快取後才提作為優化層。

## 快取架構模式(四種核心)

- **Cache-Aside(旁載快取)** —— **最常見、面試預設**:① app 先查快取 ② 有就回傳 ③ 沒有則查 DB、存入快取、再回傳。只在需要時快取(保持精簡),缺點是 cache miss 多一次延遲。**只記一種就記 cache-aside**。
- **Write-Through(同步寫穿)**:app 只寫快取,快取**同步**寫 DB,兩者都完成才算成功。需支援 write-through 的快取實作(Redis 原生不支援,需 app/框架實作)。取捨:寫入較慢、可能用永不再讀的資料污染快取、仍有 **dual-write 問題**(快取成功 DB 失敗或反之 → 不一致)。適用:讀取必須永遠回新資料、可接受寫入慢一點。
- **Write-Behind(非同步回寫)**:app 只寫快取,快取在背景**非同步批次**寫 DB。寫入很快,但風險是快取 flush 前崩潰 → 資料遺失。適用:高寫入吞吐、可接受最終一致性(分析、metrics pipeline)。
- **Read-Through(讀穿快取)**:快取充當智慧代理層,app 不直接碰 DB;cache miss 時由**快取本身**去 DB 取、存、回傳。是 write-through 的讀取端對應,常結合使用。把快取邏輯集中但增複雜度,需特殊 library,實務比 cache-aside 少見。**CDN 本質是一種 read-through 快取**。除非討論 CDN 等基礎設施,面試中很少主動提。

## 快取淘汰策略 (Eviction Policy)

快取記憶體有限,滿了需決定移除哪些:
- **LRU(最近最少使用)**:移除最久沒被存取的;用 linked list / ring buffer 追蹤,常數時間移除最舊。「最近用過很可能再用」適應大多數工作負載,**許多系統的預設**。
- **LFU(最不常使用)**:移除存取次數最少的;每 key 維護計數器(部分用近似 LFU 省成本)。適合某些 key 長期持續熱門(熱門影片、排行榜)。
- **FIFO(先進先出)**:只看插入時間移除最早的;簡單 queue 實作,但忽略使用模式、可能移除熱用項目,真實系統很少用。
- **TTL(存活時間)**:本身**不是淘汰策略**,而是為每 key 設過期時間,常和 LRU/LFU 搭配,在新鮮度與記憶體間平衡。只要資料須最終刷新(API 回應、session token)就必備。

## 常見快取問題

- **快取雪崩 (Cache Stampede / Thundering Herd)**:熱門快取項目過期瞬間,大量請求同時 miss、直接打 DB,一個查詢瞬間變幾百幾千個,可能打垮 DB。例:首頁 feed TTL 60s,12:01:00 到期那瞬間全部 miss。應對:**請求合併 (Request coalescing / Single flight)**——只讓一個請求重建快取、其他等結果(**最有效**);**快取預熱 (Cache warming)**——熱門 key 過期前主動刷新(只在 TTL 過期場景有效,寫入失效則無效)。
- **快取一致性 (Cache Consistency)**:快取和 DB 對同一資料回不同值。常見因為「從快取讀、先寫 DB」製造了快取還拿舊資料的窗口。例:用戶更新大頭照,新值寫 DB 但舊值還在快取,別人刷新前看到舊照。**沒完美解**,按新鮮度需求選:**寫入時失效快取**(更新 DB 後刪快取項目,下次讀以新資料填充)、**短 TTL 容許過時**、**接受最終一致性**(feed、metrics、分析短暫延遲通常 OK)。
- **熱 Key (Hot Keys)**:單一 key 流量遠多於其他。就算整體 hit rate 高,單一熱 key 也可能讓某 Redis shard 過載。例:Twitter 上 `user:taylorswift` 每秒幾百萬請求打掛單台 Redis。應對:**複製熱 key**(存多個節點分散讀取)、**加行程內備援快取**(避免一直打 Redis)、**套用 Rate Limiting**。
- **熱點快取 (Hotspot Cache)**:把熱 key 複製到多節點分散負載;**注意不要讓各副本過期時間完全相同**(以免同時過期造成 stampede)。

## 面試中怎麼談快取

**不要一上來就說快取**,先確立為什麼需要。發現以下問題時提出:
- **讀取密集**:「1000 萬 DAU,每人每天 20 請求 = 每天 2 億次讀取打 DB,每查 20-50ms。快取降到 <2ms 並把負載移開 DB。」
- **昂貴查詢**:「個人化 feed 要 join 多張表跑 200ms,可快取算好的 feed 60s,從 Redis 1ms 回。」
- **DB CPU 過高**:「尖峰 CPU 80%、幾乎都服務讀取且同查詢一再跑,快取熱門查詢可降 70-80% 負載。」
- **延遲需求**:「需 API <10ms,DB 查詢 30-50ms,必須快取。」

模式:**確認效能問題 → 用大概數字量化 → 說明快取如何解決**。

**介紹快取策略的五步**:① **確認瓶頸**(什麼慢、為何慢)② **決定快取什麼**(讀取頻繁、不常變、取得成本高的;想清楚 key 設計如 `user:123:profile`、`trending:posts:global`)③ **選快取架構**(符合一致性需求的模式,預設 cache-aside;靜態內容提 CDN;極端熱 key 提行程內)④ **設淘汰策略**(LRU 穩妥預設 + TTL 防過時)⑤ **說明缺點**(快取失效?快取故障 Redis 掛了降級查 DB + circuit breaker?快取雪崩用機率性提前過期或請求合併?)。

**核心取捨**:快取讓讀取更快、降後端負載,但引入資料過時與失效複雜度。不要快取所有東西——展示你知道何時值得那份複雜度,何時一個設計良好的 index 就夠了。

## 自我測驗 (Self-test)

**Q1:** 將 External Cache (Redis)、CDN、In-Process Cache 依延遲由低到高排序。
> **In-Process < External (Redis) < CDN**。In-Process:零網路、直接讀行程記憶體;Redis:同區 <1ms 但有網路呼叫;CDN:20-40ms(仍遠快於直連 origin 250-300ms)。

**Q2:** Cache-Aside 和 Write-Through 核心差異?
> Cache-Aside:app 自己管快取,讀 miss 時自己查 DB 並寫快取,快取與 DB 分開管理。Write-Through:app 寫快取、快取同步寫 DB,兩者都完成才算成功,快取永遠最新但寫入較慢。

**Q3:** 什麼是 Cache Stampede?最有效解法?
> 熱門快取項目過期時大量請求同時 miss 打 DB 可能打垮。最有效:**Request Coalescing(請求合併/Single Flight)**——只讓一個請求重建、其他等結果。

**Q4:**(何者錯誤)(A)LRU 移除最久沒存取 (B)LFU 移除存取次數最少 (C)FIFO 是生產最常用 (D)TTL 常搭配 LRU/LFU
> **(C)**。FIFO 忽略使用模式很少在生產用,LRU 才是最常用預設。

**Q5:** 極端熱門 key(名人個資)可能造成什麼問題?如何解?
> **Hot Key**:單一 key 大量流量讓某 Redis 節點過載成瓶頸。解:複製熱 key 到多節點、加行程內備援快取、套用 rate limiting。

**Q6:** 面試介紹快取策略的五步?
> 確認瓶頸 → 決定快取什麼 → 選快取架構(預設 cache-aside)→ 設淘汰策略(LRU+TTL)→ 說明缺點(失效/故障/雪崩)。
