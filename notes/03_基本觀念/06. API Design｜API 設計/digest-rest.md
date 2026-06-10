# 06 API — REST(來源:`REST.pdf` 繁體;`Rate Limiting.pdf` 是簡體副本)

> 2026-06-03。已蒸餾進 KG(API Paradigms 節點)。

## 基本概念
REST (Representational State Transfer) 是一種**架構風格 (architectural style)**,基於 HTTP 原則。核心:把伺服器上的**資源 (resource)** 用 URI 表達,用標準 HTTP 方法操作。

## HTTP 動詞與冪等性 (idempotency)
**idempotency 定義**:一次操作和多次操作對伺服器狀態影響相同。
- **GET**:讀取資源、不改狀態、**idempotent**。
- **POST**:建立新資源、每次可能建不同 ID、**非 idempotent**。
- **PUT**:更新整個資源、多次相同內容結果不變、**idempotent**。
- **PATCH**:局部更新、**依設計而定**(一般不保證)。
- **DELETE**:刪除、刪一次或多次結果相同、**idempotent**。

## 如何傳資料給 API
- **Path Parameter**:放 URL 路徑,標示唯一資源(`GET /users/123`)。
- **Query Parameter**:篩選/排序/分頁(`GET /users?age=30&sort=desc&page=2`)。
- **Request Body**:POST/PUT/PATCH 傳結構化資料(JSON)。

## Response 設計
**HTTP 狀態碼**:`200 OK`、`201 Created`(常搭 `Location` header)、`400 Bad Request`、`401 Unauthorized`(未驗證身份)、`403 Forbidden`(有身份沒權限)、`404 Not Found`、`500 Internal Server Error`。
**Response Body**:主要內容(JSON)。**Meta 資訊**:分頁、超連結 (HATEOAS)、快取標頭。

## 優缺點
- 優點:簡單直觀、跨平台性好、**快取機制完善**(天然支援瀏覽器/CDN 快取)、分層架構友好(配合 Proxy/LB/API Gateway)。
- 缺點:**Over-fetching**(拿到不需要的欄位)、**Under-fetching**(一頁多資源要多次 call)、複雜操作難表達(批次/複雜 query)、版本控制問題(`/v1/`、`/v2/`)。

## 面試
預設選擇:沒特別限制時 REST 是最常見對外 API。對外公開 → REST 業界標準;需快取/廣泛相容 → REST;前端要避免 over-fetching → GraphQL;內部高效能 → gRPC。

### 自我測驗
- Q1: 哪些 HTTP 方法冪等?→ GET、PUT、DELETE(POST 非冪等;PATCH 依設計)。
- Q2: Path/Query/Body 用途?→ Path 標示唯一資源;Query 篩選排序分頁;Body 傳結構化資料(POST/PUT/PATCH)。
- Q3: Over/Under-fetching?→ Over:回傳欄位比需要的多;Under:一頁多資源要多次 call。
