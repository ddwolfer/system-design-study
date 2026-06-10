# 06 API — API Security: Authentication & Authorization(來源:`API Versioning.pdf` / `Error Handling.pdf`,檔名貼錯;繁體)

> 2026-06-03 蒸餾;2026-06-07 已入庫 KG(節點 `API Security: Authentication vs Authorization` = 78c9781f,principle;refines API Paradigms 5919ac22)。

## 基本概念
- **Authentication(身份驗證)= Who you are**:確認使用者/服務身份是否合法。例:帳密登入、API key 或 Token 驗證。
- **Authorization(授權)= What you can do**:決定已驗證身份者是否有權限執行某操作/存取某資源。例:Admin 可 `DELETE /users/123`,普通使用者只能 `GET /users/123` 看自己。

## 常見 Authentication 方式
- **API Key**:最簡單,header 帶一組 key。缺點:洩漏風險高、缺細粒度權限。
- **Basic Auth**:`username:password` Base64 編碼放 header。簡單但安全性差。
- **Token-based (Session Token / JWT)**:登入後伺服器發 token,之後呼叫帶 token。
  - **JWT (JSON Web Token)**:內含使用者資訊與簽章,可 **stateless 驗證**。
  - **Access Token**:短期(幾分鐘~幾小時),每次 API 呼叫驗證用。
  - **Refresh Token**:長期,Access Token 過期時用來換新的。
  - **Expire Time**:Access Token 有明確到期時間,避免被盜長期有效。
- **OAuth 2.0**:第三方 API 授權(Google/Facebook/GitHub 登入)。流程:使用者同意授權 → 發 access token → API 用 token 驗證。
- **mTLS (Mutual TLS)**:雙方都出示憑證,常用於**內部微服務間**安全通訊。

## 常見 Authorization 模型
- **RBAC (Role-Based Access Control)**:依**角色**控制權限。Admin → `GET/DELETE /users`;Editor → `POST/PUT /articles`;Viewer → 只能 `GET`。場景:公司內部管理系統。
- **ABAC (Attribute-Based Access Control)**:依**屬性**(時間/地點/部門等)決定。例:部門=HR 可存取 `/payroll`;只有辦公室 IP 網段才能呼叫 `/internal-api`;文件標 Confidential 只有 clearance ≥3 可讀。場景:金融、政府,需細緻控制。
- **Scope-based (OAuth Scopes)**:權限隨 Token 的 scope 發放。GitHub:scope=`repo` 可存取私人 repo、`read:user` 只讀基本資訊、`write:org` 可改組織。場景:第三方應用授權。

## API 設計實例
```
GET /users/123
Authorization: Bearer <JWT_TOKEN>
```
- Authentication:驗證 JWT 是否有效(合法用戶)。
- Authorization:token 是普通用戶只能查自己 `id=123`;是 Admin 則可查他人。

## 重點
Authentication 確認「你是誰」,Authorization 決定「你能做什麼」,是保護系統安全的核心。

### 自我測驗
- Q1: Authentication vs Authorization?→ 前者確認「你是誰」(驗證身份合法),後者決定「你能做什麼」(權限)。
- Q2:(配對)正確的是?(A)Authentication=你能做什麼 (B)Authorization=你是誰 (C)JWT 通常放 `Authorization: Bearer` header (D)OAuth 2.0 主要用於內部微服務 → **(C)**。(D 錯,內部微服務用 mTLS,OAuth 2.0 用於第三方授權。)
- Q3: RBAC vs ABAC?→ RBAC 依角色(Admin 可 DELETE、Viewer 只 GET);ABAC 依屬性(只有辦公室 IP 才能呼叫內部 API)。
