# 03_基本觀念 / 07. Consistent Hashing｜一致性雜湊 — 投影片逐字原文

> 來源:`gemini_digest_pdf("03_基本觀念/07. Consistent Hashing｜一致性雜湊")`,2026-06-02。**尚未入庫 KG**(明天上課討論後再蒸餾)。

---

## Slide 1:為什麼需要一致性雜湊?

分散式系統中,資料常需分散到多台伺服器。最直觀的方法是簡單 hash:
```
server = hash(key) % N   # N = 節點數
```
**問題**:
- 若 N 改變(新增/移除伺服器),**幾乎所有資料都要重新分配**。
- 造成大量搬遷 → 效能問題與系統不穩定。

運算規則:`value % N = Node Number`。例:`3 % 3 = 0 → Node 0`。

**N = 3,值 1~9**:
- 3, 6, 9 → Node 0
- 1, 4, 7 → Node 1
- 2, 5, 8 → Node 2

**多加一個 node → N = 4**:
- 4, 8 → Node 0
- 1, 5, 9 → Node 1
- 2, 6 → Node 2
- 3, 7 → Node 3

→ **9 個值裡有 7 個被移動**(只有 1 和 2 留在原位)。這就是簡單 hash 的致命傷。

## Slide 2:Consistent Hashing 怎麼運作 —— Hash 環

核心價值:**伺服器數量改變時,只需搬動一小部分資料,大部分留在原節點。**

**(a) Hash 環 (Hash Ring)**:
- 把所有可能的 hash 值映射到一個圓環(0 → MAX_HASH)。
- 每個節點 (server) 用 hash 映射到環上一個位置。
- 每筆資料也用 `hash(key)` 計算位置。
- **規則:資料順時針找到第一個節點,交給它存。**

範例環(0~1024,node D 在 0/1024、A 在 250、B 在 500、C 在 750):
- hashed value = 200 → 落在 D~A 區間 → 順時針第一個是 **A**
- hashed value = 400 → 落在 A~B 區間 → **B**
- hashed value = 600 → 落在 B~C 區間 → **C**

## Slide 3:新節點加入 / 節點移除

**(b) 新節點加入**:
- 假設新節點 E 落在 150。
- **只有「D 與 E 之間」(0~150) 的資料,從 A 搬到 E**;其他資料不變。
- (只有落在「新節點與其前一個節點之間」的 key 需搬移)

## Slide 4:節點移除 + 為什麼需要虛擬節點

**(c) 節點移除**:
- 假設節點 A 下線。
- 原本屬於 A 的資料移到**下一個節點 B**;其他不受影響。

**虛擬節點 (Virtual Nodes) —— 為什麼需要?**
基本版 consistent hashing 中,每個實體節點在環上只有一個位置,問題:
1. **資料分佈不均**:hash 雖隨機,但節點少時分配可能很不平均。例:移除 A 後,B 要負擔全部一半的 hashed values。
2. **節點性能不同**:強/弱伺服器都只拿一個區間 → 沒按硬體能力分配。

→ 解法:引入虛擬節點。

## Slide 5–6:虛擬節點做法與好處

**做法**:
- 每個實體節點對應**多個虛擬節點 (vNodes)**。
- vNodes 也映射到環上,像多個小節點。
- 資料依舊順時針找最近的 vNode 存放,再交由對應的實體節點。

例:原本 A(250) B(500) C(750) D(1024);用 vNodes 後 —— A 有 A1(250)/A2(580)/A3(920),B 有 B1(500)/B2(830)/B3(80),C 有 C1(750)/C2(160)/C3(420),D 有 D1(1024)/D2(330)/D3(660)。每個實體節點散佈多個位置 → key 分布更均勻。

**好處**:
1. **平均分佈**:節點數少也能均勻切開。
2. **依硬體能力分配**:強機器放更多 vNodes、負責更多資料;弱機器少放。
3. **彈性擴展**:節點加入/移除時,因每節點有多個 vNodes,搬遷更細顆粒、負載更平滑。

**面試應用 —— 什麼時候想到 Consistent Hashing?**
口訣:**「變動的節點 + 需要穩定歸屬 + 想少搬家」→ consistent hashing**。場景:
- 分散式快取 sharding:Memcached / Redis(避免 `key % N` 大規模重分配)
- 儲存 sharding/routing:Cassandra / Dynamo 風格 partition routing
- Sticky connections:聊天室/即時服務把同一使用者/聊天室路由到固定節點
- API Gateway / Sticky Sessions:減少跨節點 session 同步
- Rate Limiting:把 key (user_id, api_key) 穩定分到對應節點做限流
- Metrics / Aggregation:相同維度聚到固定節點累加,降低跨節點匯總
- CDN / Edge Routing:URL 或內容 ID 穩定映射到 edge 節點做快取

## 自我測驗 (Self-test)

**Q1:** 簡單 hash (`hash(key) % N`) 在什麼情況下出問題?舉例。
> N 改變時(增/減伺服器),幾乎所有 key 的 `hash(key) % N` 都變,大量資料要重分配。例:N=3→4,9 個 key 有 7 個位置改變。

**Q2:**(是非)consistent hashing 中新增節點時,所有 key 都要重新分配。
> 錯。只有落在「新節點與其前一個節點之間」的 key 要搬到新節點,其他不受影響。這正是核心價值。

**Q3:** Virtual Node 解決了哪兩個問題?
> ① 資料分佈不均(節點少時位置集中、某些節點過載)② 硬體差異(強機器分配更多 vNode 承擔更多負載)。

**Q4:** 列舉三個適合用 consistent hashing 的場景。
> 任三:分散式快取 sharding、儲存路由 (Cassandra/Dynamo)、sticky connections、API Gateway/Sticky Sessions、Rate Limiting、Metrics/Aggregation、CDN/Edge Routing。
