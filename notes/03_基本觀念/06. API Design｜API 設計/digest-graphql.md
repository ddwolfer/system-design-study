# 06 API — GraphQL(來源:`Caching Headers.pdf` / `Query and Batch API.pdf`,檔名貼錯;簡體)

> 2026-06-03。已蒸餾進 KG(API Paradigms 節點)。

## 基本概念
GraphQL(Facebook 提出)是 API **查詢語言 + 執行環境**。核心:**客戶端決定需要什麼資料,伺服器只回應請求的欄位**。對比 REST 資源導向,GraphQL 查詢導向:一次查詢可跨多個資源組合出所需資料。

## 核心特徵
- **單一端點 (Single Endpoint)**:REST 多個 URI;GraphQL 通常只用一個 `/graphql`。
- **精準查詢 (Client-driven)**:客戶端決定要哪些欄位 → 避免 over-/under-fetching。
- **強型別 Schema**:定義所有型別/欄位/關聯,自帶文件與型別檢查。
- **單一請求跨多資源 (Nested Queries)**:一次查詢多個相關資源(查使用者同時抓訂單),REST 可能要多次 call。

## 三種操作
- **Query** → 讀取資料(可巢狀:`user(id:123){ name email orders{ id total } }`)。
- **Mutation** → 修改資料(新增/更新/刪除:`createUser(name,email){ id name }`)。
- **Subscription** → 即時資料推送(類似 WebSocket/SSE:`newOrder{ id total }`)。

回傳格式:JSON,結構與查詢相同(`data` 欄位下鏡像查詢結構)。

## 優缺點
- 優點:客戶端控制所需資料(避免 over/under-fetching)、強型別 schema、一次請求跨多資源減少 call。
- 缺點:
  - 伺服器端實作複雜(需 resolver + schema 管理)。
  - **N+1 問題**:resolver 是 field-based,巢狀查詢易變成「先查 users(1),再對每個 user 各查一次 orders(N)」。解法:**DataLoader / batch loading**(合併成一個 IN 查詢)。
  - **Cache 比 REST 難**:所有查詢走同一 endpoint,不同 query body 產生不同結果,無法用 URL 當 cache key → 需 persisted queries 或 Apollo/Relay 客戶端快取。
  - Query 過於靈活可能效能瓶頸/惡意查詢 → 需 query complexity 限制。

## 面試
適合前端需要靈活查詢或一次取多種資源(行動/弱網路只抓需要欄位、複雜頁面同時顯示使用者+訂單+通知)。資料結構複雜、前端需求常變 → GraphQL;簡單 CRUD 或高效快取 → REST 更直接。

### 自我測驗
- Q1: 三種操作?→ Query 讀取、Mutation 修改、Subscription 即時推送。
- Q2: N+1 問題?如何解?→ resolver field-based 致巢狀查詢變 1+N 次 DB 查詢;用 DataLoader/batch loading 合併成 IN 查詢。
- Q3: 為何 GraphQL 快取比 REST 難?→ REST 一資源對應固定 URL 可用瀏覽器/CDN 快取;GraphQL 全走同一 endpoint、不同 query body 不同結果,無法用 URL 當 key,需額外機制。
