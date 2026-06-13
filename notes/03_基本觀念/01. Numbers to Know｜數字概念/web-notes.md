# Numbers to Know｜要記住的數字

> 現代硬體規模已遠超想像:[[redis|Redis]] 能上 TB、[[postgresql|PostgreSQL]] 單機撐數十 TB、[[kafka|Kafka]] 單 broker 百萬訊息/秒。真正的瓶頸通常是 **[[throughput|每秒操作數]] 或網路頻寬,而不是記憶體大小**——這跟幾年前完全相反。先記住這些「肌肉記憶數字」,設計時才不會過早 [[sharding|分片]]。

## Caching｜快取 (Redis)

現代記憶體快取容量、效能都遠超過去。很多時候與其設計複雜的部分快取策略,不如乾脆「把所有資料都放進快取」。

| 指標 | 數字 |
| :--- | :--- |
| 記憶體容量 | 最高約 1TB（特殊配置更多） |
| [[latency]] | 同區域讀取 < 1ms；跨區寫入 1–2ms |
| [[throughput]] | 單節點 100k+ 讀取/秒；寫入數十萬/秒 |

**何時需要 [[sharding]]**:資料集逼近 1TB、長期吞吐 > 100k ops/sec、讀取 latency 需穩定 < 0.5ms。

## Databases｜資料庫 (PostgreSQL / MySQL / Aurora)

單一實例就能處理數十 TB 資料並保持毫秒級 latency。限制往往不是效能,而是備份、跨區複寫等營運考量。

| 指標 | 數字 |
| :--- | :--- |
| 儲存 | 64TiB；[[aurora|Aurora]] 可達 256TiB |
| [[latency]] | 快取讀取 1–5ms；磁碟讀取 5–30ms；commit 5–15ms |
| [[throughput]] | 讀取 50k [[tps|TPS]]；寫入 10–20k TPS |
| 連線數 | 5,000–20,000 |

**何時需要 [[sharding]]**:資料逼近 50TiB、寫入長期 > 10k TPS、未快取讀取要求 < 5ms、需跨區域佈署、或備份時間過長。

> ⚠️ 很多人在只有 500GB–2TB 時就急著談 sharding。良好調校的單機可撐**數百萬甚至上千萬用戶**——先算清楚,再談分片。

## Application Servers｜應用伺服器

伺服器資源比過去豐富許多,許多傳統設計限制已不再成立。通常第一個瓶頸是 **[[cpu|CPU]],而不是 RAM 或連線數**。

| 指標 | 數字 |
| :--- | :--- |
| 連線數 | 單機 100k+ concurrent |
| CPU | 8–64 核心 @ 2–4GHz |
| 記憶體 | 64–512GB（最高 2TB） |
| 網路 | 25Gbps |
| 啟動時間 | Container apps 30–60 秒 |

**何時需要 [[scaling|Scaling]]**:CPU 長期 > 70–80%、latency 無法符合 [[sla|SLA]]、記憶體長期 > 70–80%、頻寬逼近 20Gbps。雖然 [[stateless|stateless]] 設計對擴展重要,但單機記憶體已很大,可直接拿來做快取、計算或 session 管理。

## Message Queues｜訊息佇列 (Kafka)

訊息佇列已演進成高效能「數據高速公路」。因 latency 能穩定在 5ms 內,它甚至可直接用在 **[[synchronous-api|Synchronous API]]** 流程,不再只能做 async 背景處理;近乎無限的保存力,讓它能當整個資料流的骨幹。

| 指標 | 數字 |
| :--- | :--- |
| [[throughput]] | 每 broker 可達 1M 訊息/秒 |
| [[latency]] | 1–5ms（同區域） |
| 訊息大小 | 1KB–10MB |
| 儲存 | 單 broker 可達 50TB |
| 保存期 | 數週到數月 |

**何時需要 [[sharding]]**:throughput 逼近 800k msg/sec、partition 數逼近 200k、[[consumer-lag|consumer lag]] 持續增加、需跨區複寫。使用時仍要考慮重試與超時以確保可靠性。

## Cheat Sheet｜速查表

| Components | Key Metrics | Scale Triggers |
| :--- | :--- | :--- |
| Caching | latency ~1ms；100k+ ops/sec；記憶體 ~1TB | hit rate < 80%；latency > 1ms |
| Databases | 50k TPS；cached read < 5ms；儲存 64TiB+ | 記憶體 > 80%；churn/thrashing；寫入 > 10k TPS；uncached read > 5ms；需跨區 |
| App Servers | 100k+ 連線；8–64 核 @ 2–4GHz；64–512GB RAM | CPU > 70%；latency 超 SLA；連線逼近 10 萬/台；記憶體 > 80% |
| Message Queues | 1M 訊息/秒/broker；端到端 < 5ms；儲存 50TB | throughput 逼近 800k；分區 ~200k；consumer lag 持續增長 |

### 收尾小考

1. **配對題**:把 (1) Redis 單節點讀取延遲 (2) Kafka 單 broker 吞吐 (3) PostgreSQL cached read (4) Aurora/RDS 單節點寫入,對到 (a) ~1M/sec (b) <1ms (c) 10–20k TPS (d) 1–5ms。
2. 一個調校良好的單一資料庫能撐多少用戶?什麼時候才真的需要 [[sharding]]?
3. 現代系統中,真正的瓶頸通常是什麼?是記憶體大小嗎?
4. 為什麼說 Kafka 現在可以用在 [[synchronous-api|同步 API]] 流程,而不只是 async?

```glossary
{
  "redis": { "term": "Redis（記憶體快取）", "short": "最常見的 in-memory cache,讀取 [[latency]] < 1ms,單節點可撐數百 GB 到 TB,吞吐達數十萬 ops/sec。" },
  "postgresql": { "term": "PostgreSQL（關聯式資料庫）", "short": "主流開源 SQL DB,單機可處理數十 TB 並保持毫秒級延遲,常被低估其單機能耐。" },
  "kafka": { "term": "Kafka（分散式訊息佇列）", "short": "高吞吐 [[message-queue|訊息佇列]],單 broker 可達 1M 訊息/秒,並能保存數週到數月。" },
  "aurora": { "term": "Amazon Aurora（雲端關聯式資料庫）", "short": "AWS 託管的 MySQL/PostgreSQL 相容 DB,儲存可達 256TiB,讀取約 50k [[tps|TPS]]。" },
  "latency": { "term": "Latency（延遲）", "short": "一次操作從發出到收到回應的時間,通常以毫秒 (ms) 計;系統設計常要求穩定在某門檻內。" },
  "throughput": { "term": "Throughput（吞吐量）", "short": "單位時間能處理的操作數,如 ops/sec、TPS、訊息/秒。現代系統的瓶頸往往在此而非記憶體大小。" },
  "tps": { "term": "TPS（Transactions Per Second,每秒交易數）", "short": "資料庫吞吐的常用單位;[[aurora|Aurora]]/RDS 讀取約 50k、寫入約 10–20k。" },
  "sharding": { "term": "Sharding（分片）", "short": "把資料水平切到多個節點以突破單機上限。應在量化指標逼近上限時才做,別過早 [[sharding]]。" },
  "scaling": { "term": "Scaling（擴展）", "short": "增加機器或資源以應付負載,分為垂直（加大單機）與水平（加機器）。應用伺服器常因 [[cpu|CPU]] > 70–80% 而觸發。" },
  "cpu": { "term": "CPU（處理器）", "short": "應用伺服器最常見的第一瓶頸;現代單機有 8–64 核,使用率長期 > 70–80% 即該擴展。" },
  "sla": { "term": "SLA（Service Level Agreement,服務水準協議）", "short": "對外承諾的效能/可用性目標,例如延遲上限;[[latency]] 超過 SLA 是擴展訊號之一。" },
  "stateless": { "term": "Stateless（無狀態）服務", "short": "請求不依賴單機本地狀態,方便水平擴展;但單機記憶體已大,仍可拿來做快取或 session。" },
  "synchronous-api": { "term": "Synchronous API（同步 API）", "short": "呼叫方等待回應後才繼續的流程;因 [[kafka|Kafka]] latency < 5ms,訊息佇列也能進入同步流程。" },
  "message-queue": { "term": "Message Queue（訊息佇列）", "short": "在服務間傳遞訊息的中介,支援解耦、緩衝與重播;[[kafka|Kafka]] 是代表。" },
  "consumer-lag": { "term": "Consumer Lag（消費延遲）", "short": "訊息佇列中尚未被消費者處理的訊息堆積量;持續增長代表消費端跟不上,是 [[sharding]] 訊號。" }
}
```
