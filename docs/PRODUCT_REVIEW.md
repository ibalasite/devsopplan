# MYDEVSOP 產品 & 生產力審查報告

> 審查日期：2026-04-19
> 版本：REQUIREMENTS.md v1.3
> 審查角度：產品定位 / 生產力 ROI / 技術架構

---

## 執行摘要（Executive Summary）

MYDEVSOP 是一個技術上有真實深度的工具——Agent-per-STEP 架構解決了 context overflow 問題，20-STEP 閉環覆蓋了從 BRD 到 GitHub Pages 的完整開發週期，對熟練的獨立開發者每個專案能節省 **40–70 小時**的重複性工作。

核心問題不在技術，在產品決策：目標用戶定義矛盾（「不會寫程式的人」vs. 實際需要 CLI + git + k8s 知識）、沒有 Quick Win 路徑、沒有任何 SKILL.md 的自動化測試覆蓋、且所有 33 個技能文件都靠 copy-paste 共用邏輯——維護成本隨規模線性增長。

**最優先改善方向（3 件事）：**
1. 移除「不會寫程式」的錯誤定位，改為誠實 persona
2. 加入 feature branch + auto-PR（讓工具從個人工具變成團隊工具）
3. 提取 `scripts/devsop-lib.sh`（讓 33 個 SKILL.md 可測試、可維護）

---

## 綜合評分

| 維度 | 評分 | 關鍵問題 |
|------|------|---------|
| P1 價值主張清晰度 | 2/5 | 無單句核心承諾，README 與 REQUIREMENTS 定位矛盾 |
| P2 目標用戶 Fit | 2/5 | 「不會寫程式」宣稱與 CLI+k8s 前提衝突 |
| P3 差異化競爭力 | 3/5 | SOP 閉環有真實護城河，但未對外清楚表達 |
| P4 功能完整性 | 3/5 | Happy path 完整；hotfix、monorepo、既有專案缺失 |
| P5 採用門檻 | 2/5 | 13 個門檻步驟，無範例 BRD，無 5 分鐘體驗路徑 |
| P6 可持續性 | 2/5 | 用完一次後沒有回頭理由，無留存機制 |
| G1 時間節省 ROI | 3.5/5 | 每專案省 40-70 hr，$100/hr 工程師第一個專案就回本 |
| G2 流水線效率 | 3/5 | STEP 08/09/10 和 16/17/18 可並行卻串行；每次跑 $6-15 token 費 |
| G3 團隊協作 | 2/5 | 個人工具，state.json 無鎖，多人使用會衝突 |
| G4 維護成本 | 2.5/5 | 每月 4-8 hr，33 個文件都有重複邏輯 |
| G5 失敗復原 | 3/5 | STUBBORN 機制好；resume 路徑未充分實作 |
| G6 規模化路徑 | 2/5 | 1→10 人需大量重工，無 SaaS 路徑 |
| T1 Markdown-as-Code | 2/5 | 無可執行語意，複雜分支在 prose 中不可靠 |
| T2 狀態管理 | 2/5 | 寫入非原子性，無鎖，缺少 step_log、schema_version |
| T3 測試覆蓋 | 2/5 | Shell scripts 有好測試；32 個 SKILL.md 零覆蓋 |
| T4 錯誤處理 | 2/5 | Agent 崩潰無處理；STEP 非冪等性 |
| T5 技術債 | 2/5 | 14+ 個文件 copy-paste 版本檢查；tool 介面高耦合 |
| **整體** | **2.4/5** | 技術有深度，產品決策需要補課 |

---

## 核心發現

### 優勢（Keep）

- **Agent-per-STEP 架構**：每個 STEP 有獨立 context window，真正解決了長流水線的 context overflow 問題——這是比大多數競品更深的工程洞見
- **文件對齊閉環**：PRD → EDD → Code → Test 四層對齊檢查（REQ-2.8）是業界首創的差異化功能，無競品等效
- **基礎設施完整性**：一個指令能生成 k8s 配置、GitHub Actions CI/CD、Secrets 腳本、HTML 文件網站——這個覆蓋廣度在個人開發者工具中獨一無二
- **Shell 腳本品質**：`install.sh`、`check-update.sh`、`update.sh` 有 32 個 bats 測試，hardened 且正確

### 問題（Fix）

- **定位矛盾**：REQUIREMENTS §1「不會寫程式的人也能全程使用」vs. 實際需要 Claude Code CLI + git + GitHub + k8s 知識——這個矛盾阻止了誠實的產品設計
- **Markdown 作為執行單元的上限**：`_review_loop()`、`_write_state()`、`_ask()` 是 prose 中的偽代碼，每次執行都靠 Claude 重新詮釋，無法被單元測試，邏輯正確性無保證
- **單人工具的天花板**：`.devsop-state.json` 無鎖、直接 commit 到 main、無 PR review gate——多人使用必然衝突
- **`read -t 5` 未驗證**：整個「全自動模式」的核心假設（5 秒 timeout）在 Claude Code Bash 的非 TTY 環境中行為未定義（REQUIREMENTS Q1 自己承認），這是 v2.0 最大的技術風險

### 缺口（Build）

- **既有專案 adoption path**：沒有「我有一個現有專案，我想用 MYDEVSOP 評估它」的入口
- **Hotfix / 生產事故流程**：整個 pipeline 假設 greenfield，生產問題無法處理
- **Resume 可靠性**：中途失敗後如何精確從某個 STEP 繼續，需要 `--resume-from STEP-NN` 機制
- **成本可視性**：每次 pipeline 跑約 $6-15 token 費用，使用者完全不知道
- **樣例 BRD**：新用戶不知道什麼樣的 BRD 能通過 STEP 01，是最高的 onboarding 失敗點

---

## 分維度詳細 Findings

### 產品定位（Agent A — Product Manager）

**[P1] 價值主張清晰度 — 2/5**
- 優點：REQUIREMENTS 的核心承諾「BRD → GitHub Pages，一個指令」是具體且可驗證的
- 問題：README 定位是「文件 Review 工具」；REQUIREMENTS 定位是「全自動程式生成流水線」。這是兩個不同的產品。無任何地方有單句清楚的 value proposition
- 建議：第一行 README 改成：「寫一份 BRD，AI 生成後續所有東西——程式碼、測試、k8s、文件網站。」非程式設計師的宣稱等真正解決前置門檻問題後再加回來

**[P2] 目標用戶 Fit — 2/5**
- 優點：對有 2-5 年經驗的後端獨立開發者，工具匹配度很高
- 問題：「不會寫程式」的人在 STEP 02（GIL 限制說明）就會迷失；STEP 13（TDD GREEN 失敗 → 修 EDD）需要理解測試與設計文件的因果關係
- 建議：定義一個誠實的 primary persona（「有 git 和 API 基礎的獨立開發者」），把所有設計決策對準這個人

**[P3] 差異化競爭力 — 3/5**
- 優點：「每行程式碼都能追溯到 PRD AC，每個 AC 都能追溯到通過的測試」——這是 Copilot / Cursor / Devin 都沒有的
- 問題：這個差異化從未在任何對外文字中說清楚
- 建議：用這句話當 tagline：「MYDEVSOP enforces that every line of code traces back to a PRD acceptance criterion, and every PRD criterion traces forward to a passing test.」

**[P4] 功能完整性 — 3/5**
- 優點：Happy path（greenfield, 單一後端 API, macOS, PostgreSQL + k8s）覆蓋完整
- 缺口（前 5 大）：
  1. Hotfix / 生產事故流程（無法處理「現在 production 壞了」）
  2. 既有專案 adoption（沒有改造現有專案的入口）
  3. Monorepo 支援（假設單一 repo 單一 service）
  4. Multi-environment（只有 local k8s，無 staging/production 升版流程）
  5. Tech debt 追蹤（align-check 找出問題但不追蹤趨勢）

**[P5] 採用門檻 — 2/5**
- 從零到第一次成功 `/devsop-autodev` 的門檻步驟：13 個
- 最高摩擦點：Step 6（撰寫好的 BRD）—— 沒有範例，新用戶不知道「夠好」的標準
- 沒有 Quick Win：最小有意義的輸出（生成 PRD）需要完成 STEP 03，被 STEP 01/02 的 review loop 擋在前面
- 建議：在 repo 放一個完整範例（URL shortener），包括樣例 BRD + 所有生成的文件 + GitHub Pages URL

**[P6] 可持續性 — 2/5**
- 問題：一次跑完後沒有回頭理由；無跨專案狀態；無使用量數據
- 建議：加一個 `devsop-project-status` skill，讀取 `.devsop-state.json` + ALIGN_REPORT 生成「健康儀表板」——這給開發者理由在已建好的專案上重新打開 MYDEVSOP

---

### 生產力 ROI（Agent B — Engineering Manager）

**[G1] 時間節省評估 — 3.5/5**
- 估算節省時間（每個完整 pipeline run）：
  - PRD 生成：省 ~6 hr
  - EDD + 4 個 Mermaid 圖：省 ~9 hr
  - Schema doc：省 ~5 hr
  - k8s + CI/CD + Secrets：省 ~9 hr
  - HTML 文件網站：省 ~12 hr
  - **總計：40–70 hr/專案**
- ROI：工程師時薪 $100/hr → 第一個專案即回本
- 問題：STEP 13（TDD loop）是最被高估的；Review loop 總 overhead 估計 4-12 hr
- 建議：實作「輕量模式」跳過低風險 STEP 的 review loop（STEP 03, 11, 19），保留關鍵 STEP 01/06/14/15 的品質關卡

**[G2] 流水線效率 — 3/5**
- STEP 08/09/10（ARCH/API/Schema Review）可並行但目前串行——並行可省 ~60% 時間
- STEP 16/17/18（k8s/CI/CD/Secrets）同樣可並行
- Token 成本估算：每次完整 pipeline 約 1.5–2M tokens ≈ **$6–15/run**（Claude Sonnet 定價）
- MODE-B 無限循環：問題文件可能觸發 20+ 輪，3-5x 成本
- 最常被跳過的 STEP：STEP 13（TDD GREEN 失敗率高）、STEP 16-18（需要 local k8s）、STEP 20（依賴 GitHub Pages 設定）

**[G3] 團隊協作 — 2/5**
- 本質上是個人工具：REQUIREMENTS §18 Q4 明確說「目前個人專案」
- 多人使用會：競爭寫 `.devsop-state.json`（無鎖）、commit 到 main（無 PR gate）、5 秒自動選擇在團隊中危險
- 建議：加 feature branch + auto-PR 在 STEP 20——這一個改變讓工具從個人工具變成團隊安全工具

**[G4] 維護成本 — 2.5/5**
- 個人使用者：每月約 4-8 hr 維護
- 團隊共用：每月約 12-20 hr（支援問題 + 客製化 + 修 Claude Code 更新造成的 regression）
- 最大風險：33 個 SKILL.md 各自 copy-paste 共用邏輯，一個邏輯改動需要修 8-14 個文件
- `read -t 5` 的 TTY 假設是最高優先級的技術風險（Q1 自己承認未驗證）

**[G5] 失敗復原 — 3/5**
- 好：STUBBORN 機制防止無限循環；EDD backtrack loop 有 circuit breaker
- 問題：resume 機制語意不清——部分執行（Agent 崩潰、無 git commit）後 state 停在上個 STEP，resume 會重跑一遍，可能生成衝突 commit
- 建議：加 `--resume-from STEP-NN` 明確 escape hatch

**[G6] 規模化路徑 — 2/5**
- 1→10 人需要：branch 策略、state.json 命名空間、人工 review gate
- SaaS 化受限：MYDEVSOP 完全依賴 Claude Code，移植到 SaaS 需要重寫 Agent 編排層
- 最高 ROI 擴展方向：pipeline run 儀表板（從 git log 重建 run 歷史）+ 成本追蹤輸出

---

### 技術架構（Agent C — Staff Engineer）

**[T1] Markdown-as-Code — 2/5**
- 優點：SKILL.md 是 Claude Code 的原生格式，零膠水代碼
- 問題：
  - `_review_loop()` 等「函式」是 prose 偽代碼，每次執行靠 Claude 重新詮釋，無確定性
  - `devsop-autodev/SKILL.md` 已達 1,397 行，違反自己的 800 行限制
  - Step Summary 格式無 schema validation
- 建議：把三個核心函式提取到 `scripts/devsop-lib.sh`（可 `source` + bats 測試）

**[T2] 狀態管理 — 2/5**
- 問題：
  - `_write_state` 用 heredoc 直接覆寫，mid-crash 會產生截斷的 JSON
  - 無文件鎖（flock），concurrent 執行會 race condition
  - 缺失欄位：`step_start_time`、`step_log[]`、`schema_version`、`git_repo_root`
- 建議：改為 write-to-tmp then `mv -f`（原子性）；加 `step_log` array

**[T3] 測試覆蓋 — 2/5**
- 好：shell scripts 有 32 個 bats 測試，覆蓋安全性和邊界條件
- 問題：32 個 SKILL.md 零測試覆蓋；無 end-to-end pipeline 測試；修改 SKILL.md 後 regression 完全不可見
- 建議：
  1. `tests/unit/test_devsop_lib.bats`：測試 `_write_state` JSON 合法性、STEP_SEQUENCE 完整性
  2. `tests/integration/test_pipeline_smoke.bats`：用 fixture BRD + mock Claude stub 跑完整 pipeline

**[T4] 錯誤處理 — 2/5**
- 問題：Agent spawn 失敗無處理；STEP 非冪等（重跑會覆蓋人工編輯）；`_write_state` 失敗無錯誤檢查
- 建議：`_spawn_agent_with_retry()`（max 2 retry）；`_write_state_safe()`（tmp + mv + 驗證）

**[T5] 技術債 — 2/5**
- 問題：Step -1 版本檢查 block 被 copy-paste 進至少 14 個 SKILL.md；tool 介面名稱 hard-code，Claude Code 任何 tool rename 需要同時改 33 個文件
- 建議（優先序）：
  1. 提取 `scripts/devsop-lib.sh`（最高影響）
  2. `tests/unit/test_skill_schema.bats` 驗證 SKILL.md front-matter
  3. 把 devsop-autodev 拆分成 coordinator stub + per-step sub-skills

---

## 優先行動建議

### P0（立刻做，1 週內）

1. **驗證 `read -t 5` 是否在 Claude Code Bash tool 中有效**
   這是 v2.0 全自動模式的核心假設，REQUIREMENTS Q1 自己標注未驗證。如果不能用，整個 v2.0 架構需要換方案（改用 Python `select()` 或 `AskUserQuestion`）。
   測試方法：在 Claude Code 裡執行 `! bash -c "read -t 5 x; echo result: $x"` 等 5 秒看是否自動繼續。

2. **移除 README 和 REQUIREMENTS 中「不會寫程式的人也能全程使用」**
   替換為誠實的 persona：「熟悉 git 和 CLI 的獨立開發者或小團隊 tech lead」。

3. **加一個完整範例 BRD 到 repo**
   路徑：`examples/url-shortener/BRD.md`，展示足以通過 STEP 01 的 BRD 品質標準。

### P1（本月內，1 個月內）

4. **加 feature branch + auto-PR（STEP 20）**
   改為在 `devsop/autodev-YYYYMMDD-HHMMSS` branch 上 commit，STEP 20 自動建 PR，PR body 為 TOTAL SUMMARY。讓工具變成團隊安全的工具。

5. **提取 `scripts/devsop-lib.sh`**
   把 `_ask()`、`_write_state()`、`_review_loop()` 從 SKILL.md prose 移到可 source 的 bash 函式庫。加入 `tests/unit/test_devsop_lib.bats`。

6. **並行化 STEP 08/09/10 和 16/17/18**
   在 autodev SKILL.md 中，這兩個階段改為同時 spawn 三個 Agent。預計節省 30-45% pipeline 執行時間。

### P2（本季，3 個月內）

7. **加 `--resume-from STEP-NN` 明確復原機制**
   讓用戶在 STEP 失敗後手動修復文件，然後從指定 STEP 繼續，而非重跑整個 pipeline。

8. **加 `devsop-project-status` skill**
   讀取 `.devsop-state.json` + `docs/ALIGN_REPORT.md` + git log，生成現有專案的健康儀表板，給用戶回訪理由。

9. **加 token 成本估算到 TOTAL SUMMARY**
   每次 pipeline 結束時顯示：「估計消耗：1.8M tokens，約 $12」，讓使用者知道成本。

10. **`_write_state` 改為原子寫入（write-to-tmp + mv）**
    防止 mid-crash 產生截斷的 `.devsop-state.json`。

---

## 競爭矩陣

| 功能 | MYDEVSOP | GitHub Copilot | Cursor | Devin |
|------|----------|---------------|--------|-------|
| 全流程文件生成（BRD→EDD→ARCH） | ✅ | ❌ | ❌ | 部分 |
| 文件對齊驗證（PRD→Code→Test） | ✅ | ❌ | ❌ | ❌ |
| BDD + TDD 閉環 | ✅ | ❌ | ❌ | 部分 |
| k8s + CI/CD 生成 | ✅ | ❌ | ❌ | 部分 |
| HTML 文件網站生成 | ✅ | ❌ | ❌ | ❌ |
| 即時程式碼補全 | ❌ | ✅ | ✅ | ✅ |
| 瀏覽器 UI（非 CLI） | ❌ | ❌ | ✅ | ✅ |
| 多人協作 | ❌ | ✅ | ✅ | 部分 |
| 既有專案改造 | ❌ | ✅ | ✅ | ✅ |
| 無需 Claude Code 前提 | ❌ | ✅ | ✅ | ✅ |
| 每次使用成本 | $6-15 | $0-10/月 | $20/月 | $15/hr |

MYDEVSOP 的真實護城河：**文件對齊驗證** + **BDD-TDD 閉環強制** + **全流程文件生成**——這三個組合是其他工具都沒有的。

---

*報告生成：devsop-product-review skill v1.0*
