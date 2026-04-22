# devsop-idea vs devsop-gendoc 詳細比較與決策表

> **用途**：逐項比對兩個 skill 的實作方式，並記錄最優方案決策。  
> **填寫方式**：在「決策」欄填入選擇，例如：`idea做法` / `gendoc做法` / `兩者補齊` / `保持不同` / `刪除`  
> **更新日期**：2026-04-22

---

## 凡例

| 符號 | 說明 |
|------|------|
| ✅ | 有此功能 |
| ❌ | 無此功能 |
| ～ | 有但做法不同 |
| → | 建議採用方向 |

---

## STEP 0：Frontmatter / 基本設定

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| 0-1 | allowed-tools：WebSearch | ✅ 有 | ❌ 無 | gendoc 要有allowed-tools：WebSearch|
| 0-2 | allowed-tools：WebFetch | ✅ 有 | ✅ 有 | |
| 0-3 | allowed-tools：Edit | ✅ 有 | ✅ 有 | |
| 0-4 | State file 命名 | 動態：`.devsop-state-{user}-{branch}.json` + symlink | 固定：`.devsop-state.json` | gendoc 要跟idea 一樣|
| 0-5 | 銜接下游 skill | → `devsop-autodev`（程式碼+部署） | → `devsop-autogen`（純文件） | |
| 0-6 | description 說明多元輸入 | ✅ 明確列出 7 種輸入類型 | ✅ 明確列出 7 種輸入類型 | |

---

## STEP -1：版本自動更新檢查

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| M1-1 | 靜默檢查版本 + 有新版時 Agent 執行 `/devsop-update` | ✅ 遵循 devsop-shared §-1 | ✅ 遵循 devsop-shared §-1 | |

---

## STEP -0.5：輸入模式選擇

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| A1 | 輸入模式選單（多輪訪談 / Quick Start / AI自動填入） | ✅ 有，設定 `_QUICK_START` 與 `_FULL_AUTO_INPUT` 旗標 | ❌ 無此步驟 | idea 移除 |
| A2 | 根據輸入字數（≤30字）預判為「較短描述」 | ✅ 有 | ❌ 無 | idea 移除|
| A3 | full-auto 快速路徑（跳過選單，直設 `_FULL_AUTO_INPUT=true`） | ✅ 有 | ❌ 無（因為無此步驟） | idea 移除 |

---

## STEP 0：Session Config / 輸入來源偵測

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| B1 | Session Config 執行時機 | Step 1.5（建立工作空間後） | Step 0（最早，進入即執行） | gendoc 也要建立空間後 |
| B2 | Session Config 內容（execution_mode + review_strategy） | ✅ 遵循 devsop-shared §0 | ✅ 遵循 devsop-shared §0 | |
| B3 | 沿用已設定值（`_EXEC_MODE` 非空時靜默繼續） | ✅ 有 | ✅ 有 | |
| B4 | full-auto 下自動套用 `standard` review_strategy | ✅ 有 | ✅ 有 | |

---

## STEP 0-B：Loop Count 精修輪數

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| C1 | 每份文件精修輪數詢問（1-5 輪選單） | ❌ 無（用 review_strategy 換算 max_rounds） | ✅ 有，key=`gendoc_loop_count`，預設 2 | idea, gendoc都不能自創，要用共用的選項，選項是從config設定的，也就是有設定好state就自動執行，若連state都沒建就如同呼叫devsop-config出來設定再往下走|
| C2 | 輪數存入 state file | ❌ 無獨立 key | ✅ `gendoc_loop_count` |idea, gendoc都不能自創，要用共用的選項，選項是從config設定的，也就是有設定好state就自動執行，若連state都沒建就如同呼叫devsop-config出來設定再往下走 |
| C3 | full-auto 下沿用已設定值或預設 2 | ❌ | ✅ 有 | idea, gendoc都不能自創，要用共用的選項，選項是從config設定的，也就是有設定好state就自動執行，若連state都沒建就如同呼叫devsop-config出來設定再往下走|
| C4 | interactive 下用 AskUserQuestion 詢問輪數 | ❌ | ✅ 有 | idea, gendoc都不能自創，要用共用的選項，選項是從config設定的，也就是有設定好state就自動執行，若連state都沒建就如同呼叫devsop-config出來設定再往下走|

---

## STEP 0-C：起始步驟選擇（斷點續跑）

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| D1 | 可從任意步驟開始（`_START_STEP`） | ❌ 無，永遠從 Step -0.5 開始 | ✅ 有，可選 DOC-01～DOC-12，key=`gendoc_start_step` | idea follow devsop-config的設定與設定流程，gendoc有自己的step, 但編號會讓人看不懂，可以01-IDEA, 02-BRD, 03-PDD, 04-EDD, ....後面以此類推，這樣比較能知道有那些檔可以產生|
| D2 | full-auto 下從 Step 1 開始，已存在文件靜默跳過 | ❌ 無 skip 邏輯 | ✅ 有 `_OVERWRITE_MODE=skip` | idea, gendoc 不要用跳過邏輯，要把舊版當參考來源，放進REQ，然後再產生新的，這樣能重跑新資料，又能記錄過去的情況|
| D3 | interactive 下用 AskUserQuestion 詢問起始步驟 | ❌ | ✅ 有 | 兩個都不要創造新的互動模式下，應該是呼叫config 出來問，若之前有呼叫要合成一個問就好，不要到處都要問一次 |

---

## STEP 0-D / 0-E：輸入來源解析 + Output Directory Guard

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| E1 | 7 種輸入類型偵測表（text/image/url/git/local/codebase...） | ✅ Step 0-A | ✅ Step 0-D-2 | |
| E2 | 遠端來源（image_url/doc_url/doc_git）立即 WebFetch + 提取摘要 | ✅ Step 0-B | ✅ Step 0-D-3 | |
| E3 | 本地來源（doc_local/codebase_local）記錄路徑，稍後處理 | ✅ Step 0-B（待 Step 1.6） | ✅ Step 0-D-3（待建立 req/） | |
| E4 | codebase_git → mktemp clone + cp 關鍵檔 + rm tmp | ✅ Step 1.6 | ✅ Step 0-D-3 | |
| E5 | codebase 複製清單（README.md/rst/txt、package.json、go.mod、Cargo.toml、pom.xml、build.gradle） | ✅ 有 README.txt / pom.xml / build.gradle | ❌ 缺 README.txt / pom.xml / build.gradle | gendoc 把缺的補上|
| E6 | docs/ 子目錄複製（`cp -r source/docs/ → req/source-docs/`） | ✅ 有 | ✅ 有 | |
| E7 | `_INPUT_TYPE` / `_INPUT_SUMMARY` / `req_dir` 寫入 state | ✅ 有 | ✅ 有 | |
| E8 | **Output Directory Guard**（docs/ 已存在時詢問 skip/overwrite/abort） | ❌ 無 | ✅ Step 0-E | 從idea, gendoc 出發都是要建新系統或新文件，因此在此之前就要決定要建在那一個目錄，而且要能標示是我們這個方法做出來的目錄，但有些情況是做一半，中斷了要接續回來做，在該新建目錄要能讀出狀態來了解之前做到那了，然後接著做，而這個保護要能判斷是全新建，那就絕對不能去建一個己經存在的目錄以及遠端REPO不能存在的才能建，而若是建一半掛的，狀態會告訴這個流程我們做到那接著做，而且檢查最後那一個STEP做的事有沒有做完，沒做完要把最後一個STEP全重做，再走下一個STEP |
| E9 | full-auto 下 docs/ 已存在時自動 skip | ❌ | ✅ 有 | 兩個都是看狀態處理 |
| E10 | interactive 下 docs/ 已存在時用 AskUserQuestion 詢問 | ❌ | ✅ 有（3選項：skip/overwrite/abort） | 兩個都是看狀態處理 |

---

## STEP 1：專案目錄設定

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| F1 | 偵測慣用根目錄（~/projects / ~/workspace / ~/code） | ✅ 有 | ❌ 在當前目錄執行 | 要跟idea一樣，但兩者都要尊守這個原則"從idea, gendoc 出發都是要建新系統或新文件，因此在此之前就要決定要建在那一個目錄，而且要能標示是我們這個方法做出來的目錄，但有些情況是做一半，中斷了要接續回來做，在該新建目錄要能讀出狀態來了解之前做到那了，然後接著做，而這個保護要能判斷是全新建，那就絕對不能去建一個己經存在的目錄以及遠端REPO不能存在的才能建，而若是建一半掛的，狀態會告訴這個流程我們做到那接著做，而且檢查最後那一個STEP做的事有沒有做完，沒做完要把最後一個STEP全重做，再走下一個STEP" |
| F2 | AskUserQuestion 選擇目錄位置（3選項） | ✅ 有 | ❌ 無 | gendoc 在互動模式下，也要有, 自動兩者都是AI選|
| F3 | full-auto 下使用 `$(pwd)/$_PROJECT_SLUG` | ✅ 有 | ❌ 無 | gendoc要跟idea一樣|
| F4 | `mkdir -p $_PROJECT_DIR/docs` + `cd` | ✅ Step 1.5 | ❌ 無（假設已在目錄） | gendoc 要跟idea要在最前面|
| F5 | `git init` | ✅ Step 1.5 | ❌ 無 | |
| F6 | git user 設定含 fallback（whoami） | ✅ Step 1.5 | ❌ 無 | gendoc要跟idea一樣|

---

## STEP 1.5 / DOC-01 等：專案初始化 + 衝突檢查

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| G1 | PM Expert Agent 分析輸入，提取 project_name / type / features / target_users | ❌ 無獨立步驟 | ✅ DOC-01 | idea 要跟gendoc 一樣 |
| G2 | **父目錄衝突檢查**（`$_PARENT_DIR/$_PROJECT_NAME` 是否存在） | ❌ 無 | ✅ DOC-01 | 要一致，兩者都要尊守這個原則"從idea, gendoc 出發都是要建新系統或新文件，因此在此之前就要決定要建在那一個目錄，而且要能標示是我們這個方法做出來的目錄，但有些情況是做一半，中斷了要接續回來做，在該新建目錄要能讀出狀態來了解之前做到那了，然後接著做，而這個保護要能判斷是全新建，那就絕對不能去建一個己經存在的目錄以及遠端REPO不能存在的才能建，而若是建一半掛的，狀態會告訴這個流程我們做到那接著做，而且檢查最後那一個STEP做的事有沒有做完，沒做完要把最後一個STEP全重做，再走下一個STEP" |
| G3 | **GitHub repo 衝突檢查**（`gh api repos/owner/name`） | ❌ 無 | ✅ DOC-01 |  要一致，兩者都要尊守這個原則"從idea, gendoc 出發都是要建新系統或新文件，因此在此之前就要決定要建在那一個目錄，而且要能標示是我們這個方法做出來的目錄，但有些情況是做一半，中斷了要接續回來做，在該新建目錄要能讀出狀態來了解之前做到那了，然後接著做，而這個保護要能判斷是全新建，那就絕對不能去建一個己經存在的目錄以及遠端REPO不能存在的才能建，而若是建一半掛的，狀態會告訴這個流程我們做到那接著做，而且檢查最後那一個STEP做的事有沒有做完，沒做完要把最後一個STEP全重做，再走下一個STEP" |
| G4 | 衝突時 interactive：詢問加後綴/手動指定/忽略 | ❌ | ✅ 有 |  要一致，兩者都要尊守這個原則"從idea, gendoc 出發都是要建新系統或新文件，因此在此之前就要決定要建在那一個目錄，而且要能標示是我們這個方法做出來的目錄，但有些情況是做一半，中斷了要接續回來做，在該新建目錄要能讀出狀態來了解之前做到那了，然後接著做，而這個保護要能判斷是全新建，那就絕對不能去建一個己經存在的目錄以及遠端REPO不能存在的才能建，而若是建一半掛的，狀態會告訴這個流程我們做到那接著做，而且檢查最後那一個STEP做的事有沒有做完，沒做完要把最後一個STEP全重做，再走下一個STEP" |
| G5 | 衝突時 full-auto：自動加日期後綴 `$NAME-$(date +%Y%m%d)` | ❌ | ✅ 有 |  要一致，兩者都要尊守這個原則"從idea, gendoc 出發都是要建新系統或新文件，因此在此之前就要決定要建在那一個目錄，而且要能標示是我們這個方法做出來的目錄，但有些情況是做一半，中斷了要接續回來做，在該新建目錄要能讀出狀態來了解之前做到那了，然後接著做，而這個保護要能判斷是全新建，那就絕對不能去建一個己經存在的目錄以及遠端REPO不能存在的才能建，而若是建一半掛的，狀態會告訴這個流程我們做到那接著做，而且檢查最後那一個STEP做的事有沒有做完，沒做完要把最後一個STEP全重做，再走下一個STEP" |
| G6 | 初始化完成後輸出 `project_name` / `project_type` | ❌ | ✅ 有 |要一致，兩者都要尊守這個原則"從idea, gendoc 出發都是要建新系統或新文件，因此在此之前就要決定要建在那一個目錄，而且要能標示是我們這個方法做出來的目錄，但有些情況是做一半，中斷了要接續回來做，在該新建目錄要能讀出狀態來了解之前做到那了，然後接著做，而這個保護要能判斷是全新建，那就絕對不能去建一個己經存在的目錄以及遠端REPO不能存在的才能建，而若是建一半掛的，狀態會告訴這個流程我們做到那接著做，而且檢查最後那一個STEP做的事有沒有做完，沒做完要把最後一個STEP全重做，再走下一個STEP"  |

---

## STEP 1.6 / Step 0-D：素材保存至 docs/req/

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| H1 | `mkdir -p docs/req` | ✅ `$_PROJECT_DIR/docs/req` | ✅ `$(pwd)/docs/req` | 要一致，兩者都要尊守這個原則"從idea, gendoc 出發都是要建新系統或新文件，因此在此之前就要決定要建在那一個目錄，而且要能標示是我們這個方法做出來的目錄，但有些情況是做一半，中斷了要接續回來做，在該新建目錄要能讀出狀態來了解之前做到那了，然後接著做，而這個保護要能判斷是全新建，那就絕對不能去建一個己經存在的目錄以及遠端REPO不能存在的才能建，而若是建一半掛的，狀態會告訴這個流程我們做到那接著做，而且檢查最後那一個STEP做的事有沒有做完，沒做完要把最後一個STEP全重做，再走下一個STEP" |
| H2 | READ-ONLY 原則聲明（嚴禁寫入原始來源） | ✅ 有 | ✅ 有 | |
| H3 | text_idea → `docs/req/idea-input.md` | ✅ 有 | ✅ 有 | |
| H4 | image_url → `docs/req/image-analysis.md`（含原始 URL） | ✅ 有 | ✅ 有 | |
| H5 | doc_url → `docs/req/source-url.md` | ✅ 有 | ✅ 有 | |
| H6 | doc_git → `docs/req/source-git.md` | ✅ 有 | ✅ 有 | |
| H7 | doc_local → `cp` 原檔到 req/ + 更新 `_IDEA` 為檔案摘要 | ✅ 有 | ✅ 有 | |
| H8 | codebase_local → tree + cp 關鍵檔 + cp docs/ + 更新 `_IDEA` | ✅ 有 | ✅ 有 | |
| H9 | codebase_git → clone + cp + rm tmp + 更新 `_IDEA` | ✅ 有 | ✅ 有 | |
| H10 | req/ 空時建立 `.gitkeep` | ✅ 有 | ✅ 有 | |
| H11 | 完成後 `ls -la docs/req/` | ✅ 有 | ✅ 有 | |

---

## STEP 2：需求澄清（Q1-Q5）

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| I1 | Q1 主要使用者（AskUserQuestion，3-4 選項） | ✅ 有 | ❌ 無（從素材推斷） | gendoc 也要有|
| I2 | Q2 核心痛點（AskUserQuestion，3-4 選項） | ✅ 有 | ❌ | gendoc 也要有|
| I3 | Q3 技術限制或偏好（AskUserQuestion，3-4 選項） | ✅ 有 | ❌ | gendoc 也要有|
| I4 | Q4 預期使用規模（動態追問，Q3 選無限制時觸發） | ✅ 有 | ❌ | gendoc 也要有|
| I5 | Q5 其他補充（動態追問，Q2 模糊時觸發） | ✅ 有 | ❌ | gendoc 也要有|
| I6 | full-auto 下跳過全部，AI 自動推斷所有欄位 | ✅ 有 | ❌（無此步驟） | gendoc 也要有|

---

## STEP 3：網路背景研究

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| J1 | 3 次 WebSearch（競品+開源 / 技術最佳實踐 / 已知挑戰） | ✅ 有 | ❌ 無 | gendoc 也要有 |
| J2 | `_RESEARCH_SUMMARY`（競品/技術建議/已知風險） | ✅ 有 | ❌ | gendoc 也要有|
| J3 | `_RESEARCH_COUNT`（找到的參考資源數） | ✅ 有 | ❌ | gendoc 也要有|
| J4 | 研究結果進入 BRD §0 + IDEA.md Appendix A | ✅ 有 | ❌ | gendoc 也要有|

---

## STEP 4 / 初始 git commit

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| K1 | 確認工作空間（`pwd`） | ✅ Step 4 | ❌ 無獨立步驟 | gendoc 也要有,兩者都要尊守這個原則"從idea, gendoc 出發都是要建新系統或新文件，因此在此之前就要決定要建在那一個目錄，而且要能標示是我們這個方法做出來的目錄，但有些情況是做一半，中斷了要接續回來做，在該新建目錄要能讀出狀態來了解之前做到那了，然後接著做，而這個保護要能判斷是全新建，那就絕對不能去建一個己經存在的目錄以及遠端REPO不能存在的才能建，而若是建一半掛的，狀態會告訴這個流程我們做到那接著做，而且檢查最後那一個STEP做的事有沒有做完，沒做完要把最後一個STEP全重做，再走下一個STEP"|
| K2 | `git add .gitkeep && git commit -m "chore: init"` | ✅ Step 4 | ❌ | gendoc 也要有,兩者都要尊守這個原則"從idea, gendoc 出發都是要建新系統或新文件，因此在此之前就要決定要建在那一個目錄，而且要能標示是我們這個方法做出來的目錄，但有些情況是做一半，中斷了要接續回來做，在該新建目錄要能讀出狀態來了解之前做到那了，然後接著做，而這個保護要能判斷是全新建，那就絕對不能去建一個己經存在的目錄以及遠端REPO不能存在的才能建，而若是建一半掛的，狀態會告訴這個流程我們做到那接著做，而且檢查最後那一個STEP做的事有沒有做完，沒做完要把最後一個STEP全重做，再走下一個STEP"|

---

## STEP 5 / DOC-02：BRD 生成

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| L1 | 讀取 MYDEVSOP template（`$_REPO/templates/BRD.md`） | ✅ 有 | ❌ 無（直接以 Agent 生成） |gendoc 也要有 |
| L2 | Document Control 欄（DOC-ID/版本/狀態/作者/日期/建立方式） | ✅ 有 | ✅ 有 | |
| L3 | §0 背景研究（from `_RESEARCH_SUMMARY`） | ✅ 有 | ❌ 無研究結果 | gendoc 也要有|
| L4 | §1 Executive Summary（PR-FAQ 風格，假設新聞稿+FAQ） | ✅ 有 | ✅ 有 Executive Summary | |
| L5 | §2 Problem Statement（5 Whys + 問題規模 TAM/SAM/SOM） | ✅ 有 | ❌ 無 5 Whys 詳細版 |gendoc 也要有 |
| L6 | §3 Business Objectives（SMART 目標，含 ROI 3情境） | ✅ 有 | ✅ 有 Business Objectives | gendoc 要跟idea一樣|
| L7 | §3.4 RTM（業務目標→成功指標→REQ-ID 對應） | ✅ 有 | ❌ 無 |gendoc 也要有 |
| L8 | §3.5 Benefits Realization Plan（3M/6M/12M 效益測量） | ✅ 有 | ❌ 無 | gendoc 也要有|
| L9 | §4 Stakeholders（含 Target Users / Not Our Users / RACI） | ✅ 有 | ✅ 有 Stakeholders | gendoc 要跟idea一樣|
| L10 | §5 Proposed Solution（MoSCoW + Value Proposition Canvas） | ✅ 有 | ✅ 有 Scope In/Out | gendoc 要跟idea一樣|
| L11 | §6 Market & Competitive Analysis（來自研究） | ✅ 有 | ❌ 無（無研究步驟） | gendoc 要跟idea一樣|
| L12 | §7 Success Metrics（North Star + 業務指標階層 + Go/No-Go） | ✅ 有 | ✅ 有 Success Metrics |gendoc 要跟idea一樣 |
| L13 | §8 Constraints & Assumptions | ✅ 有 | ✅ 有 Constraints | gendoc 要跟idea一樣|
| L14 | §9 Regulatory & Compliance（含 DPIA/Data Governance/IP） | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣|
| L15 | §10 Risk Assessment（≥3 項，含 Mitigation） | ✅ 有 | ✅ 有 Risk Register |gendoc 要跟idea一樣 |
| L16 | §11 Business Model（商業模式畫布 + Unit Economics） | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣|
| L17 | §12 High-Level Roadmap（Mermaid timeline） | ✅ 有 | ❌ 無 |gendoc 要跟idea一樣 |
| L18 | §13 Dependencies（含 Vendor Tier 1/2/3 退出計畫） | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣|
| L19 | §14 Open Questions | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣|
| L20 | §15 Decision Log | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣|
| L21 | §16 Glossary | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣|
| L22 | §17 References | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣|
| L23 | §18 BRD→PRD Handoff Checklist（8 項確認清單） | ✅ 有 | ❌ 無 |gendoc 要跟idea一樣 |
| L24 | §19 OCM（組織變革管理，含訓練計畫/抗拒緩解） | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣|
| L25 | §20 Approval Sign-off | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣|
| L26 | git commit BRD.md | ✅ `git commit -m "docs(devsop-idea): init BRD"` | ❌ 無 git commit | gendoc 要跟idea一樣|

---

## STEP 5.5 / DOC-01.5-後：IDEA Quality Score

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| M1 | 5 維度評分（目標清晰/使用者具體/痛點可量化/範圍邊界/可行性） | ✅ 有，各 0-1 分 | ❌ 無 | gendoc 要跟idea一樣|
| M2 | 分數轉換為星等（0-5 星） | ✅ 有 | ❌ | gendoc 要跟idea一樣|
| M3 | 質性評論（優質/良好/尚可/需補充/概念模糊） | ✅ 有 | ❌ | gendoc 要跟idea一樣|
| M4 | 評分進入 Step 6 BRD 摘要 + IDEA.md §11 | ✅ 有 | ❌ | gendoc 要跟idea一樣|

---

## STEP 5.6 / DOC-02 Review：BRD Review Loop

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| N1 | 輪次上限依 review_strategy（rapid=3/standard=5/exhaustive=99） | ✅ 有 | ✅ 有（但用 `_LOOP_COUNT` from 0-B） | idea, gendoc都不能自創，要用共用的選項，選項是從config設定的，若是互動模式就叫devsop-config出來問review 的方式，若是自動模式就看共用state去跑，若連state都沒建就如同呼叫devsop-config出來設定再往下走|
| N2 | Reviewer Agent 輸出 REVIEW_JSON（findings 陣列） | ✅ 有，20+ 檢查點 | ✅ 有（較簡，8 個主要項目） | gendoc 要跟idea一樣詳細 |
| N3 | Finding 趨勢追蹤（`_FINDINGS_TREND` 每輪記錄） | ✅ 有 | ❌ 無 |gendoc 要跟idea一樣詳細 |
| N4 | Per-Round Summary（`【Round N/MAX】findings=X，已修復Y項`） | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣詳細|
| N5 | findings==0 提前 break | ✅ 有 | ✅ 有 | |
| N6 | 每輪修復後 git commit | ✅ `git commit -m "docs(pdd-review): round N fixes"` | ❌ 無 git commit |gendoc 要跟idea一樣 |
| N7 | BRD Review 完成報告（策略/輪次/趨勢/最終狀態） | ✅ 有 | ❌ 無完成報告 |gendoc 要跟idea一樣詳細 |
| N8 | BRD Review 詳細檢查點（§1-§19 逐節驗證） | ✅ 有 20+ 檢查點（包含 RTM/Benefits/Data Governance/Vendor Risk/OCM）| ✅ 有 8 個基礎項目（Executive Summary/Objectives/Metrics/Stakeholders/Scope/Constraints/Timeline/Risk）| gendoc 要跟idea一樣詳細詳細|

---

## STEP 5.7 / DOC-01.5：IDEA.md 生成

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| O1 | Document Control（DOC-ID/版本/作者/日期/建立方式/下游文件） | ✅ 11 欄（含輸入類型/來源/輸入模式） | ✅ 8 欄（含建立來源） | gendoc 要跟idea一樣詳細|
| O2 | §0 Input Source（類型/來源/素材清單/摘要） | ✅ 有 | ✅ 有 | |
| O3 | §1 Idea Essence（Elevator Pitch + 核心假說 + 成功願景） | ✅ 有 | ✅ 有（簡版） | gendoc 要跟idea一樣詳細|
| O4 | §1.4 Innovation Type Classification（Incremental/Sustaining/Adjacent/Disruptive/Radical） | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣詳細|
| O5 | §2 Problem Statement（5 Whys + 問題規模量化） | ✅ 有 | ❌ 無 |gendoc 要跟idea一樣詳細 |
| O6 | Key Features（≥5 項，含優先度） | ❌ 在 BRD §5 | ✅ 獨立節 | idea要跟gendoc一樣|
| O7 | §3 Target Users（含 JTBD + Not Our Users） | ✅ 有 JTBD | ✅ 有（無 JTBD） | gendoc 要跟idea一樣詳細|
| O8 | §4 Value Hypothesis（Value Proposition Canvas：Pain Relievers + Gain Creators） | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣詳細|
| O9 | §4.2 差異化定位 | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣詳細|
| O10 | §5 MVP & Learning Plan（MVP 邊界 + 驗證指標 + Riskiest Assumption） | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣詳細|
| O11 | §6 Clarification Interview Record（Q1-Q5 逐題記錄 + 原始 IDEA 原文） | ✅ 有 | ❌ 無（無 Q1-Q5 流程） | gendoc 要跟idea一樣詳細|
| O12 | §7 Market & Competitive Intelligence（競品表 + 技術生態 + 研究來源） | ✅ 有 | ❌ 無（無研究步驟） | gendoc 要跟idea一樣詳細|
| O13 | §8 Initial Risk Assessment（風險矩陣 + Kill Conditions + Pre-mortem） | ✅ 有 | ❌ 無 |gendoc 要跟idea一樣詳細 |
| O14 | §9 Business Potential（商業模式假說 + 戰略對齊） | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣詳細|
| O15 | §10 Executive Sponsorship & Stakeholder Alignment | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣詳細|
| O16 | §11 IDEA Quality Score（含 5 維度詳細分析） | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣詳細|
| O17 | §12 Critical Assumptions（含驗證方式/截止日） | ✅ 有 | ❌ 無 |gendoc 要跟idea一樣詳細 |
| O18 | §13 Open Questions | ✅ 有 | ❌ 無 |gendoc 要跟idea一樣詳細 |
| O19 | §14 Handoff Checklist（12 項：一句話/假說/使用者/痛點/規模/競品/風險/Kill Conditions...） | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣詳細|
| O20 | §15 Traceability Note（需求溯源說明，ECR 比對準則） | ✅ 有 | ✅ 有 Forward Traceability | gendoc 要跟idea一樣詳細|
| O21 | Appendix A：Research Raw Data（搜尋詞 + 結果） | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣詳細|
| O22 | Appendix B：Document History | ✅ 有 | ❌ 無 |gendoc 要跟idea一樣詳細 |
| O23 | Appendix C：Attached Materials（附件清單 + 應用映射到具體文件章節） | ✅ 有 | ✅ 有（簡版 Appendix） | gendoc 要跟idea一樣詳細|
| O24 | git commit IDEA.md | ✅ 暗示有 | ❌ 無明確 git commit | gendoc 要跟idea一樣詳細|
| O25 | HTML 生成（`gen_html.py` → `idea.html`） | ✅ Step 5.7.1 | ❌ 無 | gendoc 要跟idea一樣詳細|

---

## STEP 6 / DOC-03-B：Handoff 摘要

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| P1 | 結構化 BRD 摘要展示（專案/使用者/痛點/技術/功能/研究/評分） | ✅ Step 6 | ❌ 無摘要展示，直接跳 Handoff | gendoc 要跟idea一樣詳細|
| P2 | Handoff Banner（表格形式：專案/輸入/文件行數/輪次） | ❌ 無 Banner | ✅ DOC-03-B 有完整 Banner | idea 要跟gendoc一樣詳細|

---

## STEP 7 / DOC-03-C：使用者確認

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| Q1 | AskUserQuestion 詢問下一步（3選項） | ✅ 「開始autodev」/「查看/修改BRD」/「重新整理BRD」 | ✅ 「立即啟動autogen」/「先查看BRD」/「稍後手動執行」 | gendoc 改成「開始autogen」/「查看/修改BRD」/「重新整理BRD」 |
| Q2 | 選「查看/修改 BRD」→ 輸出全文，修改後回到確認選單 | ✅ 有 | ✅ 有（顯示 BRD 後回選單） |  gendoc 要跟idea一樣詳細|
|
| Q3 | 選「重新整理 BRD」→ 回 Step 2 重新提問（不重建目錄/不重做研究） | ✅ 有 | ❌ 無 | gendoc 要跟idea一樣詳細|
|
| Q4 | full-auto 下直接輸出「直接啟動...」跳過詢問 | ✅ 有 | ✅ 有 | |

---

## STEP 8 / DOC-03-A+D：Handoff State + 呼叫下游 Skill

| # | 項目 | idea 作法 | gendoc 作法 | 決策 |
|---|------|-----------|-------------|------|
| R1 | 寫入 Handoff State（`handoff=true` / `brd_path` / `idea_path` / `req_dir` / `handoff_source`） | ❌ 無寫 state | ✅ DOC-03-A 完整寫入 | idea 要跟 gendoc 一樣，但變數，idea吃config那邊的步數，gendoc 自己獨立|
| R2 | 告知下游 skill 可從哪步開始（`brd_path` 已就緒，跳過前幾步） | ❌ | ✅ 有（`gendoc_handoff=True`） | idea 要跟 gendoc 一樣，但變數，idea吃config那邊的步數，gendoc 自己獨立 |
| R3 | 用 Skill 工具呼叫下游 | ✅ Step 8（Skill tool → `devsop-autodev`） | ✅ DOC-03-D（Skill tool → `devsop-autogen`） | 維持兩邊主要就是步數獨立，下游不同|
| R4 | 無法自動觸發時提示使用者手動執行指令 | ✅ 有 | ✅ 有 | |

---

## 決策彙總（待填寫）

填寫完上方各項目後，在此彙總需要修改的項目：

### gendoc 需補充的項目
> 請在決策欄填完後，將「gendoc 補充」的項目條列於此

- [ ] （待填）

### idea 需補充的項目
> 請在決策欄填完後，將「idea 補充」的項目條列於此

- [ ] （待填）

### 保持設計不同（不需修改）
> 兩者定位不同導致的合理差異

- [ ] （待填）

---

*此文件由 MYDEVSOP 框架維護。最後更新：2026-04-22*
