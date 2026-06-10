# 03_基本觀念 / 03. Client-Server Architecture｜客戶端-伺服器架構 — 投影片逐字原文

> 來源:`gemini_digest_pdf("03_基本觀念/03. Client-Server Architecture｜客戶端-伺服器架構")`,2026-06-02。
> 投影片本身即 ground truth,Gemini 僅做 OCR/轉錄。知識精華已蒸餾進 KG。

---

## Slide 1：什麼是 Client-Server Architecture

現代網路應用最基本的架構模式。幾乎所有系統設計面試的系統都建立在此模型上。
- **Client(客戶端)**:發起請求的一方(瀏覽器、手機 App、或另一個服務)。
- **Server(伺服器)**:接收請求、處理邏輯、回傳結果的一方。
兩者透過網路溝通,遵循 **Request-Response(請求-回應)** 模型。

例子:打開瀏覽器輸入網址 → 瀏覽器 (Client) 發 HTTP 請求 → 伺服器 (Server) 處理後回傳網頁 → 瀏覽器渲染。

**為什麼重要:**
1. **職責分離**:Client 負責展示/互動,Server 負責商業邏輯/資料處理 → 易維護、易擴展。
2. **集中管理**:資料與邏輯集中在 Server,方便更新、維護、安全管控。
3. **多客戶端支援**:同一 Server 可同時服務網頁、App、第三方 API。
4. **擴展基礎**:流量增加時對 Server 做垂直/水平擴展,不需改動 Client。

## Slide 2:Client 與 Server 的角色

**Client** 常見類型:瀏覽器(HTTP/HTTPS)、手機 App(API)、桌面程式(Slack/VS Code)、其他服務(微服務中一個服務可當另一個的 Client)。
典型職責:呈現 UI、收集輸入、發請求、接收並展示回應。

**Server** 通常含:Web Server(Nginx/Apache,收 HTTP)、Application Server(Node.js/Django/Spring Boot,執行邏輯)、Database(PostgreSQL/MongoDB)。
典型職責:驗證與授權 (Authentication & Authorization)、執行商業邏輯、讀寫資料庫、回傳結果。
> 實際上「Server」通常不是一台機器,而是多台伺服器組成的叢集,前面有 Load Balancer 分流。

## Slide 2–3:Thin Client vs Thick Client

- **Thin Client(瘦客戶端)**:大部分邏輯在 Server,Client 只負責顯示。例:傳統伺服器端渲染 (SSR),Server 產生完整 HTML,瀏覽器只渲染。優:Client 輕量、易維護、安全(邏輯不暴露)。缺:每次互動都要請求 Server,體驗可能較慢。
- **Thick Client(胖客戶端)**:Client 承擔較多邏輯,Server 主要提供資料。例:React/Vue 等 SPA,前端處理路由、狀態、UI 邏輯,只向 Server 要資料 (API)。優:體驗流暢、減少 Server 負擔。缺:前端複雜、部分邏輯暴露在 Client。

> 面試提示:現代系統設計通常假設 **Thick Client (SPA 或手機 App) + RESTful API**。設計重點在 Server 端,但 Client 選擇會影響 API 設計。

## Slide 3–4:Client-Server vs Peer-to-Peer (P2P)

| 特性 | Client-Server | Peer-to-Peer |
| :--- | :--- | :--- |
| 通訊方式 | Client 向 Server 請求 | 節點之間直接溝通 |
| 集中控制 | 有(Server 控制一切) | 無(去中心化) |
| 擴展方式 | 擴展 Server | 加入更多 Peer |
| 典型應用 | 幾乎所有 Web 應用 | 視訊通話 (WebRTC)、檔案分享 (BitTorrent) |

> 面試提示:面試中 99% 的問題都是 Client-Server。只有明確涉及視訊/音訊通話,才考慮 P2P (WebRTC)。

## Slide 4:在系統設計面試中的應用(畫架構圖的起點)

1. **從 Client 開始**:使用者透過什麼介面使用系統?(瀏覽器?App?)
2. **加入 Server**:處理請求的後端服務是什麼?
3. **定義 API**:Client 和 Server 怎麼溝通?(REST? GraphQL? WebSocket?)
4. **加入 Database**:Server 的資料存在哪裡?
5. **擴展**:流量大了怎麼辦?加 Load Balancer、Cache、CDN…

這就是為什麼 Client-Server 是基礎中的基礎——它是你畫出的第一條線,後續所有設計都在這框架上展開。

## 自我測驗 (Self-test)

**Q1:** Client 和 Server 的核心職責分別是什麼?
> Client:發起請求、呈現 UI、收集輸入、展示回應。Server:接收請求、驗證授權、執行商業邏輯、讀寫資料庫、回傳結果。

**Q2:** 為什麼 Client-Server 的職責分離很重要?
> 易維護與擴展:Server 可獨立更新邏輯/資料結構不影響 Client;同一 Server 服務多種 Client;流量增加時只擴展 Server。

**Q3:** Thin vs Thick Client 的主要差異?現代偏向哪種?
> Thin:邏輯在 Server,Client 只渲染(SSR)。Thick:Client 承擔較多邏輯,Server 提供資料 API(React SPA、手機 App)。現代偏向 Thick Client + RESTful API。

**Q4:** 什麼時候考慮 P2P 而非 Client-Server?
> 需要客戶端之間直接、低延遲通訊時(視訊/音訊通話 WebRTC)。面試除非明確涉及即時音視訊,否則預設 Client-Server。
