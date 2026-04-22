# MYDEVSOP 問題解決方案路線圖

> 核心原則：「不會寫程式也能開發」是正確定位。就像 Gamma.app 讓人不懂 CSS 也能做簡報——
> 工具按想法生成，能不能用全看 IDEA 品質，不看使用者的技術能力。
> 所有解決方案都圍繞這個原則：技術複雜度由 AI 吸收，不暴露給使用者。

---

## 問題與解決方案全表

### 🔴 IDEA 管理（最核心）

| # | 問題 | 解決方案 | AI 技術 | 工程量 |
|---|------|---------|---------|--------|
| 1 | 非程式人員不知道怎麼寫 BRD | `/devsop-idea` 改為**多輪對話式** IDEA 訪談，Claude 從對話中提取 BRD，使用者永遠不看 Markdown | Extended Thinking + Structured Output | M（2週） |
| 2 | `/devsop-idea` 不是強制入口點 | `autodev` 開頭加 guard：沒有 `.devsop-state.json` → 直接導向 `/devsop-idea` | Bash guard | S（2天） |
| 3 | 沒有 IDEA 可行性驗證（花 45 分鐘才發現 IDEA 做不出來） | 加「可行性關卡」：LLM-as-judge 評估 4 維度，發現問題用白話文問使用者澄清，通過才跑 pipeline | Subagent + structured scoring | M（2週） |

---

### 🔴 Onboarding 門檻

| # | 問題 | 解決方案 | AI 技術 | 工程量 |
|---|------|---------|---------|--------|
| 4 | 13 個門檻步驟才能第一次成功 | 加「Quick Start 模式」：使用者說 1-3 句話 → AI 填所有 BRD 空白 → 15 分鐘跑完 | Claude defaults + skeleton BRD | M（2週） |
| 5 | 沒有 5 分鐘的 Quick Win 體驗 | 建 `/devsop-demo`：對著內建 URL shortener BRD 只跑 STEP 03，3 分鐘看到 PRD 生成結果 | Pre-built fixture | S（1週） |
| 6 | 沒有範例讓新用戶知道「好的 IDEA 長什麼樣」 | `examples/` 目錄放 3 個範例 BRD（簡單/中/複雜）+ IDEA 輸入後顯示品質評分條 | LLM scoring rubric | S（3天） |

---

### 🔴 Pipeline 自動化

| # | 問題 | 解決方案 | AI 技術 | 工程量 |
|---|------|---------|---------|--------|
| 7 | `read -t 5` 在 Claude Code Bash 的 TTY 行為未驗證（全自動模式核心風險） | **Full Auto 模式**：所有決策從 `.devsop-state.json` 讀取（IDEA 階段預設好），完全移除 `_ask()`；**Interactive 模式**：改用 `AskUserQuestion` | 純 Bash 架構調整 | S（3天） |
| 8 | 非程式人員不知道 20 個步驟在做什麼 | 加白話文進度條取代技術 Step Summary：「正在寫程式碼...（預計剩餘 18 分鐘）」，技術細節折疊給開發者看 | 靜態文字映射 | S（3天） |
| 9 | STEP 02 語言選型暴露 GIL、pydantic 等技術術語 | 改為使用場景選單：「Web API / 資料處理 / 即時系統 / 全端」，Claude 在背後決定技術棧，使用者不看框架名稱 | Claude reasoning | S（2天） |

---

### 🟡 驗證與確認（讓非程式人員能判斷輸出品質）

| # | 問題 | 解決方案 | AI 技術 | 工程量 |
|---|------|---------|---------|--------|
| 10 | 非程式人員看不懂 ARCH/API/Schema Review 輸出 | 每個 Review STEP 加「AI 翻譯官」子 Agent：把技術 Review 翻成白話文摘要（「你的系統最多支援 10,000 人/天」） | LLM-as-translator subagent | M（2週） |
| 11 | TDD GREEN 失敗讓非程式人員以為出錯了 | 改為白話文訊息：「我發現一個設計需要調整，正在自動修復，大約多 5 分鐘」，技術細節可展開 | LLM-as-explainer | S（3天） |
| 12 | k8s HPA 配置驗證需要 DevOps 知識 | 完全移除人工確認：AI 從 BRD 的「預期使用規模」自動推算 HPA，只在費用超過門檻時問使用者 | Claude inference from BRD | S（3天） |

---

### 🟡 留存與可持續性

| # | 問題 | 解決方案 | AI 技術 | 工程量 |
|---|------|---------|---------|--------|
| 13 | 跑完一次後沒有理由回來 | 建 `/devsop-project-status`：健康儀表板，白話文顯示「文件是否同步、測試是否通過、有無安全問題」 | Git log 解析 + LLM 摘要 | M（2週） |
| 14 | 沒有專案健康儀表板 | GitHub Pages 首頁加入對齊率與健康指標區塊（讓任何人用 URL 就能看到狀態） | 擴展 STEP 19 模板 | S（2天） |
| 15 | 跑完之後不知道怎麼「改」 | 建 `/devsop-change`：「我想改什麼」→ AI 分析影響哪些 STEP → 只重跑受影響的 STEP → 建 PR | Claude reasoning for STEP mapping | L（4週） |

---

### 🟡 團隊協作

| # | 問題 | 解決方案 | AI 技術 | 工程量 |
|---|------|---------|---------|--------|
| 16 | 多人使用時 `.devsop-state.json` 會衝突 | State file 依 git username + branch 命名空間化：`.devsop-state-{user}-{branch}.json` + flock 寫入鎖 | Bash file-locking | S（2天） |
| 17 | AI 輸出直接 commit 到 main，沒有人工確認關卡 | STEP 20 改為建 feature branch + 自動建 PR（TOTAL SUMMARY 作為 PR body），使用者看到：「你的專案已準備好，點這個連結確認後 merge」 | gh CLI | S（3天） |

---

### 🟢 技術韌性（使用者感受不到，但影響可靠性）

| # | 問題 | 解決方案 | AI 技術 | 工程量 |
|---|------|---------|---------|--------|
| 18 | SKILL.md prose 邏輯每次靠 Claude 詮釋，無確定性 | 提取 `scripts/devsop-lib.sh`：`_write_state()`、`_review_loop()`、`_ask()` 改為真實 Bash 函式可 bats 測試 | 純 Bash | M（2週） |
| 19 | State file 寫入非原子性（crash → 截斷 JSON） | Python 原子寫入（write-to-tmp → mv）+ backup + `step_log[]` array 記錄每 STEP 完成狀態 | Python file I/O | S（2天） |
| 20 | 32 個 SKILL.md 零測試覆蓋 | 三層測試：(a) front-matter schema 驗證 (b) devsop-lib 單元測試 (c) fixture BRD pipeline smoke test | bats + Claude stub | S/M/L |
| 21 | Agent 失敗無處理（45 分鐘 pipeline 中途崩潰） | 加 `_spawn_agent_with_retry()`（最多 2 次重試）+ 每 STEP 輸出 `STEP_COMPLETE`/`STEP_FAILED` 信號 + 冪等性檢查 | Bash | S（3天） |
| 22 | 14+ 個 SKILL.md copy-paste 共用邏輯 | 全部提取到 `devsop-lib.sh`，建 migration tracker 追蹤進度 | Bash | M（3週） |

---

## 「非程式設計師也能使用」實現路徑

```
現況（有門檻）：
  使用者 → 寫 BRD.md → /devsop-autodev → 看技術 Step Summary → 評估 ARCH/API/Schema → 判斷 TDD 失敗原因

目標（Gamma.app 模式）：
  使用者說 IDEA → 對話式訪談補充細節 → 可行性確認 → 全自動跑 pipeline
              → 白話文進度條 → 白話文審查摘要 → PR 連結 → merge 就上線
```

**實現的關鍵槓桿點：**
1. IDEA 輸入：對話式取代 Markdown（問題 #1）
2. 自動化：Full Auto 模式移除所有技術問題（問題 #7, #9）
3. 輸出翻譯：每個技術輸出都有白話文版本（問題 #8, #10, #11, #12）
4. 確認機制：PR review 給使用者一個清楚的「確認點」（問題 #17）

---

## 分階段實施建議

### Phase 1：讓非程式設計師能真正用（4 週）
- #7 Fix read -t 5 → Full Auto 模式
- #17 Feature branch + auto-PR
- #9 STEP 02 改用場景選單
- #2 強制 /devsop-idea 入口
- #1 對話式 IDEA 捕捉
- #4 Quick Start 模式
- #8 白話文進度條

### Phase 2：讓體驗流暢（4 週）
- #5 /devsop-demo + 範例 BRD
- #3 可行性關卡
- #10 AI 翻譯官（ARCH/API/Schema）
- #19 原子寫入
- #21 Agent retry + 冪等性

### Phase 3：讓工具有黏性（8 週）
- #13 /devsop-project-status
- #15 /devsop-change
- #18 devsop-lib.sh 提取
- #22 共用邏輯統一
- #20 測試覆蓋
