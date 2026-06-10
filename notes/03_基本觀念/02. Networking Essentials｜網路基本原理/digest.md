# 03_基本觀念 / 02. Networking Essentials｜網路基本原理 — 投影片逐字原文

> 來源:`gemini_digest_pdf("03_基本觀念/02. Networking Essentials｜網路基本原理")`,2026-06-02。
> 投影片本身即 ground truth,Gemini 僅做 OCR/轉錄。供「回撈某張投影片逐字原話」用;知識精華已蒸餾進 KG。

---

## 網路 101 與分層 (OSI)

網路建立在分層架構(OSI 模型)上,層次是一層層抽象,讓應用開發者用簡單語言思考通訊(像用 `open` 開檔,不必管磁碟怎麼讀 bytes)。系統設計面試最常出現三層:

- **網路層 Network Layer (L3)**:IP 協定,負責路由與定址。把資料拆成封包 (packet),提供盡力而為 (best-effort) 交付到目的 IP。
- **傳輸層 Transport Layer (L4)**:TCP、QUIC、UDP,提供端到端通訊;在 L3 之上加上可靠性、排序、流量控制。
- **應用層 Application Layer (L7)**:DNS、HTTP、WebSocket、WebRTC,建立在 TCP(或 UDP,如 WebRTC)之上。

完整 OSI:L7 Application(HTTP/DNS/WebSocket/WebRTC)、L6 Presentation(TLS/SSL/encoding)、L5 Session、L4 Transport(TCP/UDP/QUIC)、L3 Network(IP/routing)、L2 Data Link(Ethernet/MAC)、L1 Physical(cables/signals)。重點是 L3/L4/L7。

## 一個簡單的 Web 請求(在瀏覽器輸入 URL 按 Enter)

1. **DNS 解析**:把域名(www.buildmoat.org)轉成 IP(32.42.52.62)。
2. **TCP 三次握手 (Three-Way Handshake)**:SYN(客戶端請求建線)→ SYN-ACK(伺服器確認)→ ACK(客戶端完成建線)。
3. **HTTP 請求**:連線建立後送 HTTP GET。
4. **伺服器處理**:取得網頁、準備回應(這通常是工程師唯一會想到並能控制的延遲!)。
5. **HTTP 回應**:把網頁內容送回。
6. **TCP 四次揮手 (Four-Way Teardown)**:FIN → ACK → FIN → ACK。

值得注意:① 應用層可大幅簡化心智模型(TCP 保證有序可靠、DNS/IP 負責找到伺服器);② 概念上一個請求/回應,背後封包遠不止這些,帶來可忽略的延遲「直到忽略不了為止」;③ 連線是雙方要維護的狀態 —— 除非用 HTTP keep-alive 或 HTTP/2 多路複用 (multiplexing),否則每個請求都要重做建線,開銷可觀(設計持久連線系統時關鍵)。

## 傳輸層協定:TCP / UDP / QUIC

面試真正要做的選擇是 **TCP vs UDP**。QUIC 是較新協定,可視為「更好版本的 TCP」,但尚未廣泛普及。

**UDP(快速但不可靠)** —「機關槍 / 向前衝聽天命」。在 IP 之上幾乎沒加東西:
- 無連線(不需握手)、不保證送達、不保證順序、更低延遲。
- 適合:直播串流、線上遊戲、VoIP、DNS 查找 —— 速度比可靠性重要、可容忍偶爾丟包。VoIP 丟一個包就音訊小卡頓,比重傳塞爆網路好。
- 注意:除 WebRTC 外瀏覽器對 UDP 支援不廣。

**TCP(可靠但有開銷)** — 網際網路主力:
- 面向連線(三次握手)、可靠交付、維護順序、流量控制、壅塞控制。連線叫 stream,同一 stream 內訊息照序到達。沒被 ACK 就重傳。

| 特性 | UDP | TCP |
|---|---|---|
| 連線方式 | 無連線 | 面向連線 |
| 可靠性 | 盡力而為 | 保證交付 |
| 順序 | 不保證 | 維護順序 |
| 流量控制 / 壅塞控制 | 無 | 有 |
| 標頭大小 | 8 bytes | 20-60 bytes |
| 速度 | 較快 | 因開銷較慢 |
| 使用場景 | 串流、遊戲、VoIP | 其他所有情況 |

**選哪個**:面試預設 TCP(通常不必說明)。能為 UDP 提出合理論據(低延遲關鍵、可容忍丟包、高流量遙測、不需瀏覽器)而不搞砸細節 → 加分。現代應用常兩者並用(如視訊會議用 TCP/HTTP 做信令+驗證,UDP/WebRTC 做音視訊串流)。

## 應用層協定

**HTTP/HTTPS**:請求-回應、**無狀態 (stateless)**(每個請求獨立,伺服器不需記前一個 → 好事,盡量縮小有狀態部分)。
- 方法:GET(請求資料,應冪等,無 body)、POST(送資料)、PUT(更新)、PATCH(部分更新)、DELETE(刪除,應冪等)。
- 狀態碼:2xx 成功(200 OK、201 Created);3xx 重定向(301/302);4xx 客戶端錯誤(404、401、403、429 Too Many Requests);5xx 伺服器錯誤(500、502 Bad Gateway)。
- Headers = 鍵值對 metadata(如 Accept-Encoding 讓客戶端表明能處理 gzip/brotli)。
- HTTPS = HTTP + TLS/SSL 加密,公開網站一律要用。⚠️ 安全提醒:HTTPS 只保證加密,**不保證請求由你的客戶端產生**;API 在沒驗證前永遠別信任 body 內容(常見錯誤:body 帶 user ID 直接拿去查 DB → 攻擊者改 ID 讀任意用戶資料)。

**API 三範式:REST / GraphQL / gRPC**
- **REST**:面試最常用、預設選擇。對「資源 (resource)」做操作(資源≈DB 表/檔案)。用 HTTP 動詞 + 路徑慣例,JSON 表示。`GET /users/{id}`、`PUT /users/{id}`、`POST /users`、`GET /users/{id}/posts`。核心實體 (Core Entities) 通常直接對應資源。`updateUser` 是 operation 不是 resource → 不 RESTful,應為 `PUT /users/{id}`。
- **GraphQL**(2015 Facebook):讓客戶端精確請求所需資料,解決 under-fetching(要多次往返)與 over-fetching(回太多)。適合前端要快速迭代、多團隊重疊查詢。面試裡好處較模糊(需求固定),只在明顯聚焦彈性時提。
- **gRPC**(Google,HTTP/2 + Protocol Buffers):Protobuf 像 JSON 但有更嚴格綱要、二進位編碼更省空間/CPU(同資料 JSON 40 bytes vs Protobuf 15 bytes);吞吐某些基準達 10 倍。強型別在編譯時抓錯。適合內部微服務間通訊。**策略:內部 API 用 gRPC,外部 API 用 REST**。面試提示:別過早優化協定選擇(過早優化是萬惡之源)。

**即時推送協定:SSE / WebSocket / WebRTC**
- **SSE (Server-Sent Events)**:HTTP 之上的巧妙改裝,伺服器在單一 HTTP 連線/回應中隨時間持續串流多則訊息(`data: {...}` 一行一則),客戶端逐行即時處理。限制:連線不能太久(會被 LB/代理關)、EventSource 斷線會用 last event ID 自動重連、伺服器要追蹤斷線期間漏掉的訊息、不規範網路可能批次聚合。適合:客戶端一有事件就要立刻收到(如競標出價即時看當前價)。
- **WebSocket**:持久、類 TCP 的雙向連線,伺服器可主動推、客戶端可隨時推,廣泛支援(含瀏覽器)。透過 HTTP「升級 (upgrade)」發起(可沿用 cookies/headers)。⚠️ 中間每個基礎設施元件(防火牆/代理/LB)都要支援 WebSocket。適合:高頻、持久、雙向(即時應用、遊戲)。⚠️ 面試警告:沒理由就跳 WebSocket = thumbs down;有狀態連線在規模下開銷大。
- **WebRTC**:瀏覽器間點對點 (peer-to-peer),唯一用 UDP 的應用層協定。透過信令伺服器 (signaling server) 交換 peer 連線資訊,再直連。多數客戶端在 NAT 後 → 用 **STUN**(打洞 hole punching,取得可公開路由的 address/port)或 **TURN**(中繼伺服器轉送)。⚠️ 極難做對,小眾解法,面試只建議用在視訊/音訊通話會議。

## 負載平衡 (Load Balancing)

擴展兩選擇:更大的伺服器(垂直擴展 vertical)或更多伺服器(水平擴展 horizontal)。2020- 年代盡量垂直擴展(現代硬體很強),但面試最常見是水平擴展 → 需要負載平衡告訴客戶端跟哪台溝通。

**客戶端負載平衡 (Client-side LB)**:客戶端自己向 service registry 要可用伺服器列表,直接選一台發請求(可選最快的、不增延遲;只需定期同步列表)。
- 例 **Redis Cluster**:節點間用 gossip 協定互通;客戶端 hash key 決定分片→直連該節點;發錯節點會回 MOVED。
- 例 **DNS**:解析器回傳輪換 (rotated) 的 IP 列表,不同客戶端打到不同伺服器;設兩個不同區域的 LB + DNS 輪換 → 避免 LB 單點故障。
- 何時用:① 少量可控客戶端(Redis Cluster、gRPC 內部服務),或 ② 大量客戶端但可容忍緩慢更新(DNS)。

**專用負載平衡器 (Dedicated LB)**:位於客戶端與後端間的伺服器/硬體,決定路由。每請求多一跳,換得快速更新列表 + 細粒度路由控制。
- **L4 LB**(傳輸層):依 IP/port 路由,不看內容;維護持久 TCP 連線、快、開銷低;無法依應用層資料路由。同一 TCP session 後續請求都到同一台 → 適合 **WebSocket** 等持久連線。
- **L7 LB**(應用層):理解 HTTP,可依 URL/header/cookie 路由(類 API Gateway);終止傳入連線並對後端建新連線;CPU 消耗較高、功能更多。適合除 WebSocket 外所有 HTTP 流量。
- **健康檢查與容錯**:LB 監控後端健康,壞了就停止路由直到恢復(自動 failover → 高可用)。TCP 健康檢查(檢查能否接受連線)、L7 健康檢查(發 HTTP 確認 200)。
- **演算法**:Round Robin(輪詢)、Random、Least Connections(最少連線)、Least Response Time(最快回應)、IP Hash(依客戶端 IP,適合 session 持久)。無狀態應用通常輪詢/隨機即可;持久連線服務(SSE/WebSocket)用 least connections 避免某台累積所有連線。
- 實作:硬體(F5 BIG-IP,每秒幾億請求)、軟體(HAProxy/NGINX/Envoy)、雲端(AWS ELB/ALB/NLB 等)。

## 區域化與延遲 (Regionalization & Latency)

全球服務伺服器分散各地。常見模式:單一區域內有多個資料中心(Amazon 稱「**可用區域 Availability Zones**」),一棟樓斷線不癱瘓整個服務;再在全球各城市複製此模型。

**物理距離顯著影響延遲 —— 光速限制**:紐約↔倫敦 <1ms(附近) vs >80ms。光在光纖約 2/3 真空光速(~200,000 km/s),NY↔London 往返(~5,600km)理論最低延遲約 **56ms**(還沒算處理時間)。→ 回到**資料局部性 (data locality)** 原則:資料盡量靠近需要它的計算。

- **CDN (Content Delivery Network)**:策略性分佈全球的伺服器(數百~數千城市的「邊緣位置 edge location」)。邊緣能回答 → 極快(資料就在附近),靠快取實現,對靜態內容(圖片/影片/資源)特別有效。面試:資料可快取性高且需全球查詢時常用。
- **區域分片 (Regional Partitioning)**:單一區域大量用戶時,按區域分片資料,每區只存相關資料。例 Uber:在邁阿密叫車不會想預約紐約的司機 → 把附近城市捆成本地區域(「美國東北」),每區有自己的 co-located 資料庫,區域服務查本地庫 → 非常快。

## 處理故障與失敗模式

最危險的假設:「網路是可靠的」。永遠假設網路呼叫會失敗/延遲/回意外結果來設計。

- **逾時 + 重試 + 退避 (Timeouts, Retries with Backoff)**:設逾時、超時放棄重試。重試處理暫時故障(關鍵:API 要冪等)。但重試是雙刃劍 → 加 **退避 (backoff)**(失敗後等久一點再試)。退避要加 **抖動 (jitter,隨機性)**,否則所有客戶端同時重試像打樁機(thundering herd)。面試魔法短語:「指數退避重試 (retry with exponential backoff)」+ jitter。
- **冪等性 (Idempotency)**:冪等 API 可被呼叫多次、結果相同。GET 天生冪等。寫入場景用 **冪等鍵 (idempotency key)**(如 user ID + 日期),伺服器檢查該鍵是否已處理,只處理一次 → 避免支付重複扣款。
- **熔斷器 (Circuit Breaker)**:處理級聯故障 (cascading failures) / thundering herd(DB 崩潰重啟時,湧入的重試讓實例起不來)。三狀態:**Closed**(正常,監控失敗數)→ 失敗超閾值 → **Open**(請求立即失敗、不實際呼叫)→ 逾時後 **Half-Open**(放少量測試請求決定閉合或保持開路)。優點:快速失敗、減少負載、自我修復、改善 UX、系統穩定。適合:外部第三方 API、DB 連線/查詢、微服務間通訊、可能逾時的重操作。

## 總結

- 理解基礎:IP 定址、DNS、TCP/IP 模型
- 了解協定:TCP vs UDP;HTTP/HTTPS;REST/GraphQL/gRPC;SSE;WebSocket;WebRTC —— 各自用途與取捨
- 掌握負載平衡:客戶端 LB(Redis Cluster、DNS)與專用 LB(L4 vs L7)
- 規劃實際問題:區域化、CDN、重試退避、冪等性、熔斷器
- 網路決策影響系統每個面向(延遲、吞吐、可靠性、安全);面試要根據系統具體需求論證取捨,沒有單一正確答案。

## 自我測驗 (Self-test)

**Q1:** TCP 和 UDP 最根本的差異?各自適合什麼場景?
> TCP:面向連線、保證可靠交付與順序、有流量/壅塞控制,Header 20-60 bytes,適合幾乎所有應用(HTTP/API/檔案傳輸),面試預設。UDP:無連線、不保證送達與順序、Header 僅 8 bytes,適合低延遲且可容忍丟包(串流/遊戲/VoIP)。

**Q2:** 以下哪個 HTTP method 不是冪等的?(A) GET (B) PUT (C) DELETE (D) POST
> (D) POST。GET/PUT/DELETE 都冪等(多次呼叫結果相同),POST 不是。

**Q3:** REST、GraphQL、gRPC 各自最適合的場景?
> REST:外部 API 預設,簡單、跨平台、易快取。GraphQL:前端需靈活查詢且資料結構複雜時,避免 over/under-fetching。gRPC:內部微服務間,二進位序列化 (Protobuf),吞吐約 JSON 10 倍。策略:內部 gRPC、外部 REST。

**Q4:** L4 和 L7 Load Balancer 的差異?WebSocket 用哪個?
> L4:傳輸層,依 IP/port 路由,不理解 HTTP,快、開銷低,維護持久 TCP 連線。L7:應用層,理解 HTTP,可依 URL/header/cookie 路由,功能強但 CPU 高。WebSocket 用 L4(長連線,L4 維持持久 TCP)。

**Q5:** Circuit Breaker 的三個狀態?解決什麼問題?
> Closed(正常,監控失敗)、Open(失敗超閾值,請求立即失敗不呼叫)、Half-Open(逾時後試少量請求測試恢復)。解決級聯故障 —— 下游不健康時不斷重試只會更糟;熔斷器讓請求快速失敗,給下游恢復時間。

**Q6:** 為什麼重試策略需要加入「抖動 (jitter)」?
> 若大量客戶端同時同步重試,會形成突刺流量 (thundering herd),讓已掙扎的服務更不堪負荷。加隨機抖動讓重試時間錯開、分散負載。
