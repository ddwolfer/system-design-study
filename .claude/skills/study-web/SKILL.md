---
name: study-web
description: Use when interacting through the study-web browser cockpit (messages tagged channel source="study-web"), or when asked to put a lesson on the web / 上課到網頁 / rewrite a lesson into clickable web-notes. Covers the reply / show_notes tools and the clickable-term contract ([[id|surface]] markers + one glossary JSON block).
---

# study-web — 網頁陪讀座艙

你透過瀏覽器座艙 `study-web` 和使用者互動。**這是一個 `claude/channel`**:使用者的話以 `<channel source="study-web" chat_id="web" …>` 進到你的 session;**你想讓使用者看到的東西,只有透過工具送才會到瀏覽器**(終端機輸出到不了)。

## 兩個出口工具

- **`reply(text)`** — 送一則訊息到右側**聊天面板**。`text` 是 markdown(可含 mermaid ```mermaid 區塊、表格、程式碼)。深入解釋、回答問題都走這裡。
- **`show_notes(lesson, markdown)`** — 把你重寫好的**整課網頁筆記**推到左側**閱讀面板**。

收到的 `chat_id`(通常是 `web`)原樣帶回 `reply`。

## 何時做什麼

- 使用者說「**上 X 課 / 開 X 課 / 看 X**」→ 跑下方「重寫流程」→ `show_notes`。
- 使用者按術語小卡的「深入」→ 進來的訊息會像 `[在課程脈絡中深入解釋術語「…」] …`,用 `reply` 給**簡短、白話**的解釋(再有 jargon 就也包成 `[[…]]`,見下)。
- 一般提問 → `reply` 富文字回答。
- 一律遵循專案 `CLAUDE.md`:教練人設、anti-fabrication、信任規則、KG 流程、間隔複習。

## 重寫流程(PDF → 可點網頁筆記)

1. `Glob` 該章節資料夾,**複製精確課名**(含全形 `｜`、空格)當 `lesson`。
2. 先查快取:若 `notes/<NN_章節>/<課>/web-notes.md` 已存在 → 直接讀它 `show_notes`,**不重跑 Gemini/重寫**。
3. 否則:`gemini_digest_pdf(lesson[, file])`(多 PDF 課:先不帶 `file` 拿清單,再逐份)。把回傳逐字原文存成 `notes/<NN_章節>/<課>/digest.md`(沿用既有 header 慣例)。
4. 把 digest **重寫成乾淨雙語筆記**:標題層次、短段落、必要處用 ```mermaid 畫架構/流程。關鍵術語用**可點標記**(見下)。
5. 結尾附**一個** glossary JSON 區塊。把成品**存成 `notes/<NN_章節>/<課>/web-notes.md`**(下次直接讀檔),再 `show_notes(lesson, 該 markdown)`。

> 省 token 鐵則(同 `CLAUDE.md`):每課 PDF 只用 Gemini 讀一次;重寫結果寫檔快取;之後複習查 KG / 讀 `web-notes.md`,別重灌投影片。

## 可點術語約定 (term contract)

- 行內標記:`[[id|顯示文字]]`。`id` = kebab-case 穩定鍵;顯示文字可省 → `[[consistent-hashing]]`。
- 結尾**一個** ```glossary 區塊,JSON 以 id 為鍵:`{ "<id>": { "term": "...", "short": "...", "deeper": "..." } }`。
  - `term` = 卡片標題(英文術語 + 中文)。
  - `short` = **一兩句白話**小卡本文,**可再含 `[[…]]`** 巢狀術語。
  - `deeper`(可選)= 按「深入」時要問的問題;省略則前端自動用「請深入解釋:<term>」。
- `short` 來源:`search_memory(term)` → `get_knowledge(id)`,**優先取 `content`(雙語解釋)**,缺則取 `quote`(逐字)。查無 KG 節點就自己寫一句白話,別硬湊。
- 每個 `[[id]]` 都要在 glossary 有對應條目;沒有就別包成可點。

### 範例

````markdown
## 一致性雜湊

當叢集要加減節點時,用 [[consistent-hashing|一致性雜湊]] 只搬動少量 key,
比樸素的 [[mod-hashing|取模雜湊]] 幾乎全搬好得多。

```glossary
{
  "consistent-hashing": {
    "term": "Consistent Hashing 一致性雜湊",
    "short": "把節點與 key 都雜湊到同一個 [[hash-ring|hash 環]] 上,節點增減只影響環上相鄰的一段 key。",
    "deeper": "為何一致性雜湊在 cache 叢集擴縮容時優於取模?"
  },
  "mod-hashing": { "term": "Modulo Hashing 取模雜湊", "short": "用 key % N 決定節點;N 一變,幾乎所有 key 重新映射。" },
  "hash-ring": { "term": "Hash Ring 雜湊環", "short": "把雜湊值首尾相接成環,順時針找到的第一個節點即負責該 key。" }
}
```
````

## 注意

- **表格儲存格內不要用 `[[id|顯示]]` 的豎線形式** —— `|` 會被當成表格欄位分隔符,把儲存格拆掉、標記也壞掉。表格內改用 `[[id]]`(顯示文字自動取 glossary 的 `term`),或把豎線跳脫成 `\|`。
- mermaid 訊息/節點文字避免半形 `;`、`[`、`]`(會語法錯);節點標籤要含特殊字元時用 `"…"` 包起來。
- 投影片可能有 OCR 雜訊;重寫是**詮釋**,逐字原話以 `digest.md` 為準,要 quote 時去那裡撈。
