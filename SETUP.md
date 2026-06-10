# SETUP — 在新機器(例如 MacBook)跑起來

這套陪讀系統 = 一個 Claude Code 專案 + 3 個本地 MCP server(`kg` 知識圖譜、`mcp-gemini-video` Gemini 眼睛、`study-web` 網頁座艙)。`git clone` 之後**還需要幾個手動步驟**才能跑,因為有些東西刻意不進 git(原生套件、密鑰、課程素材)。

> 懶人驗證:做完下面步驟,在專案根目錄執行 **`node scripts/check-setup.mjs`**,它會逐項 ✅/⚠️/❌ 告訴你還缺什麼。或直接 `claude` 進來問「**驗證我的設定**」。

---

## 前置需求
- **Node.js 18+**(建議 20 LTS) — 三個 server 都是 Node。
- **Claude Code CLI** 已安裝、且用 claude.ai 或 Console API key 登入(channels 不支援 Bedrock/Vertex)。
- **git**。

## 步驟

### 1. Clone
```bash
git clone <repo-url> system-design-study
cd system-design-study
```

### 2. 安裝相依套件(★ 必須在這台機器上跑 ★)
`node_modules/` 沒進 git,而且 `kg` 用到**原生編譯**套件(`better-sqlite3`、`sqlite-vec`)——**不能**把 Windows 的 `node_modules` 複製過來,架構不符會載入失敗。請各自安裝:
```bash
cd kg && npm install && cd ..
cd mcp-gemini-video && npm install && cd ..
cd study-web && npm install && cd ..
```
> `kg` 首次還會用 `@huggingface/transformers` 下載 embedding 模型權重(要等一下、需網路)。

### 3. 設定 Gemini API key
key 不進 git。擇一:
```bash
cp mcp-gemini-video/.env.example mcp-gemini-video/.env   # 然後填入你的 key
# 或
export GEMINI_API_KEY="你的key"                          # 寫進 ~/.zshrc 較持久
```

### 4. 放課程素材(用 Google 雲端手動下載)
`現代系統設計_課程講義/` 被 gitignore(版權 + 影片 >100MB),**clone 不會有**。把整個資料夾放到專案根目錄:
```
system-design-study/現代系統設計_課程講義/02_講義導讀/...
```
> 只有「開新課 / 重讀 PDF / 看影片」需要它。純複習(查知識圖譜)不需要。

### 5. 信任 MCP server(避免網頁座艙卡在權限詢問)
`.claude/settings.local.json` 是 gitignore 的,clone 沒有。複製範本:
```bash
cp .claude/settings.local.json.example .claude/settings.local.json
```
(或不複製,等 Claude Code 首次啟動時逐一按「同意」也行。)

### 6. ★ 修正 hook 的絕對路徑(macOS 必做一次)★
`.claude/settings.json` 裡 4 個 hook(開場複習、auto-recall、search-enforcer、自動入庫)目前寫死了 Windows 絕對路徑 `D:/AI/system-design-study/...`,在 Mac 上會失效。**把這段前綴刪成相對路徑**:

把檔案中所有 `D:/AI/system-design-study/` 整段刪掉(置換成空字串),例如:
```
"command": "node D:/AI/system-design-study/kg/hooks/session-start.js \"D:/AI/system-design-study/kg/system-design.db\""
```
改成:
```
"command": "node kg/hooks/session-start.js \"kg/system-design.db\""
```
4 個 hook(`session-start` / `post-compact` / `auto-recall` / `search-enforcer`)都要改。
> 這個檔案是 Claude Code 的設定,Claude 預設不會自己改(安全保護)。你可手動改,或在 session 裡**明確授權** Claude 改。`scripts/check-setup.mjs` 會偵測有沒有漏掉的絕對路徑。

### 7. 啟動
- **macOS / Linux**:第一次先 `chmod +x study-coach.command`,之後雙擊或 `./study-coach.command`。
- **Windows**:`study-coach.cmd`。
- 啟動後開瀏覽器 **http://127.0.0.1:7654**。
- 若只想純文字複習、不用網頁,直接 `claude` 即可(KG 與 Gemini 仍可用;只是沒有網頁座艙)。

---

## 哪些東西「會 / 不會」跟著 git 走

| 項目 | 進 git? | 說明 |
|---|---|---|
| 程式碼、skills、CLAUDE.md、plans | ✅ | 正常追蹤 |
| **`kg/system-design.db`(你學到的知識)** | ✅ | 已 commit → 複習/考問在新機器立刻可用 |
| `node_modules/` | ❌ | 各機器 `npm install`(原生套件要本機編譯) |
| `GEMINI_API_KEY` / `.env` | ❌ | 密鑰不進 git,手動設定 |
| `現代系統設計_課程講義/`(課程素材) | ❌ | 版權 + 影片過大,雲端手動搬 |
| `notes/`(digest、web-notes 快取) | ❌ | 版權內容;新機器重新產生即可 |
| `.claude/settings.local.json` | ❌ | 用 `.example` 複製 |

---

## 驗證
```bash
node scripts/check-setup.mjs
```
全綠就能上課。有 ❌ 照它的提示修,再跑一次。
