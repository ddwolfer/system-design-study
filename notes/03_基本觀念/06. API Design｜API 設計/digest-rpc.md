# 06 API — RPC(來源:`Auth.pdf` / `API Gateway Pattern.pdf`,檔名貼錯;部分簡體)

> 2026-06-03。已蒸餾進 KG(API Paradigms 節點)。

## 基本概念
RPC (Remote Procedure Call):一種通訊模型,允許程式呼叫**遠端伺服器上的函式,就像呼叫本地函式一樣**。核心:隱藏網路細節。設計導向對比:REST 資源導向、GraphQL 查詢導向、**RPC 方法導向**(直接呼叫一個函式/服務)。

## 核心特徵
- **方法導向**:介面以「動作」為中心(`CreateUser()`、`GetUser()`),像 function call 而非用 URI 找資源。
- **跨語言協定支援**:gRPC/Thrift/Avro 用 **IDL (Interface Definition Language)** 定義介面,生成 stub/client library 讓不同語言互相呼叫。
- **高效序列化**:二進位格式(Protobuf/Thrift/Avro)取代 JSON → 傳輸更快更省。
- **雙向串流**:gRPC 支援 client/server/雙向 streaming,適合即時通訊或大流量。

## 常見操作(gRPC)
1. 定義 IDL:
   ```
   service UserService {
     rpc GetUser(GetUserRequest) returns (GetUserResponse);
     rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
   }
   ```
2. 產生 Client/Server Stub(client 呼叫 `GetUser()` → 底層自動轉成網路請求)。
3. 呼叫像本地函式:`user := client.GetUser(ctx, &GetUserRequest{Id: 123})`。

## 優缺點
- 優點:呼叫直觀(像本地函式)、效能高(Protobuf 二進位)、強型別(IDL 清晰契約跨語言)、支援串流。
- 缺點:**耦合度高**(Client/Server 依賴相同介面定義,版本管理難)、不如 REST 通用(要先生成 stub、無法直接 curl 測)、Debug/可觀察性差(二進位封包)、對外開放性差(適合內部微服務)。

## 面試
微服務間內部通訊 → gRPC;需高效能低延遲大量呼叫 → RPC;對外公開 API → REST/GraphQL。一句話:**REST/GraphQL 對外、RPC 對內**。

### 自我測驗
- Q1: gRPC 用什麼序列化?為何比 JSON 快?→ Protocol Buffers (Protobuf),二進位、更緊湊解析更快,吞吐約 JSON 10 倍。
- Q2: REST/GraphQL/RPC 設計導向?→ 資源導向 / 查詢導向 / 方法導向。
- Q3: 對外 vs 內部微服務各用什麼?→ 對外 REST(或 GraphQL);內部 gRPC(高效能、強型別、支援串流)。
