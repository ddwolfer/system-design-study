# 03_基本觀念 / 06. API Design｜API 設計 — 總覽 (INDEX MAP)

> 2026-06-03:原本 10 個 PDF 其實只是 5 個主題各重複一份(檔名亂貼 + 簡體副本)。**已清理**:刪掉 5 個重複/簡體檔、把留下的改成正確名。清理後 5 個繁體檔:

| 檔名(清理後)| 主題 | digest 檔 |
|---|---|---|
| `API Design.pdf` | 決策樹 | 本檔(下方)|
| `REST.pdf` | REST | `digest-rest.md` |
| `GraphQL.pdf` | GraphQL | `digest-graphql.md` |
| `RPC.pdf` | RPC | `digest-rpc.md` |
| `API Security.pdf` | API Security | `digest-api-security.md` |

> 已刪:`Pagination`(決策樹重複)、`Rate Limiting`(REST 簡體)、`Caching Headers`(GraphQL 簡體)、`Auth`(RPC 混簡體)、`Error Handling`(API Security 重複)。
> 驗證心得:Gemini 對「繁/簡」的快速判斷不可靠,要它**原樣引用 raw 原文**自己看字體才準(`查詢`=繁、`查询`=簡)。

---

## 決策樹(來源:`API Design.pdf`)

```
這 API 給誰用?
 ├─ 內部 (internal) ─────────────→ RPC(如 gRPC)
 └─ 外部 (external) → 在意 over-/under-fetching?
                       ├─ Yes ──→ GraphQL
                       └─ No ───→ REST
```
- over-fetching = 一個 endpoint 回一坨,拿到用不到的欄位;under-fetching = 一個 endpoint 不夠,要打好幾次湊資料。GraphQL 一次查詢解掉這兩個。
