# 03_基本觀念 / 09. Database Transactions｜資料庫交易 — 投影片逐字原文

> 來源:`gemini_digest_pdf("03_基本觀念/09. Database Transactions｜資料庫交易")`,2026-06-02。**尚未入庫 KG**(明天討論後蒸餾)。

---

## 什麼是 Transaction?

真實系統很少「一步完成」。轉帳要從 A 扣、加到 B(兩步缺一不可);下訂單要建訂單、扣庫存、產生付款請求(全部成功或全部失敗)。**Transaction(交易)** 把一組相關操作包成一個邏輯單元,保證作為整體執行。沒有 transaction,中間任一步失敗 → 資料不一致(錢從 A 扣了卻沒加到 B;訂單建了卻沒扣庫存),這些是生產環境真實且難復現的 bug。

## ACID:四個保證

- **Atomicity(原子性)**:transaction 內所有操作要麼全部成功、要麼全部失敗,沒有中間狀態。轉帳若 A 扣成功但 B 加失敗 → **rollback(回滾)** 撤銷 A 的扣款,回到開始前狀態。實現:通常透過 **WAL(Write-Ahead Log)**——真正寫入前先把「打算做什麼」記到 log;中途崩潰時依 log 判斷續做或回滾。
- **Consistency(一致性)**:transaction 完成後,DB 從一個合法狀態轉到另一個合法狀態,所有 constraints 在前後都成立。例:設了「餘額不能為負」,想把 100 扣成 -50 的 transaction 會被拒。⚠️ **此 C 和 CAP 的 C 不同**:這裡指「商業邏輯層面的正確性」,不是複本間同步。
- **Isolation(隔離性)**:並發的多個 transactions 不應互相干擾,每個看起來像獨立執行。**這是四者中最複雜、最有取捨空間的**——完全隔離代價高,實務上有不同隔離等級。
- **Durability(持久性)**:一旦 commit 成功,資料永久保存,系統立刻崩潰也不遺失。靠寫入持久儲存(磁碟)+ WAL;重啟後可從 log 重建已 commit 但未刷到磁碟的資料。

## 隔離等級 (Isolation Levels)

「完全隔離」= 所有 transaction 按順序逐一執行,最安全但犧牲並行性。四種等級:

| 隔離等級 | Dirty Read | Non-repeatable Read | Phantom Read |
| --- | --- | --- | --- |
| Read Uncommitted | ✅ 可能發生 | ✅ 可能發生 | ✅ 可能發生 |
| Read Committed | ❌ 防止 | ✅ 可能發生 | ✅ 可能發生 |
| Repeatable Read | ❌ 防止 | ❌ 防止 | ✅ 可能發生 |
| Serializable | ❌ 防止 | ❌ 防止 | ❌ 防止 |

- **Read Uncommitted**:最低等級,可讀到別人未 commit 的資料。最危險(對方 rollback 你就讀到從未存在的資料),生產環境幾乎不用。
- **Read Committed**:只讀已 commit 的資料,避免 dirty read。**PostgreSQL 預設**。但同一 transaction 內兩次讀同一筆可能不同(別人中間 commit 了)。
- **Repeatable Read**:同一 transaction 內多次讀同一筆保證相同結果。**MySQL InnoDB 預設**。
- **Serializable**:最高等級,結果等同所有 transaction 按某順序逐一執行。最安全代價最高,通常只在金融、庫存扣除等「絕不能有並發異常」時用。

## 三種並發異常

- **Dirty Read(髒讀)**:A 讀到 B 尚未 commit 的資料,B 之後 rollback → A 讀到「從未存在」的資料。
  ```
  T1: 餘額 1000→500(未 commit)
  T2: 讀到 500 ← Dirty Read
  T1: 失敗 rollback,餘額回 1000
  T2: 錯誤相信餘額是 500
  ```
- **Non-repeatable Read(不可重複讀)**:同一 transaction 內兩次讀同一筆,結果不同(別人中間 commit 了更新)。
  ```
  T1: 讀餘額=1000
  T2: 餘額 1000→500,commit
  T1: 再讀=500 ← 和第一次不同
  ```
- **Phantom Read(幻讀)**:同一 transaction 內兩次同樣的範圍查詢,第二次多出新資料行(別人中間 insert)。
  ```
  T1: 查餘額>1000 的帳戶,找到 5 筆
  T2: 新增餘額 2000 的帳戶,commit
  T1: 再查,找到 6 筆 ← 多出「幽靈」資料
  ```
  > 差別:Non-repeatable 是**同一筆被改值**;Phantom 是**新增/刪除行**使範圍結果集改變。

## MVCC:資料庫如何實現隔離

不靠「讓每個 transaction 等前一個完成」(效能差)。現代 DB(PostgreSQL、MySQL InnoDB)普遍用 **MVCC(Multi-Version Concurrency Control,多版本並行控制)**:對同一份資料保存多個版本,讀操作看 transaction **開始時的快照**而非最新版本 → 讀不用等寫、寫不被讀擋。
```
version 1: 1000 (T1 commit)
version 2: 500  (T2 commit)
T3 在 T2 commit 前開始 → 看到 version 1 (1000)
T4 在 T2 commit 後開始 → 看到 version 2 (500)
```
舊版本在沒有 transaction 需要後由垃圾回收清理。

## Lost Update(更新遺失)

兩個 transaction 同時讀同一筆、各自算新值、都寫回 → 後寫的覆蓋前一個,前者更新消失。
```
餘額 1000
T1: 讀 1000,算 1000-200=800,準備寫
T2: 讀 1000,算 1000-300=700,準備寫
T1: 寫 800 commit
T2: 寫 700 commit ← 覆蓋 T1,扣款消失!
正確應為 500,實際卻是 700
```
兩種解法:
- **樂觀鎖 (Optimistic Locking)**:更新時帶版本號,版本不符就拒絕。適合衝突不常發生。
  ```sql
  UPDATE accounts SET balance=800, version=2 WHERE id=1 AND version=1; -- version 被改過則影響 0 筆
  ```
- **悲觀鎖 (Pessimistic Locking)**:讀取時就鎖住,阻止其他 transaction 讀/改,直到完成。較安全但降低並行性,可能引發 **Deadlock(死鎖)**;DB 通常有 deadlock detection 會 rollback 其中一個。
  ```sql
  SELECT balance FROM accounts WHERE id=1 FOR UPDATE; -- 排他鎖
  UPDATE accounts SET balance=800 WHERE id=1;
  COMMIT;
  ```

## 分散式 Transaction 的挑戰

單一 DB 內的 transaction 直觀;但拆成微服務、資料分散多 DB 就複雜。例:訂單服務一個 DB、庫存服務另一個 DB,無法用一個 BEGIN/COMMIT 包起來。

- **Two-Phase Commit(2PC,兩階段提交)**:由協調者 (coordinator) 確保所有節點同意 commit。
  - **第一階段 (Prepare)**:協調者問「準備好了嗎?」每節點把資料寫好但先不 commit,回 Yes/No。
  - **第二階段 (Commit/Abort)**:全 Yes → 發 Commit;任一 No → 發 Abort 全部 rollback。
  - 問題:協調者在第二階段發 Commit 前崩潰 → 部分節點卡在「已 prepare 未 commit」,需人工干預。高可用系統難用。
- **Saga Pattern**(現代微服務更常用):把分散式 transaction 拆成一系列**有補償機制**的本地 transaction。
  ```
  1. 訂單服務:建訂單(本地 commit)
  2. 庫存服務:扣庫存(本地 commit)
  3. 付款服務:扣款(本地 commit)
  若步驟3失敗:補償步驟2(庫存加回)、補償步驟1(訂單標記取消)
  ```
  放棄強一致性,用**最終一致性**換可用性與容錯,是絕大多數微服務的現實選擇。

## 面試中怎麼談 Transaction

- **何時主動提**:多步驟原子操作(建訂單+扣庫存包進同一 transaction)、防並發更新錯誤(搶購最後庫存用 `SELECT FOR UPDATE` 防超賣)、高一致性場景(金融系統用 Serializable)。
- **說明隔離等級選擇**:別只說「我用 transaction」。例:讀用戶資料 Read Committed 就夠;庫存扣除需 Serializable(防 phantom read 導致超賣),必要時用 `SELECT FOR UPDATE` 鎖特定行而非提升整個隔離等級。
- **微服務坦誠取捨**:跨服務強一致難保證,2PC 高可用場景不適合 → 改用 Saga,各服務本地 transaction、失敗補償、接受最終一致性。
- **主動說死鎖風險**:用 FOR UPDATE 時若多 transaction 以不同順序鎖資源可能 deadlock;解法是所有 transaction 按一致順序取鎖(例:永遠先鎖較小的 account_id),DB 的 deadlock detection 也會自動 rollback 一方,application 層需處理重試。

## 自我測驗 (Self-test)

**Q1:** ACID 各代表什麼?
> Atomicity:全成功或全回滾;Consistency:DB 從合法態轉合法態、constraints 成立;Isolation:並發交易互不干擾;Durability:已 commit 的資料崩潰後仍存在。

**Q2:** PostgreSQL / MySQL InnoDB 預設隔離等級?
> PostgreSQL = **Read Committed**;MySQL InnoDB = **Repeatable Read**。

**Q3:**(配對)(1)Dirty Read (2)Non-repeatable Read (3)Phantom Read / (a)範圍查詢回傳不同數量 rows (b)讀到未 commit 資料 (c)同一 row 讀到不同值
> 1→(b)、2→(c)、3→(a)。

**Q4:** Lost Update 是什麼?樂觀鎖/悲觀鎖如何解?
> 兩交易同讀同算同寫,後者覆蓋前者更新而遺失。樂觀鎖:版本號,不符就拒絕、由 app 重試(衝突少時用);悲觀鎖:`SELECT...FOR UPDATE` 鎖行、其他等待(較安全但降並行、可能 deadlock)。

**Q5:** 2PC vs Saga 主要差異?微服務常用哪個?
> 2PC:協調者確保所有節點同意 commit,強一致但協調者崩潰會卡死;Saga:拆成本地交易+補償,用最終一致性換可用性與容錯。微服務更常用 **Saga**。

**Q6:** ACID 的 C 和 CAP 的 C 差異?
> ACID 的 C = 商業邏輯正確性(constraints,如餘額不能為負);CAP 的 C = 所有節點同時看到相同資料(複本同步)。完全不同概念。
