# 03_基本觀念 / 12. Replication｜複寫 — 投影片逐字原文

> 來源:`gemini_digest_pdf("03_基本觀念/12. Replication｜複寫")`,2026-06-02。**尚未入庫 KG**(明天討論後蒸餾)。本課最長(17 張),內容對應《DDIA》第 5 章。

---

## 什麼是 Replication?

把同一份資料的副本放在多台透過網路連接的機器上。三個原因:
- **降低延遲**:資料放在地理上靠近用戶處,讀取更快。
- **提升可用性**:某些節點失效時系統仍能運作。
- **擴展讀取吞吐量**:把讀取請求分散到多台。

難點在於**資料會改變**——若資料永不變,複製一次就完工;真正挑戰是資料持續更新時如何讓所有副本保持一致。三種主要架構:**Single-leader、Multi-leader、Leaderless**。

## 架構一:Single-Leader Replication

最常見、最直覺。每個存副本的節點叫 **replica**。運作:
1. 一台 replica 指定為 **leader(主節點 / master / primary)**,所有寫入必須先打到 leader。
2. 其他叫 **follower(從節點 / slave / secondary / read replica)**。Leader 把每次寫入包成 **replication log** 發給所有 follower,follower 照順序套用。
3. 讀取可打 leader 或任何 follower,但**寫入只能打 leader**。

被 PostgreSQL(9.0+)、MySQL、MongoDB、Kafka、RabbitMQ 廣泛採用。

### 同步 vs 非同步 Replication(最關鍵取捨之一)

- **同步 (Synchronous)**:Leader 等 follower 回報成功才通知 client 完成。好處:follower 保證有最新資料;壞處:那台 follower 沒回應,整個系統寫入全卡住。
- **非同步 (Asynchronous)**:Leader 發出去不等確認就通知 client 成功。寫入更快,但 leader 在 follower 同步前掛掉,那些寫入永久消失。
- **半同步 (Semi-synchronous)**:保持一台 follower 同步、其他非同步,至少兩份節點(leader + 一台 follower)有最新資料。
- 🎤 **面試必考**:同步保一致性犧牲可用性;非同步保可用性但有資料遺失風險。沒有哪個更好,看系統對一致性/可用性的要求。

### 新增 Follower(不停機)

不能只複製檔案(DB 一直在寫,複製過程不一致)。標準流程:① 對 leader 拍一致的**快照 (snapshot)**(通常不鎖整個 DB)② 複製快照到新 follower ③ follower 連 leader 請求自快照以來的所有變更(快照對應 replication log 精確位置:PostgreSQL 叫 log sequence number、MySQL 叫 binlog coordinates)④ 追上後即時處理 leader 的變更。

### 處理節點失效

- **Follower 失效:追趕恢復**。每台 follower 本地保留 replication log,崩潰重啟後知道上次處理到哪,向 leader 請求中斷期間的變更,追上後繼續。
- **Leader 失效:Failover**(較棘手)。把某台 follower 提升為新 leader。自動 failover 步驟:① **偵測失效**(沒有萬無一失的方法,多用 timeout,如 30 秒沒回應就認定掛了)② **選新 leader**(多數 replica 投票選舉,或由 controller 指定;最好選資料最新的)③ **重新設定系統**(client 寫入打去新 leader;舊 leader 回來要變成 follower)。
  - **Failover 地雷**:① 非同步下新 leader 可能還沒收到舊 leader 最後的寫入 → 通常被丟棄、違反持久性期待 ② **Split brain**(兩節點都以為自己是 leader、都接受寫入 → 資料損毀;有些用 **STONITH**(Shoot The Other Node In The Head)關掉一個,設計不好可能兩個都被關)③ timeout 長短難拿捏(太長恢復慢、太短忙時誤觸發)。

### Replication Log 的實作方式

- **Statement-based(語句複製)**:Leader 把每個 SQL 發給 follower 執行。問題:非確定性函數(`NOW()`、`RAND()`)、autoincrement、有副作用的語句會不一致。MySQL 5.1 前用。
- **WAL shipping(預寫日誌傳送)**:把 storage engine 做崩潰恢復的 WAL 直接傳給 follower。缺點:WAL 在底層位元組層次,replication 與 storage engine 緊耦合,升版本可能需停機。PostgreSQL、Oracle 用。
- **Logical (row-based) log(邏輯日誌複製)**:用和 storage engine 分離的格式記錄 row 層次變更。leader/follower 可跑不同版本,易被外部解析,是 **Change Data Capture (CDC)** 的基礎。MySQL binlog 用。
- **Trigger-based(觸發器複製)**:把 replication 邏輯搬到應用層(trigger + stored procedure)。overhead 高,但需只複製部分資料、跨不同類型 DB 複製時有用。

### Replication Lag 的三種問題

非同步下從 follower 讀可能看到過時資料,這種暫時不一致叫 **eventual consistency(最終一致性)**。lag 通常幾毫秒,但接近滿載或網路有問題時可能拉長到幾秒甚至幾分鐘。

- **問題一:Read-After-Write(讀自己剛寫的)不一致**。用戶發貼文馬上看卻看不到(讀取打到還沒同步的 follower)。解法:① 讀自己可能改過的東西時從 leader 讀(自己的 profile 只有自己能改 → 自己看自己永遠從 leader,看別人從 follower)② 追蹤最後更新時間,夠近就從 leader 讀 ③ client 記住最後寫入 timestamp,確保讀取的 replica 至少和該 timestamp 一樣新。(多裝置情境更複雜,timestamp 需中央同步。)
- **問題二:Monotonic Reads(單調讀取)不一致**。先從 lag 小的 follower 讀到朋友留言,刷新被路由到 lag 大的 follower,留言消失了(時光倒流)。解法:**確保同一用戶的讀取永遠打同一台 replica**(用 user ID 的 hash 決定,那台掛了再換)。
- **問題三:Consistent Prefix Reads(因果一致性)不一致**。A 問問題、B 回答,第三個觀察者因不同 partition 的 lag 不同,先看到答案才看到問題,因果順序顛倒。解法:**有因果關係的寫入都寫到同一 partition**;或用記錄因果依賴的演算法(**version vector**)。
- **核心洞察**:與其讓應用自己處理這些微妙問題,更好的做法是用 transaction 讓 DB 提供更強保證;但分散式 transaction 有效能與可用性代價,沒有放諸四海皆準的答案。

## 架構二:Multi-Leader Replication

Single-leader 弱點:所有寫入都得通過那一台 leader,連不到就寫不進去。Multi-leader 讓多個節點都能接受寫入,每個 leader 同時是其他 leader 的 follower。

- 🎤 面試提示:**單一 datacenter 內用 multi-leader 幾乎從不值得**(複雜度遠超好處);**跨多 datacenter 就完全不同**。
- **適用場景**:① **多 datacenter 部署**(每個 DC 有自己的 leader,DC 內部用一般 leader-follower,DC 之間各 leader 互相非同步複製)② **離線操作**(手機日曆 App 離線新增、連網再同步,每台裝置是一個 leader)③ **協作編輯**(Google Docs,每個用戶本地副本是 leader,即時套用本地、非同步複製,帶來和 multi-leader 一樣的衝突問題)。

| 面向 | Single-Leader | Multi-Leader |
|---|---|---|
| 效能 | 每次寫入跨網路到 leader 的 DC,增延遲 | 寫入在本地 DC 處理,跨 DC 延遲對用戶不可見 |
| DC 失效容忍 | Leader 所在 DC 掛了要 failover | 每個 DC 可獨立運作,恢復後再同步 |
| 網路問題容忍 | 對 DC 間連線問題敏感 | 非同步可容忍暫時網路中斷 |

### 處理寫入衝突(multi-leader 最大挑戰)

A、B 同時在不同 leader 改同一份資料都成功 → 非同步複製時衝突。
- **衝突迴避 (Conflict Avoidance)**:確保特定記錄所有寫入走同一 leader(同一用戶請求永遠路由到同一 DC)。但那個 DC 掛了就失效。
- **收斂到一致狀態 (Converging)**:① **Last Write Wins (LWW)**:每寫入附 timestamp,挑最大的當勝者、丟棄其他。簡單但丟資料,是 Cassandra 唯一支援的解法、Riak 也有 ② **Replica 優先序**:編號高的優先(同樣丟資料)③ **合併值**(按字母排序串接)④ **記錄衝突稍後解決**(保留所有版本,讓應用/用戶決定)。
- **自訂衝突解法**:on write(偵測到立刻呼叫處理函數)或 on read(保留所有版本、下次讀取全回傳給應用決定)。
- **自動衝突解法研究方向**:**CRDT (Conflict-free Replicated Datatypes)**(集合/計數器/有序列表,自動合理解衝突)、**Mergeable Persistent Data Structures**(類 Git,三方合併)、**Operational Transformation**(Etherpad、Google Docs 用,專為並發編輯有序字元設計)。

### Replication Topology(寫入在 leader 間如何傳播)

- **Circular**:每節點只從一個接收再轉發給下一個(MySQL 預設)。
- **Star**:一個根節點轉發給所有其他。
- **All-to-all**:每個 leader 發給所有其他 leader。
- Circular/Star 問題:任一節點掛掉中斷 replication 流動。All-to-all 容錯好,但可能出現寫入到達不同 replica 順序不一致(causality 違反)→ 需 version vector。

## 架構三:Leaderless Replication

完全拋棄 leader,任何 replica 都能直接接受寫入。Amazon Dynamo 讓它重新流行,Riak、Cassandra、Voldemort 受啟發。

- **節點掛掉時怎麼寫入**:client 把寫入同時發給所有 replica,夠多台成功回應就算成功(如 3 台中 2 台),忽略沒回應的。問題:那台恢復後資料過時,client 從它讀會拿到舊值。解法:**讀取時也同時向多台發請求,用 version number 判斷哪個更新,以最新為準**。
- **讓 Stale Replica 追上**:① **Read repair(讀取修復)**:client 並行讀多台,發現某台 version 舊就把新值寫回(對頻繁讀的值效果好)② **Anti-entropy process(反熵程序)**:背景持續掃描差異補資料(不保證順序、可能明顯延遲)。

### Quorum Reads 和 Writes

n 個 replica,寫入需 w 個確認、讀取需查 r 個。只要 **w + r > n**,讀取時至少有一個節點有最新資料(w 個寫入節點和 r 個讀取節點必重疊)。
- 常見配置:n=3, w=2, r=2(容忍一個失效);n=5, w=3, r=3(容忍兩個失效)。
- 取捨:少讀多寫可設 w=n, r=1(讀超快但一個節點掛寫入就失敗);多讀少寫相反。
- **Quorum 一致性的限制**(即使 w+r>n 仍可能讀到舊值):sloppy quorum、並發寫入(LWW 因 clock skew 丟寫入)、並發讀寫、節點失效+舊資料恢復。Dynamo-style 主要針對能接受最終一致性的場景;quorum 不提供 read-after-write / monotonic reads / consistent prefix reads,需更強一致性要用 transaction 或 consensus。

### Sloppy Quorum 和 Hinted Handoff

網路中斷讓 client 連不到該存資料的 n 台「家」節點但連得到其他節點。兩個選擇:① 返回錯誤 ② **Sloppy quorum**:接受寫入暫存到現在連得到的節點(不在原 n 台裡)。提升寫入可用性(任何 w 台可達就能寫),但代價是即使 w+r>n 也無法保證讀到最新(最新值可能在臨時保管節點)。網路恢復後臨時寫入送回家節點 = **hinted handoff(帶提示的交接)**(像沒帶鑰匙借住鄰居家,拿到鑰匙再回家)。

### 偵測並發寫入

- **Last Write Wins (LWW)**:附 timestamp 取最大。最終一致但犧牲持久性(並發寫入只一個存活、其他靜默丟棄),只在可接受資料遺失(caching)時用。
- **Happens-before 關係**:B 知道 A、依賴 A、或建立在 A 之上 → A 先於 B;兩者互不知情 = **並發 (concurrent)**(不是「同一時間點」而是「互不知情」)。
- **Version Vector 追蹤因果依賴**:每個 replica 為每個 key 維護自己的 version number,每次寫入遞增、連同值儲存。client 讀取時 server 回傳所有未被覆蓋的值 + 最新 version number;client 寫入時須帶上前次讀取的 version number 並合併前次收到的所有值;server 可覆蓋等於或低於該版本的值,保留更高版本的(那些是並發的)。多 replica 時需每個 replica 各自 version number 組成的集合 = **version vector**(有時叫 vector clock,嚴格說有所不同)。

## 面試裡什麼時候用這些

- **讀取擴展**:讀取量太高 → read replica(follower)。「single-leader,寫入走 primary,讀取分散到多台 replica」,接著主動提 replication lag 與解法。
- **高可用**:服務不能停 → 「primary 掛掉用 automatic failover 提升一台 replica」,講 failover 挑戰(split brain、資料遺失)。
- **地理分散**:多 DC → multi-leader。「每個 DC 放一台 leader,用戶寫到最近 DC 減延遲」,討論衝突解法。
- **高可用+寫入擴展**:leaderless 的 quorum。「Cassandra quorum 寫入(w=2,n=3)容忍一個節點失效」,注意 LWW 資料遺失。

### 常見面試情境

- **社群媒體(Twitter/IG)**:讀遠多於寫,read replica 第一工具。Replication lag 可接受,但自己剛發的貼文要馬上看到 → read-after-write,自己的內容從 primary 讀。
- **金融系統(支付/庫存)**:強一致性 > 可用性。Single-leader + 同步(或 semi-sync)確保每筆交易至少兩節點持久化。Failover 謹慎,寧願停機也不要不一致。
- **全球 CDN / 快取**:multi-leader 最有說服力。每地區自己的 leader,衝突用 LWW(cache 資料遺失通常可接受),TTL 讓過時資料失效。
- **IoT / 指標收集**:大量時序寫入,leaderless(Cassandra)適合,LWW 資料遺失在感測器場景通常可接受。
- **協作應用(Google Docs)**:multi-leader,衝突需 Operational Transformation 或 CRDT,不是簡單 LWW。

### 常見 Deep Dive

- **「怎麼保證用戶寫入後馬上讀得到?」**(read-after-write):① 自己的資料從 primary 讀 ② 追蹤用戶最後寫入 timestamp(存 session/cookie),只接受夠新的 replica,否則從 primary ③ 短暫讀 primary 視窗(剛寫入後如 5 秒所有讀取打 primary)。先承認問題、再提解法、討論取捨。
- **「Failover 時怎麼避免資料遺失?」**:同步/semi-sync(等至少一台 follower 確認)、追蹤最新 follower(選資料最新的當新 primary)、Raft/Paxos consensus(真正強一致,PostgreSQL Patroni、etcd 實作,代價是複雜度+延遲)。
- **「Multi-leader 寫入衝突怎麼設計解法?」**:先問清楚一致性需求(金融絕不能丟、社群 profile 也許可 LWW);LWW(簡單但丟資料)、衝突迴避(路由到同一 leader)、保留所有版本+應用層解決(購物車取聯集)、CRDT(計數器/集合/有序列表自動合併,Riak 2.0 原生、Cassandra 有計數器類型)。
- **「Quorum 讀寫在網路分區時怎麼辦?」**(CAP 核心):**CP 配置**(嚴格 quorum,連不到足夠節點就返回錯誤,不丟資料但分區期間可能無法服務,適合金融);**AP 配置**(sloppy quorum,接受寫入先存可達節點、恢復後 hinted handoff,一直可寫但可能讀到舊資料,適合用戶行為記錄/IoT)。面試官要你說出取捨,不是假裝有完美解法。

### 怎麼提 Replication

主動帶進議題(別等問):讀取密集主動提 read replica + lag 討論;高可用主動提「單點 primary 是 SPOF,設計 automatic failover」。**永遠討論取捨**;**用數字支撐決策**(「每秒 50,000 次讀取超過單台上限 → 加三台 read replica,lag 通常 <100ms 可接受」)。並用 fencing token 確保舊 primary 完全下線後新 primary 才接管(防 split brain)。

## 總結

| 架構 | 優點 | 缺點 | 適用場景 |
|---|---|---|---|
| Single-Leader | 簡單、無衝突 | Leader 是單點、所有寫入走同一節點 | 大多數讀取密集應用 |
| Multi-Leader | 跨 DC 寫入延遲低、容忍 DC 失效 | 寫入衝突需解決、實作複雜 | 多 DC 部署、離線操作 |
| Leaderless | 高可用寫入、容忍節點失效 | 一致性模型弱、需應用層合併 | 高可用、能接受最終一致性 |

Replication Lag 的三個一致性保證(read-after-write、monotonic reads、consistent prefix reads)**不是靠架構自動獲得,需在設計中明確處理**。沒有「最好的」架構——面試官在乎你能否根據需求做有根據的選擇、解釋取捨、知道哪些保證你的設計沒提供。

## 自我測驗 (Self-test)

**Q1:** 三種架構各自最大優點/缺點?
> Single-Leader:簡單無衝突 / leader 單點、所有寫入走同一節點。Multi-Leader:跨 DC 寫入延遲低、DC 故障容錯好 / 寫入衝突處理複雜。Leaderless:寫入可用性高、容忍節點故障 / 一致性弱、需應用層合併。

**Q2:** 什麼是 Read-After-Write Inconsistency?兩個解法?
> 用戶寫入後立刻讀卻讀到舊值(被路由到還沒同步的 follower)。解法:① 讀自己可能改過的東西從 leader 讀 ② 追蹤最後寫入時間,短時間內強制從 leader 讀。

**Q3:** Quorum 條件公式?n=3 時 w、r 通常多少?
> **w + r > n**(確保讀寫節點有交集)。n=3 時通常 w=2, r=2(容忍一個失效)。

**Q4:**(何者不是 Single-Leader Failover 的挑戰)(A)Split brain (B)Conflict resolution (C)非同步複製資料遺失 (D)Timeout 門檻難設
> **(B)**。Conflict resolution 是 Multi-Leader 的挑戰;Single-Leader 只有一個 leader 接受寫入,不會有寫入衝突。

**Q5:** 同步 vs 非同步 replication 核心取捨?
> 同步:保證 follower 有最新資料(一致性),但 follower 沒回應整個系統寫入卡住(犧牲可用性)。非同步:寫入更快、可用性高,但 leader 在 follower 同步前掛掉那些寫入永久消失(資料遺失風險)。折衷:半同步(一台同步、其他非同步)。

**Q6:** 什麼是 Sloppy Quorum 和 Hinted Handoff?
> Sloppy Quorum:原本該存資料的 n 台「家」節點連不到時,接受寫入到其他可達節點(提升寫入可用性)。Hinted Handoff:網路恢復後把臨時保管的寫入送回家節點。代價:即使 w+r>n 也無法保證讀到最新(最新值可能在臨時節點)。
