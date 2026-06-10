# 07_真實大型應用設計 / 01. Design QR Code Generator｜QR Code 生成器 — digest (pre-read cache)
> 2026-06-08 pre-read。來源:Design QR Code Generator PDF。此課另有影片(.mp4),預讀只做 PDF;影片留待現場上課時用 Gemini 看。**尚未入庫 KG**。
> 若內容是簡體或檔名與內容不符,在此註記,但**不要**刪改任何檔案。

---

## Slide 1

-   **Verbatim text**:
    > **Design a QR code Generator**
    >
    > QR code generator 是一種工具 (software、library 或 service), 用來接收輸入資料 (例如 URL、email、或 unique ID), 並將其轉換成 QR code 圖像——一種由黑白方塊組成的二維 (2-D) 矩陣, 可被相機或 App 掃描。
    >
    > **功能性需求 (Functional Requirements)**
    >
    > *   使用者可以上傳指定網址 (ASCII, 最長 20 字元), 服務會回傳產生的 QR code。
    > *   使用者可以管理自己建立過的 QR Code。
    > *   當使用者掃描 QR code 時, 會被重新導向 (redirect) 到原始的 URL。
    >
    > **非功能性需求 (Non-Functional Requirements)**
    >
    > *   High Availability: 服務需 24/7 可用。
    > *   Redirection 延遲需極低 (<100ms)。
    > *   支援 10 億個 QR codes 與 1 億使用者。
    >
    > **API 設計 (API design)**
    >
    > ```
    > API
    > # Create a QR Code
    > POST v1/qr_code
    > {
    >     url
    > }
    > Response: {
    >     qr_token
    > }
    > ```
-   **Diagram**: This slide does not contain a diagram.

## Slide 2

-   **Verbatim text**:
    > ```
    > # Get QR Code image
    > GET v1/qr_code_image/:qr_token
    > {
    >     image_spec: {
    >         dimension,
    >         color,
    >         border,
    >         ...
    >     }
    > }
    > Response: {
    >     image_location
    > }
    > 
    > # Edit a QR Code
    > PUT v1/qr_code/:qr_token {
    >     url
    > }
    > 
    > # Delete a QR Code
    > DELETE v1/qr_code/:qr_token
    > 
    > # Get the original url
    > GET v1/qr_code/:qr_token
    > Response: {
    >     url
    > }
    > ```
    >
    > **High-Level Design**
    >
    > 1.  QR Code Creation / Edit Flow
-   **Diagram**: This slide does not contain a diagram.

## Slide 3

-   **Verbatim text**:
    > *   使用者呼叫 POST `v1/qr_code`, 並在 request body 中帶入 url。
    > *   服務先驗證 URL 是否合法。若合法,則產生一個全域唯一的 token: `qr_token`, 並在 `QrCodes` table 中建立一筆資料。
    >     *   暫時假設我們有一個 black box 可以產生 unique token, 細節會在 deep dive 說明。
    > *   `qr_token` 在所有使用者之間必須全域唯一 (由 DB schema 保證)。
    > *   Server 回傳包含 `qr_token` 的 response。
    >
    > **2. QR Code Retrieval Flow**
    >
    > *   使用者呼叫 GET `v1/qr_code_image/:qr_token`, 並在 request body 中指定 image spec。

-   **Diagram**:
    *   **First Diagram (QR Code Creation/Edit Flow):**
        *   **Components:** The diagram shows four main components: `Client`, `API Gateway`, `QR Code Service`, and a `Database`.
        *   **Flow:** A two-way arrow connects `Client` and `API Gateway`. Another two-way arrow connects `API Gateway` and `QR Code Service`. Finally, a two-way arrow connects `QR Code Service` and `Database`. This represents a request/response flow from the client, through the gateway, to the service, which then reads from or writes to the database.
        *   **Database Schema:** The `Database` component points to a table schema named `QrCodes` with the following columns: `id`, `user_id`, `qr_token`, `url`, `created_at`.
    *   **Second Diagram (QR Code Retrieval Flow):**
        *   **Components:** The diagram is identical to the first one, showing `Client`, `API Gateway`, `QR Code Service`, and a `Database`.
        *   **Flow:** The flow is identical to the first diagram, illustrating a request from the client being processed by the system to retrieve data.
        *   **Database Schema:** The `Database` component again points to the `QrCodes` table schema with the columns: `id`, `user_id`, `qr_token`, `url`, `created_at`.

## Slide 4

-   **Verbatim text**:
    > *   Server 依照 token 產生 QR code 圖像, 然後回傳 image resource location。
    > *   QR Code 內嵌的 URL 會是 `https://myqrcode.com/qr_token`。當使用者連線時, backend 會查詢 `QrCodes` table, 並回傳 HTTP redirect 到原始 URL。常見有兩種 redirect:
    >     *   301 (Permanent Redirect): 瀏覽器會快取結果, 未來可能直接跳過我們的 server。
    >     *   302 (Temporary Redirect): 瀏覽器不會快取, 確保每次都經過我們的 server。
    >     *   這裡選擇 302, 因為我們希望 QR code 擁有者可以刪除或修改對應關係, 確保每次都取得最新狀態。
    > *   使用者也可呼叫 GET `v1/qr_code/:qr_token` 直接取得原始 URL (backend 同樣查詢 `QrCodes` table)。
    >
    > **深入探討 (Deep Dives)**
    >
    > **1. 如何產生唯一 token**
    >
    > 我們需要足夠的熵 (entropy) 來讓產生的 token 盡可能保持唯一性。為此, 可以使用像 SHA-256 這樣的雜湊函數來產生固定長度的 hash 值。雜湊函數會把輸入轉成一個確定性 (deterministic)、固定長度的字串。純雜湊函數是確定性的: 同一個長網址永遠會對應到同一個短碼, 而不需要查資料庫。
    >
    > 如果想避免這種「同輸入必得同輸出」的情況, 可以在輸入中加入一個 secret 或 nonce (在密碼學中指只使用一次、任意或不重複的隨機數值), 讓每次產生的結果都不同。
    >
    > 完成 hashing 之後, 可以把輸出再用編碼方式 (例如 Base62) 轉成文字, 讓二進位資料變成更短、更容易傳輸的字串。接著取前 N 個字元當作精簡後的唯一 token。N 的大小取決於 key space 的大小, 用來把 collision (碰撞) 機率降到可接受的程度。
    >
    > **Base62 說明**
    >
    > Base62 使用 62 個字元 `[0-9A-Za-z]` 來表示數字, 例如把 3842 轉成 Base62:
    >
    > 3842 ÷ 62 = 61 餘 60 → “y”
    >
    > 61 ÷ 62 = 0 餘 61 → “z”
    >
    > 反向讀取餘數 → `[61, 60]` 對照 Base62 字元表 → `zy`
-   **Diagram**: This slide does not contain a diagram.

## Slide 5

-   **Verbatim text**:
    > 即使如此, 當 key space 變得夠大時, 仍然可能發生 collision。為了解決這個問題, 可以在資料庫中把 `qr_token` 設為 UNIQUE。當發生碰撞時, 資料庫會回報錯誤, 我們再重新產生一個 token 並重試即可。
    >
    > **2. 如何確保 redirect 速度快**
    >
    > **Indexing (索引)**
    >
    > 為了避免 full table scan (全表掃描), 我們可以在資料庫中為 `qr_token` 建立 index。它可以是 primary index, 也可以是 secondary index。要注意的是, 如果把它設為 primary index, 就同時會具有 UNIQUE 的特性。
    >
    > Index 可以想像成書本的目錄或圖書館的卡片目錄, 它讓我們不用一頁一頁翻、一本一本找, 就能快速定位到需要的資料。
    >
    > **Caching (快取)**
    >
    > 我們的服務是 read-heavy (讀多寫少, write:read ≈ 1:100)。如果每一次讀取都直接打到資料庫, 當流量上來時, DB 的效能很快就會成為瓶頸。
    >
    > 舉例來說:
    >
    > 100,000,000 users × 5 redirects = 500,000,000 redirects / day
    >
    > 500,000,000 ÷ 86,400 seconds ≈ 5,787 redirects / second
    >
    > 其中一個作法是使用 cache, 把常被查詢的結果先存在快取中。每次讀取時先查 cache, 只有 cache miss 才打到 DB。寫入時則同時更新 DB, 並在必要時 (例如刪除) 做 cache invalidation。
    >
    > Cache 可以是:
    >
    > *   每台 server instance 的 local cache (實作簡單, 但 hit rate 低)
    > *   獨立的 distributed cache (例如 Redis / Memcached, hit rate 高, 但系統複雜度與成本也較高)
    >
    > 事實上, 大多數 DB 自身在存取磁碟前也有 buffer cache, 只是彈性與功能不如獨立的 cache layer 強。
    >
    > **CDN**
    >
    > CDN (Content Delivery Network) 就是把網站或 APP 的靜態內容 (圖片、影片、JS、CSS、QR 圖等) 預先複製到全球各地的 edge nodes, 讓使用者從最近的節點取得資料, 減少跨國或跨洲傳輸造成的延遲。
-   **Diagram**: This slide does not contain a diagram.

## Slide 6

-   **Verbatim text**:
    > 在這個設計中, 我們可以把 QR Code 和 URL mapping cache 在 CDN 節點上。對於熱門的 QR Code, URL redirection 甚至可以直接在 CDN 完成, 完全不需要經過我們的 application server, 大幅降低整體 latency。
    >
    > 在我們的 design 裡, QR Code 本身就是非常適合放在 CDN 的 static data。這樣一來, 對於高流量的 QR Code, request 可以直接由 CDN 回傳對應的內容, 而不必回源到我們的 server。
    >
    > **3. Scaling the System**
    >
    > *   Server 採用 stateless 設計, 表示在回應完成後不會保留任何 request state, 因此可以很容易透過增加 instance 來做 horizontal scaling。
    > *   Database:
    >     *   假設有 10 億 (1B) 筆資料, 每筆 200 bytes, 總量約為 200GB, 對於可預見的未來, 一台單一 DB instance 應該就足以承載。
    >     *   為了 fault tolerance (容錯), 可以加入多個 read replica, 因為讀取流量遠高於寫入。當 write replica 掛掉時, 可以把其中一個 read replica 升級為新的 write replica。
    > *   我們也需要考慮定期清理長時間沒有被點擊的 URL, 例如先通知使用者這些連結即將被刪除, 然後用 cron jobs 定期掃描資料庫並移除過期紀錄。
    > *   由於 QR code image 是靜態資源, 如果圖片請求的 QPS 很高, 又不希望每次都回源到 object store 讀取, 可以導入 CDN (Content Delivery Network), 讓不同地區的使用者都能以更低 latency 取得圖片。
-   **Diagram**: This slide does not contain a diagram.
