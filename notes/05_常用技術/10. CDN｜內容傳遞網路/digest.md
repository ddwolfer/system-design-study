# 05_常用技術 / 10. CDN｜內容傳遞網路 — CDN — digest (pre-read cache)
> 2026-06-07 pre-read。來源:CDN.pdf。**尚未入庫 KG**(預讀快取,日後上課時才蒸餾)。
> 若內容是簡體或檔名與內容不符,在此註記,但**不要**刪改任何檔案。
> 註記:本份內容為正體中文,檔名與內容相符。

---

## Slide 1

- **Verbatim text**:
    CDN

    # CDN (Content Delivery Network)

    ## 基本概念

    現代系統通常面向全球使用者提供服務,這使得將內容快速傳送給世界各地的使用者 變得具有挑戰性。使用者(以及面試官)期望快速的載入時間,而延遲可能導致不良 的使用者體驗與流量流失。內容傳遞網路(Content Delivery Network,CDN)是一種 使用分散式伺服器,根據使用者地理位置來傳送內容的快取系統。CDN 通常用於傳送 靜態內容,例如圖片、影片與HTML 檔案,但也可以用於傳送動態內容,例如 API 回 應。

    CDN 的運作方式是將內容快取在靠近使用者的伺服器上。當使用者請求內容時,CDN 會將請求導向距離最近的伺服器。如果該伺服器已經快取了該內容,CDN 會直接回傳 快取內容。如果該伺服器沒有快取該內容,CDN 會從原始伺服器(origin server)抓 取內容,將其快取在該伺服器上,然後再回傳給使用者。

    在面試中,CDN 最常見的應用場景是快取靜態媒體資產,例如圖片與影片。舉例來 說,如果你有一個像 Instagram 這樣的社群平台,你可能會使用 CDN 來快取使用者的 頭像圖片。這樣可以讓世界各地的使用者都能快速載入頭像圖片。

    [Image of a world map with servers and users]

    Legend:
    - Origin Server
    - CDN Server
    - User

    ## 架構特徵

- **Diagram**:
    The diagram illustrates the basic architecture of a Content Delivery Network (CDN) on a world map background.
    - **Components**:
        - **Origin Server**: A single server icon, colored orange, is located in North America. This represents the central repository of the original content.
        - **CDN Server**: Multiple server icons, colored blue, are distributed across the globe in locations like North America, South America, Europe, and Asia. These are the edge servers that cache content.
        - **User**: Several user icons are also spread across different continents.
    - **Relationships**:
        - An orange line connects the **Origin Server** to all the **CDN Servers**, indicating that the CDN servers pull content from the origin.
        - Red lines connect the **Users** to their nearest **CDN Server**. For example, a user in South America is connected to the CDN server in South America, and a user in Europe is connected to the CDN server in Europe. This shows that user requests are routed to the geographically closest edge server for faster delivery.
        - A blue arrow points from a CDN server in North America back towards the Origin Server, possibly indicating a cache miss scenario where the CDN server needs to fetch data from the origin.
        - Another blue arrow points from a CDN server in Asia towards a user, illustrating the delivery of cached content to the end-user.

---

## Slide 2

- **Verbatim text**:
    *   **Edge Node (邊緣節點)**
        *   最靠近使用者的伺服器,負責回應快取內容。
    *   **Origin Server (源站)**
        *   原始資料存放位置(例如應用伺服器或 Blob Storage)。
    *   **Cache Hierarchy**
        *   多層快取設計:Edge Cache → Regional Cache → Origin。
    *   **協定支援**
        *   HTTP/HTTPS,部分 CDN 支援 WebSocket 與 API 加速。
    *   **全球分佈**
        *   多個 PoP (Point of Presence),減少跨國延遲。

    ## 常見功能

    *   **內容快取 (Caching)**
        *   對靜態檔案(CSS、JS、圖片、影片)進行快取。
    *   **動態加速 (Dynamic Content Acceleration)**
        *   TCP 優化、路由優化,縮短跨國傳輸延遲。
    *   **安全 (Security)**
        *   DDoS 防護、WAF (Web Application Firewall)、Bot 防護。
    *   **流量壓縮與優化**
        *   Gzip / Brotli 壓縮、圖片壓縮、自動轉 WebP。
    *   **Edge Compute (邊緣運算)**
        *   部分 CDN(Cloudflare Workers、Akamai EdgeWorkers)支援在邊緣節點上 執行程式邏輯。

    ## 常見產品與服務

    *   **商業 CDN**
        *   Cloudflare
        *   Akamai

- **Diagram**:
    This slide does not contain a diagram.

---

## Slide 3

- **Verbatim text**:
        *   Fastly
    *   **雲端服務商提供的CDN**
        *   AWS CloudFront
        *   Azure CDN
        *   Google Cloud CDN

    ## 適用場景

    *   **靜態資源分發**
        *   圖片、影片、JS/CSS、HTML
    *   **API 加速**
        *   提高 API 回應速度,減少跨區延遲
    *   **大型活動流量高峰**
        *   遊戲發佈、線上演唱會、直播
    *   **安全防護**
        *   防止惡意流量直接打到源站

    ## Cache Invalidation Strategy in CDN

    *   **TTL (Time-to-Live)**
        *   靜態檔案設 TTL,例如圖片設一週,CSS 設一天
    *   **版本號 / Hash Busting**
        *   在 URL 加上版本號或 hash,例如: `app.css?v=12345`
    *   **Purge API**
        *   主動通知 CDN 清除指定檔案快取
    *   **Stale-while-revalidate**
        *   提供過期但仍可用的快取,同時在背景更新

    ## 在系統設計面試中如何談 CDN

    *   不要在所有流量前都畫 CDN → 只針對靜態資源與高頻內容。

- **Diagram**:
    This slide does not contain a diagram.

---

## Slide 4

- **Verbatim text**:
    *   **正確表達方式:**
        *   系統對靜態檔案(圖片、影片、CSS/JS)使用CDN,減少延遲與源站壓力。
        *   API 若有全球用戶,也可透過 CDN 進行加速。
        *   若需要保護源站(origin),CDN 也可作為安全防護層(DDoS/WAF)。
    *   **常見面試追問**
        *   如何確保快取更新? → TTL + Hash Busting
        *   動態內容怎麼處理? → Edge Compute / API 加速
        *   CDN 掛掉怎麼辦? → 設定 fallback 到 Origin

- **Diagram**:
    This slide does not contain a diagram.
