# Database Transactions 資料庫交易

> 真實系統很少「一步完成」:轉帳要從 A 扣、給 B 加(兩步缺一不可)。[[transaction|Transaction]] 把一組操作包成**一個邏輯單元**,保證整體執行——要麼全成、要麼全不做。

## 1. ACID — 交易的四個保證

```mermaid
flowchart TB
  A["A 原子性 Atomicity<br/>全做或全不做"]
  C["C 一致性 Consistency<br/>合法狀態→合法狀態"]
  I["I 隔離性 Isolation<br/>並發互不干擾"]
  D["D 持久性 Durability<br/>commit 後永不遺失"]
```

- **[[atomicity|Atomicity 原子性]]**:全部成功或全部失敗,沒有中間狀態。失敗就 [[rollback|rollback 回滾]] 撤銷已做的。靠 [[wal|WAL]] 實現。
- **[[consistency-acid|Consistency 一致性]]**:交易前後所有 constraints 都成立(如「餘額不能為負」)。⚠️ 這個 C **不是** [[cap-theorem|CAP]] 的 C(那是複本同步),這裡指**商業邏輯正確性**。
- **[[isolation|Isolation 隔離性]]**:並發的多個交易互不干擾,每個看起來像獨立執行。**四者中最複雜、最有取捨**。
- **[[durability|Durability 持久性]]**:一旦 commit,系統立刻崩潰也不遺失(寫磁碟 + WAL 重建)。

## 2. 隔離等級 (Isolation Levels)

「完全隔離」最安全但最慢。實務用不同等級,在「安全」和「並行效能」之間取捨。等級越高,擋掉越多異常:

| 隔離等級 | Dirty Read | Non-repeatable Read | Phantom Read |
|---|---|---|---|
| Read Uncommitted | 可能 | 可能 | 可能 |
| Read Committed(**PostgreSQL 預設**) | 防止 | 可能 | 可能 |
| Repeatable Read(**MySQL InnoDB 預設**) | 防止 | 防止 | 可能 |
| Serializable | 防止 | 防止 | 防止 |

## 3. 三種並發異常(隔離等級在擋的東西)

- **[[dirty-read|Dirty Read 髒讀]]**:讀到別人**還沒 commit** 的資料,對方一 rollback,你就讀到「從未存在」的值。
- **[[non-repeatable-read|Non-repeatable Read 不可重複讀]]**:同一交易內讀**同一筆**兩次,值不同(別人中間改了並 commit)。
- **[[phantom-read|Phantom Read 幻讀]]**:同一交易內跑**同樣的範圍查詢**兩次,第二次多/少了資料列(別人中間 insert/delete)。
  > 區別:Non-repeatable = **同一筆被改值**;Phantom = **整列新增/刪除**改變結果集數量。

## 4. MVCC — 現代 DB 怎麼做到隔離又不卡

不靠「讓交易排隊等」(太慢)。[[mvcc|MVCC]] 對同一份資料**保留多個版本**,讀操作看「交易**開始時的快照**」而非最新值 → **讀不用等寫、寫不被讀擋**。PostgreSQL、MySQL InnoDB 都用它。

```
version 1: 1000  (T1 commit)
version 2: 500   (T2 commit)
T3 在 T2 commit「前」開始 → 看到 version 1 (1000)
T4 在 T2 commit「後」開始 → 看到 version 2 (500)
```

## 5. Lost Update 更新遺失 + 兩種鎖

兩個交易同時讀同一筆、各自算、各自寫回 → 後寫的**覆蓋**前者,前者更新消失:
```
餘額 1000
T1 讀 1000 → 算 800 → 寫 800 commit
T2 讀 1000 → 算 700 → 寫 700 commit  ← 覆蓋 T1!正確應 500,結果卻 700
```

- **[[optimistic-lock|樂觀鎖]]**:更新時帶版本號,版本被改過就拒絕、由 app 重試。**適合衝突少**。
  `UPDATE ... SET balance=800, version=2 WHERE id=1 AND version=1;`
- **[[pessimistic-lock|悲觀鎖]]**:讀時就鎖住(`SELECT ... FOR UPDATE`),別人得等。較安全但降並行,可能 [[deadlock|死鎖]]。

## 6. 分散式交易:跨多個 DB 怎麼辦

微服務各有各的 DB(訂單服務、庫存服務…),沒辦法用一個 `BEGIN/COMMIT` 包起來。

- **[[two-phase-commit|2PC 兩階段提交]]**:協調者先問大家「準備好了嗎(Prepare)」,全部 Yes 才發 Commit。**強一致**,但協調者在發 Commit 前崩潰 → 節點卡死,高可用系統難用。
- **[[saga|Saga Pattern]]**(微服務更常用):拆成一串**有補償機制**的本地交易,某步失敗就**反向補償**前面的步驟。放棄強一致,用 [[eventual-consistency|最終一致性]] 換可用性與容錯。

---

### 收尾小考(讀完在聊天回答)
1. ACID 四個字母各是什麼?
2. PostgreSQL 跟 MySQL InnoDB 的**預設隔離等級**分別是?
3. Dirty / Non-repeatable / Phantom Read 三者差在哪?
4. Lost Update 是什麼?樂觀鎖 vs 悲觀鎖怎麼解?
5. 2PC vs Saga 差別?微服務常用哪個?

```glossary
{
  "transaction": { "term": "Transaction 交易", "short": "把一組相關操作包成一個邏輯單元,保證整體執行(全成或全不做),由 [[atomicity|ACID]] 保證。" },
  "atomicity": { "term": "Atomicity 原子性", "short": "交易內所有操作全部成功或全部失敗;失敗就 [[rollback|回滾]]。靠 [[wal|WAL]] 實現。" },
  "consistency-acid": { "term": "Consistency 一致性 (ACID)", "short": "交易前後 constraints 都成立(商業邏輯正確);注意這 ≠ [[cap-theorem|CAP]] 的 C(複本同步)。" },
  "isolation": { "term": "Isolation 隔離性", "short": "並發交易互不干擾,各自像獨立執行。實務用不同隔離等級在安全與效能間取捨。" },
  "durability": { "term": "Durability 持久性", "short": "一旦 commit,即使系統立刻崩潰也不遺失;靠寫磁碟 + [[wal|WAL]] 重建。" },
  "rollback": { "term": "Rollback 回滾", "short": "交易失敗時撤銷其已做的所有變更,回到開始前的狀態。" },
  "wal": { "term": "WAL 預寫日誌", "short": "Write-Ahead Log:真正寫入前先把「打算做什麼」記到 append-only 日誌;崩潰後可重播續做或回滾,是原子性+持久性的基礎。" },
  "isolation-level": { "term": "Isolation Level 隔離等級", "short": "Read Uncommitted < Read Committed < Repeatable Read < Serializable;越高擋掉越多並發異常但越慢。" },
  "dirty-read": { "term": "Dirty Read 髒讀", "short": "讀到別人尚未 commit 的資料;對方一 rollback,你就讀到從未存在的值。Read Committed 起可防。" },
  "non-repeatable-read": { "term": "Non-repeatable Read 不可重複讀", "short": "同一交易內讀同一筆兩次值卻不同(別人中間改了並 commit)。Repeatable Read 起可防。" },
  "phantom-read": { "term": "Phantom Read 幻讀", "short": "同一交易內跑同樣範圍查詢兩次,列數不同(別人中間 insert/delete)。只有 Serializable 完全防。" },
  "mvcc": { "term": "MVCC 多版本並行控制", "short": "同一資料保留多版本,讀看交易開始時的快照而非最新值 → 讀不等寫、寫不被讀擋。PostgreSQL/InnoDB 都用。" },
  "lost-update": { "term": "Lost Update 更新遺失", "short": "兩交易同讀同算同寫,後寫者覆蓋前者更新而消失。用樂觀鎖或悲觀鎖解。" },
  "optimistic-lock": { "term": "Optimistic Lock 樂觀鎖", "short": "更新時帶版本號,版本被改過就拒絕、app 重試;適合衝突不常發生。" },
  "pessimistic-lock": { "term": "Pessimistic Lock 悲觀鎖", "short": "讀時就鎖(SELECT ... FOR UPDATE),別人得等;較安全但降並行、可能 [[deadlock|死鎖]]。" },
  "deadlock": { "term": "Deadlock 死鎖", "short": "兩交易互相等對方釋放鎖而卡死。解法:固定順序取鎖;DB 也有 deadlock detection 會 rollback 一方。" },
  "two-phase-commit": { "term": "2PC 兩階段提交", "short": "協調者先 Prepare 問大家是否就緒,全 Yes 才 Commit。強一致,但協調者崩潰會讓節點卡在 prepared 狀態。" },
  "saga": { "term": "Saga Pattern", "short": "把分散式交易拆成一串本地交易,每步配補償動作;某步失敗就反向補償。用最終一致性換可用性。" },
  "eventual-consistency": { "term": "Eventual Consistency 最終一致性", "short": "不要求每刻都一致,但保證沒有新更新後最終會收斂到一致。微服務/跨區常見取捨。" },
  "cap-theorem": { "term": "CAP Theorem", "short": "分散式系統在網路分區時,一致性(C)與可用性(A)只能二選一。注意此 C 與 ACID 的 C 不同。" }
}
```
