# MYDEVSOP Skill 修改決策文件

> **版本**：v1.0 — 2026-04-22  
> **用途**：以第一性原則重新分析 devsop-idea / devsop-gendoc 的差距，整理需決策的修改項目。  
> **填寫方式**：在「決策」欄填入選擇，例如：`採納` / `否決` / `調整後採納：...`  
> **閱讀建議**：Tier 1 → Tier 2 → Tier 3 → Tier 4 依序決策，後面的項目可能依賴前面的決定。

---

## 第一性原則分析框架

在逐項決策前，先確認三個根本問題的答案：

### 根本問題 1：這兩個 skill 的本質差異是什麼？

| | devsop-idea | devsop-gendoc |
|--|-------------|---------------|
| 輸入特性 | 無中生有（想法尚未成形）+ 已有素材（文件/程式碼/圖片） | 無中生有（想法尚未成形）+ 已有素材（文件/程式碼/圖片）） |
| 需求清晰度 |  低到高都要能接受，但不管多清楚都要做 澄清與研究, 表材只是參考資料之一 |低到高都要能接受，但不管多清楚都要做 澄清與研究, 表材只是參考資料之一 |
| 輸出 | IDEA.md + BRD.md (要先idea.md, 再關聯idea.md產生 BRD.md 跟專案要求文件上下遊的累積關聯有關) → autodev（有程式碼 有文件） | IDEA.md + BRD.md (要先idea.md, 再關聯idea.md產生 BRD.md 跟專案要求文件上下遊的累積關聯有關)  → autogen（純文件） |
| 下游 | 程式碼 + 部署 + local 測試環境啟動 + 文件 + HTML Pages +git pus and deploy page| 文件 + HTML Pages+git pus and deploy page |

**核心結論**：兩者輸出物（ IDEA＋BRD.md）應達到相同品質標準，但抵達輸出的「過程」可以因輸入性質不同而有所差異。

### 根本問題 2：修改的優先原則是什麼？

1. **安全性優先**：不能意外覆蓋已存在的工作成果
2. **正確性次之**：斷點後能正確接續
3. **品質第三**：輸出文件的完整度
4. **一致性最後**：兩 skill 共用設計模式

### 根本問題 3：什麼是「最小可行」修改集？

有些修改是**架構正確性**所必需（不改就會出錯），有些只是**品質提升**（改了更好但不影響正確性）。本文件明確區分兩者。

---

## Tier 1：架構基礎決策（最高優先，其他一切依賴於此）

---

### TF-01：gendoc 的工作空間創建模式

**影響**：gendoc｜**類型**：架構正確性（必改）

**現況**

gendoc 目前假設使用者**已在**某個目錄下執行，直接在 `$(pwd)/docs/` 寫出文件。它沒有「偵測根目錄 → 詢問位置 → mkdir → cd → git init」的流程。

**第一性原則分析**

若 gendoc 要實作「全新建時禁止覆蓋已存在目錄」的保護（即 TF-02 斷點守衛），前提是 gendoc 必須**明確知道並控制**工作空間的建立。但目前它不建立工作空間，無從實施保護。

這揭示了一個根本性設計問題：

- **現有設計**：gendoc 是一個「就地執行」的 skill（在呼叫者當前目錄運行）
- **目標設計**：gendoc 變成一個「自主建立工作空間」的 skill（與 idea 相同架構）

這兩種設計不能同時成立。必須選一個。

**方案選擇**

| 方案 | 描述 | 優點 | 缺點 |
|------|------|------|------|
| **A（自主建立）** | gendoc 完全複製 idea 的 Step 1～1.5 工作空間建立流程，從偵測根目錄到 git init | 架構一致；可實施 TF-02 守衛；支援斷點續跑 | 工作量最大；需大幅重寫 Step 0 前段 |
| **B（就地執行+守衛）** | 保持「就地執行」模式，但在 Step 0 前加入「此目錄是否由 MYDEVSOP 管理」的識別標記檢查 | 改動最少；保持現有使用慣例 | 識別標記的可靠性存疑；無法完整實施 TF-02 |
| **C（混合模式）** | gendoc 保持就地執行，但 Step 0 之前新增「是否在 MYDEVSOP 目錄中」偵測：是→斷點續跑；否→詢問是否建立新工作空間 | 靈活；向下相容 | 邏輯複雜；有兩條不同路徑，測試負擔重 |

**建議**：方案 A（完全對齊 idea 架構），理由是長期維護成本最低，且 TF-02 守衛的實施最為乾淨。

| 決策 | |
|------|--|
| **你的決策** | A，但idea 要先調到對，他現在也有錯，在/Users/tobala/projects/MYDEVSOP/docs/idea-vs-gendoc-comparison.md我有明確指出，要先改好，才能完整複制idea|

---

### TF-02：斷點續跑狀態機設計

**影響**：兩者｜**類型**：架構正確性（必改）

**現況**

- **gendoc**：Step 0-E 有「docs/ 已存在時 skip/overwrite/abort 三選項」，但邏輯依賴使用者選擇，不是基於 state 自動判斷
- **idea**：完全無此保護

**第一性原則分析**

正確的狀態機應滿足以下不變量（invariants）：

```
不變量 1：全新建 ≡ state file 不存在
不變量 2：斷點續跑 ≡ state file 存在 + completed_steps 陣列不完整
不變量 3：已完成 ≡ state file 存在 + 所有 STEP 都在 completed_steps 中
但兩者有不同的STEP，不能共用，要在devsop-config 多一個設定doc 文件產生的STEP，這樣才會各自獨立
```

在此基礎上，守衛邏輯為：

```
if state_file 不存在:
  → 全新建路徑
  → 驗證：目標專案名目錄 （不一定是父目錄）不存在 AND GitHub repo 不存在
  → 若任一存在 → ABORT（給出精確原因）但在自動模式（devsop-config 可以選）下，不能無故ABORT，要給出一個可以用的專案名稱，可以用在建專案目錄與REPO

elif state_file 存在, 但要確定是這個流程的狀態，因為現在所在目錄也有可能有別的工作在進行，所以要註記是那一個skill的狀態，同一個SKILL的狀態才能斷點續跑，你不能idea-> autodev 的狀態，拿來做gendoc-> autogen 做為斷點續跑:
  → 斷點續跑路徑
  → 讀取 completed_steps
  → 找出最後完成的 STEP N
  → 驗證 STEP N 的輸出物是否完整（見各步的 completion_criteria）
  → 若不完整：重做 STEP N，再繼續 STEP N+1
  → 若完整：從 STEP N+1 繼續
```

**關鍵問題：每個 STEP 的 completion_criteria 如何定義？ 這裡特別注意原idea流程，並沒有做idea流程的reveiw loop 這違背原設計的文件階層要補做這段相關skill, 下遊的各層文件都要關聯上來**


| STEP | 輸出物 | completion_criteria |
|------|--------|---------------------|
| 工作空間建立 | `$_PROJECT_DIR/` + `.git/` | 兩者均存在 |
| IDEA.md 生成 | `docs/IDEA.md` | 檔案存在 + `wc -l > 30` |
| IDEA Review | state 的 `idea_review_passed=true` | state key 存在且為 true |
| git commit | git log 有對應 commit | `git log --oneline | grep "init idea"` |
| BRD 生成 | `docs/BRD.md` | 檔案存在 + `wc -l > 50` |
| git commit | git log 有對應 commit | `git log --oneline | grep "init BRD"` |
| BRD Review | state 的 `brd_review_passed=true` | state key 存在且為 true |

**與現有 gendoc skip 邏輯的根本差異**

- **現有 skip 邏輯**：「文件存在就跳過」→ 問題：文件可能是不完整的（被中斷的上次生成）
- **新邏輯**：「state 記錄此 STEP 完成」→ 可靠，不依賴文件是否存在

**方案選擇**

| 方案 | 描述 | 優點 | 缺點 |
|------|------|------|------|
| **A（完整狀態機）** | 上述完整設計，每步寫入 `completed_steps` 陣列，守衛在 Step 0 最前端執行 | 最正確；完全可靠 | 實施複雜；每步都要寫 state |
| **B（輕量守衛）** | 只檢查「state 存在 vs 不存在」，不追蹤個別 STEP，斷點後從 `gendoc_start_step` 指定的步驟重跑 | 實施簡單；利用現有 start_step 機制 | 不能自動找斷點位置；需使用者知道從哪裡接 |
| **C（文件存在守衛）** | 全新建時禁止目錄/repo 衝突；斷點後沿用現有 skip 邏輯 | 改動最少 | skip 邏輯的可靠性問題未解決 |

**建議**：方案 A，但以漸進方式實施（先實施全新建守衛，再實施斷點追蹤）。

| 決策 | |
|------|--|
| **你的決策** | A ，但不用該檔案每一個細節都找，因為每份文件的產生都有TEMPLATE參考，只要跟要執行的項目對應不起來，就要重建該文件 |

---

### TF-03：Config 委派邊界

**影響**：兩者｜**類型**：架構正確性（應改）

**現況**

- **gendoc**：自行管理 `gendoc_loop_count`（Step 0-B）和 `gendoc_start_step`（Step 0-C），用自己的 AskUserQuestion 問使用者
- **idea**：用 `review_strategy` 換算 `max_rounds`，沒有獨立 loop_count

**第一性原則分析**

Config 委派的核心原則：**執行參數不應散落在各 skill，應有單一真理來源**。

但必須注意：`devsop-config` 本身也是一個 skill，呼叫它需要有明確的觸發條件和返回機制。

**真正需要委派的是什麼？**

| 參數 | 現況 | 應由誰管理 | 理由 |
|------|------|-----------|------|
| `execution_mode` | 兩者均有，讀 state 或問使用者 | devsop-config / state | 跨 skill 共用，應只問一次 |
| `review_strategy` | 兩者均有 | devsop-config / state | 同上 |
| `loop_count` | 只 gendoc 有，自問 | gendoc 自己可管（skill 特定） | 每個 skill 的 review 輪次可以不同 |
| `start_step` | 只 gendoc 有，自問 | gendoc 自己應管 | 每個 skill 的步驟名稱不同，無法共用 |

**核心問題**：`loop_count` 和 `start_step` 本質上是 **skill-specific** 的參數，不是 cross-skill 的 config。把它們委派給 devsop-config 會讓 config 知道太多 skill 內部細節（違反關注點分離）。

**建議**

- `execution_mode` + `review_strategy`：若 state 已有，沿用；若無，顯示標準 §0 選單（保持現有做法，不需另外呼叫 devsop-config）
- `loop_count`：保留在 gendoc 自管，但**讀取 `review_strategy` 來換算預設值**（不需要問使用者）
  - `rapid` → 1 輪；`standard` → 2 輪；`thorough` → 3 輪；`exhaustive` → 5 輪
- `start_step`：保留在 gendoc 自管（斷點續跑由 TF-02 自動判斷，不需問使用者）

| 決策 | |
|------|--|
| **你的決策** | 
loop_count 就是要拿來做文件的review loop 的條件，本質跟 review_strategy 一樣，不應該讓gendoc跟其他skill不同，應取消loop_count , 直接用review-strategy 
start_step 是因為用autodev 流程的31 step, 但在其step也有分是文件產生還是非文件產生，也可以用同一個start_step, 只是要讓autogen去找出，指定的step 是自己的那一個文件step, 可以在autogen裡有autodev跟autogen 文件STEP對照表，這樣就不用兩個變數了，一個變數兩段都能用，因此當我指定的step 是autodev 非文件step 找出該step的下一個文件step來開始執行與check
因此變數共用是原則，文件產生順序要一致，以文件上下遊關係，與上下遊累積原則來看，要能符合
|

---

### TF-04：BRD 與 IDEA.md 生成邏輯的共用策略

**影響**：兩者｜**類型**：架構設計（值得決策）

**現況**

gendoc 和 idea 都生成 BRD + IDEA.md，但各自有不同的 prompt 和章節結構。

**第一性原則分析**

若兩個 skill 輸出相同的文件格式，而生成邏輯**在兩個 skill 中各自維護**，則：
- 任何模板修改都需要改兩處
- 兩份 prompt 必然漸漸偏離（維護不同步）
- Bug 修復需要在兩處修

**三種架構選擇**

| 方案 | 描述 | 優點 | 缺點 |
|------|------|------|------|
| **A（共用 devsop-shared）** | BRD 和 IDEA.md 的生成邏輯抽到 devsop-shared（共用 skill），兩者都 invoke 它 | 單一修改點；DRY | 需建立新的 devsop-shared skill；呼叫路徑增加 |
| **B（各自維護，但對齊）** | 各自保有生成邏輯，但本次修改時對齊，未來靠人工保持同步 | 零架構成本 | 長期漂移風險高 |
| **C（gendoc 呼叫 idea 的生成邏輯）** | gendoc 做完輸入處理後，把 BRD/IDEA 生成委派給 idea 的對應步驟 | 不需要額外 skill | 耦合嚴重；若 idea 改動會影響 gendoc |

**建議**：方案 B（本次對齊，長期觀察是否真的分化），原因是引入 devsop-shared 是重大架構變動，可以先對齊再看是否真的需要。

| 決策 | |
|------|--|
| **你的決策** | A，但拉出去的產生邏輯都要依文件TEMPLATE要求的項目去產生，不能TEMPLATE，說的跟實際跑的不同，例如產生idea就要依template/idea.md, 產生BRD就要依template/BRD.md, 其他文件以此類推，若其他文件沒有這麼做也要改尤其在autodev, autogen裡沒有尊守此原則都要改，因為維護只在template而不是skill, 精練項目要尊從統一來源，template|

---

### TF-05：State File 命名統一

**影響**：gendoc｜**類型**：架構正確性（應改）

**現況**

- gendoc：固定使用 `.devsop-state.json`
- idea：動態使用 `.devsop-state-{git_user}-{branch}.json` + symlink

**第一性原則分析**

固定命名在多人協作或多分支場景下會互相覆蓋。例如：
- Alice 在 `main` 分支跑 gendoc，state 為 `.devsop-state.json`
- Bob 在 `feature/v2` 分支跑 gendoc，同樣寫入 `.devsop-state.json`
- → 兩者互相覆蓋，斷點續跑失效

動態命名（user + branch）解決了這個問題。

**潛在問題**：若 TF-01 決定 gendoc 採方案 A（自主建立工作空間），則 state file 的路徑需要在工作空間建立後才能確定，這影響 Step 0 的 Session Config 讀取時機（見 TF-06）。

**建議**：gendoc 改為與 idea 相同的動態命名，在工作空間建立後（Step 1 之後）才初始化 state file。

| 決策 | |
|------|--|
| **你的決策** | gendoc 跟idea 要一樣，但不能有+ symlink 和`.devsop-state.json` 統一格式`.devsop-state-{git_user}-{branch}.json`|

---

### TF-06：Session Config 執行時機

**影響**：gendoc｜**類型**：架構正確性（依賴 TF-01）

**現況**

- gendoc：Step 0 最前端執行 Session Config（讀 `.devsop-state.json`）
- idea：Step 1.5 執行（工作空間建立後）

**第一性原則分析**

若 TF-01 決定 gendoc 採方案 A（自主建立工作空間）：
- Session Config 必須在工作空間建立**之後**才能執行（因為 state file 在工作空間裡）
- gendoc 的 Step 0 Session Config 必須後移到等效於 idea Step 1.5 的位置

若 TF-01 決定 gendoc 採方案 B 或 C（保持就地執行）：
- Session Config 保持在 Step 0 執行（維持現有邏輯）

此決策**完全依賴 TF-01**。

| 決策 | |
|------|--|
| **你的決策**（填 TF-01 決定後自然解決） | A，但gendoc 跟idea 要一樣，不能有+ symlink 和`.devsop-state.json` 統一格式`.devsop-state-{git_user}-{branch}.json`|

---

### TF-07：Step 命名規範（gendoc）

**影響**：gendoc｜**類型**：可讀性（可改可不改）

**現況**

gendoc 使用 `DOC-01`～`DOC-12` 命名，使用者看不出每步產生的是什麼文件。

**第一性原則分析**

`DOC-01` 是 opaque ID，不傳達語意。`01-IDEA` 明確告訴讀者此步產生 IDEA.md。

**改名帶來的問題**

- 若其他 skill（autogen、autodev）有寫死 `gendoc_start_step` 的步驟名稱，改名後需要同步更新
- state file 中存的 `gendoc_start_step` 值是舊名稱，升級後舊 state 會失效

**建議映射**

| 舊名 | 建議新名 | 輸出文件 |
|------|---------|---------|
| DOC-01 | `01-INIT` | 專案初始化 + 衝突檢查 |
| DOC-01.5 | `02-IDEA` | IDEA.md |
| DOC-02 | `03-BRD` | BRD.md |
| DOC-03 | `04-HANDOFF` | Handoff → autogen |

（後續 DOC-04 以後在 autogen 中，不在 gendoc 範圍）

**建議**：採納，但同時在 state 中保留舊名 alias 做向後相容（`gendoc_start_step: "02-IDEA"` 或 `"DOC-01.5"` 均可解析）。

| 決策 | |
|------|--|
| **你的決策** | 要有對照但是是autodev的31 step 文件step 標示，也不要有alias, 一次改對，不應該讓gendoc跟其他skill不同，應取消loop_count , 直接用review-strategy 
start_step 是因為用autodev 流程的31 step, 但在其step也有分是文件產生還是非文件產生，也可以用同一個start_step, 只是要讓autogen去找出，指定的step 是自己的那一個文件step, 可以在autogen裡有autodev跟autogen 文件STEP對照表，這樣就不用兩個變數了，一個變數兩段都能用，因此當我指定的step 是autodev 非文件step 找出該step的下一個文件step對照到autogen step來開始執行與check
因此變數共用是原則，文件產生順序要一致，以文件上下遊關係，與上下遊累積原則來看，要能符合|

---

## Tier 2：流程設計決策

---

### PD-01：idea 移除 Step -0.5 輸入模式選單

**影響**：idea｜**類型**：流程簡化（建議改）

**現況**

idea 的 Step -0.5 詢問使用者選擇「多輪訪談 / Quick Start / AI 自動填入」，設定 `_QUICK_START` 和 `_FULL_AUTO_INPUT` 旗標。

**第一性原則分析**

這個選單存在是因為 `execution_mode`（interactive/full-auto）和「是否讓 AI 完全自主推斷 Q1-Q5」是兩個**不完全等價**的維度：

- `interactive` + 使用者願意回答 → 多輪訪談
- `interactive` + 使用者懶得回答 → Quick Start
- `full-auto` → AI 自動填入

問題是：`interactive` 模式下的「Quick Start」實際上是「interactive 模式，但跳過 Step 2 的問題」。這個旗標只用來決定**是否執行 Step 2 的澄清問題**。

**第一性原則結論**：這個選單可以被簡化掉。如果使用者是 `interactive` 模式，就執行 Step 2 澄清問題（使用者可以直接選「AI 推薦預設」跳過）；如果是 `full-auto`，就直接跳過 Step 2。中間不需要一個額外的模式選單。

**建議**：移除 Step -0.5，刪除 `_QUICK_START` 和 `_FULL_AUTO_INPUT` 旗標。Step 2 的執行完全由 `execution_mode` 決定。

| 決策 | |
|------|--|
| **你的決策** | 不要有這個選單STEP，因為devsop-config 就有選自動與互動，就夠了，找不到狀態就直接叫出devsop-config處理，不要有太多散在各skill類似意思的選單|

---

### PD-02：gendoc 是否需要 Q1-Q5 澄清步驟

**影響**：gendoc｜**類型**：流程設計（重要判斷）

**現況**

gendoc 無 Q1-Q5 步驟（從素材直接推斷）。使用者決策：gendoc 也要有 Q1-Q5。

**第一性原則分析**

Q1-Q5 的**存在理由**是：當輸入（IDEA 文字）不夠具體時，AI 需要更多資訊才能生成高品質 BRD。

gendoc 的輸入性質與 idea 不同：

| 輸入類型 | Q1-Q5 必要性 | 理由 |
|---------|------------|------|
| `text_idea`（純文字想法） | 高 | 與 idea 相同情況 |
| `doc_url`（規格文件） | 低 | 文件已包含大部分答案 |
| `doc_git`（GitHub repo） | 低 | README/docs 已有 context |
| `codebase_local/git` | 中 | 程式碼隱含技術決策，但使用者意圖不明 |
| `image_url`（截圖） | 高 | 圖片只提供 UI，缺少業務 context |

**第一性原則結論**：gendoc 應做**條件式澄清**（Conditional Clarification），而非全面 Q1-Q5：

```
if _INPUT_TYPE in ["text_idea", "image_url"]:
    → 執行完整 Q1-Q5（與 idea 相同）
elif _INPUT_TYPE in ["doc_url", "doc_git", "codebase_local", "codebase_git"]:
    → 從素材推斷 Q1-Q5 答案
    → 僅在推斷可信度低時（AI 判斷）追問特定問題
    → interactive 模式：顯示推斷結果，詢問「確認/修正？」
    → full-auto 模式：直接使用推斷結果
```

**方案選擇**

| 方案 | 描述 | 建議 |
|------|------|------|
| **A（條件式）** | 依輸入類型決定是否執行 Q1-Q5 | ✅ 最合理 |
| **B（全面 Q1-Q5）** | 所有輸入類型都問 Q1-Q5 | 文件類輸入問法重複且低效 |
| **C（不問）** | 保持現有，全部從素材推斷 | 損失部分準確度 |

| 決策 | |
|------|--|
| **你的決策** | B 全部都要有，不管誰給什麼資料都是參考來源，不是給的多就決定項目內容了，AI要自主研究只是加入更多實際參考資料，最後統一分析，而devsop-idea 也要求要能接收更多input 因此對兩個流程來說都是一樣的 |

---

### PD-03：gendoc 是否需要 Web Research

**影響**：gendoc｜**類型**：流程設計（重要判斷）

**現況**

gendoc 無 Web Research 步驟。使用者決策：gendoc 也要做 3 次 WebSearch。

**第一性原則分析**

Web Research 的**存在理由**：填補「從想法到 BRD」過程中的背景知識空白（競品分析、技術選型、已知風險）。

gendoc 的問題：如果輸入已是完整的規格文件或 codebase，**研究的邊際價值大幅降低**：

| 輸入類型 | 研究必要性 | 理由 |
|---------|----------|------|
| `text_idea` | 高 | 與 idea 相同 |
| `doc_url`（詳細規格） | 低 | 規格已決定了技術與範疇 |
| `codebase_local/git` | 中 | 程式已決定技術棧，但市場情況未知 |
| `doc_git`（開源專案） | 中 | 可補充競品對比資訊 |

**第一性原則結論**：gendoc 應做**條件式研究**（1 次 targeted WebSearch），而非複製 idea 的 3 次全面研究。

```
若 _INPUT_TYPE in ["text_idea", "image_url"]:
    → 執行 3 次 WebSearch（同 idea）
若 _INPUT_TYPE in ["codebase_local", "codebase_git", "doc_git"]:
    → 執行 1 次 WebSearch（僅競品/市場定位）
若 _INPUT_TYPE in ["doc_url"]:
    → 跳過研究（規格已自足）
```

**方案選擇**

| 方案 | 描述 | 建議 |
|------|------|------|
| **A（條件式研究）** | 依輸入類型決定研究深度 | ✅ 最合理 |
| **B（全面 3 次研究）** | 所有輸入類型都做 3 次 | 部分情況冗餘 |
| **C（不研究）** | 保持現有，無研究步驟 | BRD §6 競品分析為空 |

| 決策 | |
|------|--|
| **你的決策** | B，不管誰給什麼資料都是參考來源，不是給的多就決定項目內容了，AI要自主研究只是加入更多實際參考資料，最後統一分析，而devsop-idea 也要求要能接收更多input 因此對兩個流程來說都是一樣的|

---

### PD-04：gendoc Loop Count 來源

**影響**：gendoc｜**類型**：流程設計（依賴 TF-03）

**現況**

gendoc Step 0-B 自行用 AskUserQuestion 問使用者選 1-5 輪，存 `gendoc_loop_count`。

**第一性原則分析**

TF-03 建議用 `review_strategy` 換算 loop_count，而非獨立問使用者。

**換算關係建議**

```
review_strategy → max_rounds（每份文件）
rapid       → 1
standard    → 2
thorough    → 3
exhaustive  → 5
```

這樣做的好處：使用者只需在 Session Config 選一次「review 策略」，不需要再被問「幾輪？」。

**方案選擇**

| 方案 | 描述 | 建議 |
|------|------|------|
| **A（從 review_strategy 換算）** | 移除 Step 0-B 的詢問，用 review_strategy 自動換算 | ✅ 建議 |
| **B（保持自問）** | 維持現有 Step 0-B，使用者選輪數 | 多了一次詢問 |
| **C（固定輪數）** | 不問，固定使用 2 輪 | 缺少彈性 |

| 決策 | |
|------|--|
| **你的決策** |不應該有loop count, 其本質跟 review_strategy 一樣，不應該讓gendoc跟其他skill不同，應取消loop_count , 直接用review-strategy 
start_step 是因為用autodev 流程的31 step, 但在其step也有分是文件產生還是非文件產生，也可以用同一個start_step, 只是要讓autogen去找出，指定的step 是自己的那一個文件step, 可以在autogen裡有autodev跟autogen 文件STEP對照表，這樣就不用兩個變數了，一個變數兩段都能用，因此當我指定的step 是autodev 非文件step 找出該step的下一個文件step來對應autogen 文件step開始執行與check
因此變數共用是原則，文件產生順序要一致，以文件上下遊關係，與上下遊累積原則來看，要能符合 |

---

### PD-05：起始步驟選擇（gendoc Start Step）

**影響**：gendoc｜**類型**：流程設計（依賴 TF-02）

**現況**

gendoc Step 0-C 在 interactive 模式下用 AskUserQuestion 問使用者「從 DOC-01 到 DOC-12 哪步開始」。

**第一性原則分析**

若 TF-02 決定實施完整狀態機（方案 A），則：
- 斷點位置由系統自動判斷，不需要問使用者
- 全新建永遠從 STEP 1 開始
- 使用者手動指定起始步驟的場景只在「想重跑特定步驟」時有用

**建議**

- 移除 `interactive` 下的起始步驟 AskUserQuestion
- 改為：斷點自動偵測（TF-02）；若使用者想重跑特定步驟，透過呼叫時的 args 傳入
- 若 TF-02 選方案 B（輕量守衛），則保留此步驟

| 決策 | |
|------|--|
| **你的決策**（填 TF-02 決定後自然解決） | 不應該有loop count, 其本質跟 review_strategy 一樣，不應該讓gendoc跟其他skill不同，應取消loop_count , 直接用review-strategy 
start_step 是因為用autodev 流程的31 step, 但在其step也有分是文件產生還是非文件產生，也可以用同一個start_step, 只是要讓autogen去找出，指定的step 是自己的那一個文件step, 可以在autogen裡有autodev跟autogen 文件STEP對照表，這樣就不用兩個變數了，一個變數兩段都能用，因此當我指定的step 是autodev 非文件step 找出該step的下一個文件step來對應autogen 文件step開始執行與check
因此變數共用是原則，文件產生順序要一致，以文件上下遊關係，與上下遊累積原則來看，要能符合|

---

### PD-06：idea 補充 PM Expert Agent + 衝突檢查

**影響**：idea｜**類型**：功能補充（應改）

**現況**

idea 目前在 Step 0-A 用 `_PROJECT_SLUG` 做初步的名稱推斷，但無 PM Expert Agent 正式提取 `project_name`、`project_type`、`key_features`、`target_users`。gendoc 的 DOC-01 有這個完整流程。

idea 目前也沒有父目錄 + GitHub repo 的衝突檢查（只在 Step 1.5 建立目錄，若目錄已存在會直接報錯）。

**建議**

在 idea 的 Step 1.5（工作空間建立後，Session Config 執行後）新增等效於 gendoc DOC-01 的 PM Expert Agent + 衝突檢查邏輯。

| 決策 | |
|------|--|
| **你的決策** | idea 要加|

---

### PD-07：舊版文件處理邏輯（skip/overwrite → REQ）

**影響**：gendoc｜**類型**：流程正確性（建議改）

**現況**

gendoc Step 0-E 和 Step 0-C 的 skip 邏輯：已存在文件靜默跳過（full-auto）或問使用者（interactive）。

**第一性原則分析**

「跳過」邏輯的問題：如果輸入素材更新了（新版規格文件），跳過已存在的文件意味著無法更新它。

用戶的決策（D2）：「不要用跳過邏輯，要把舊版當參考來源，放進 REQ，然後再產生新的」。

這個設計的問題：
1. 如何知道「舊版」是 MYDEVSOP 生成的，還是使用者自己寫的？（建議：只有在 MYDEVSOP 的 state 中有記錄的文件才算「舊版」）
2. 重新生成意味著每次跑都會產生新版，BRD Review 的成果也會被清空 → 可能需要保留 review_passed state

**建議**

```
if 目標文件存在 AND state 記錄此文件為「已生成」:
    mv 舊版 → docs/req/old-{filename}-{timestamp}.md
    重新生成新版
elif 目標文件存在 AND state 無此文件記錄:
    詢問使用者如何處理（互動模式）或直接覆蓋（full-auto）
```

| 決策 | |
|------|--|
| **你的決策** |
if 目標文件存在 AND state 記錄此文件為「已生成」:
    mv 舊版 → docs/req/old-{filename}-{timestamp}.md
    重新生成新版
elif 目標文件存在 AND state 無此文件記錄:
    詢問使用者如何處理（互動模式）或直接copy → docs/req/old-{filename}-{timestamp}.md（full-auto）
    重新產生文件
|

---

## Tier 3：內容品質決策

---

### CQ-01：BRD 章節對齊（gendoc 補充 §0-§20）

**影響**：gendoc｜**類型**：內容品質（提升）

**現況**

gendoc DOC-02 BRD 目前包含：Executive Summary / Business Objectives / Success Metrics / Stakeholders / Scope / Constraints / Timeline / Risk Register（共 8 個核心段落）。

idea 的 BRD 包含 §0-§20 共 21 個段落，含：5 Whys / TAM-SAM-SOM / ROI 情境 / RTM / Benefits Realization Plan / Value Proposition Canvas / Market Analysis / Business Model Canvas / Unit Economics / Roadmap / Dependencies / OCM / Approval Sign-off 等。

**第一性原則分析**

BRD 的核心是「讓決策者能做出 Go/No-Go 決定」。過多章節可能反而降低閱讀率。

**建議分層**

| 層級 | 章節 | 必要性 |
|------|------|--------|
| **核心層**（8 章，gendoc 已有） | Executive Summary / Objectives / Metrics / Stakeholders / Scope / Constraints / Timeline / Risk | 所有情況必有 |
| **業務分析層**（+5 章） | §2 Problem Statement 5 Whys / §6 Market Analysis / §11 Business Model / §12 Roadmap / §14 Open Questions | 適用「業務驅動」型專案 |
| **治理層**（+5 章） | §3.4 RTM / §3.5 Benefits Realization / §9 Compliance / §19 OCM / §20 Approval Sign-off | 適用「大型組織」或「合規敏感」專案 |

**方案選擇**

| 方案 | 描述 | 建議 |
|------|------|------|
| **A（完全對齊 idea 21 章）** | 機械性對齊，全部補上 | 可能過重 |
| **B（核心+業務分析共 13 章）** | 跳過治理層（小團隊通常不需要） | ✅ 建議 |
| **C（條件式）** | 根據 `project_type` 或使用者規模判斷是否加入治理層 | 實施複雜 |

| 決策 | |
|------|--|
| **你的決策** | 產生邏輯都要依文件TEMPLATE要求的項目去產生，不能TEMPLATE，說的跟實際跑的不同，例如產生idea就要依template/idea.md, 產生BRD就要依template/BRD.md, 其他文件以此類推，若其他文件沒有這麼做也要改尤其在autodev, autogen裡沒有尊守此原則都要改，因為維護只在template而不是skill, 精練項目要尊從統一來源，template|
|

---

### CQ-02：IDEA.md 章節對齊（gendoc 補充至 15 節 + 3 Appendix）

**影響**：gendoc｜**類型**：內容品質（提升）

**現況**

gendoc DOC-01.5 的 IDEA.md 包含：Document Control / Idea Essence / Input Source / Key Features / Target Users / Forward Traceability / Appendix（共 7 節）。

idea 的 IDEA.md 包含 §0-§19 + Appendix A/B/C，含：Innovation Type / 5 Whys / Value Hypothesis / MVP Plan / Clarification Record / Market Intelligence / Risk Assessment / Business Potential / Quality Score / Critical Assumptions / Open Questions / Handoff Checklist 等。

**第一性原則分析**

IDEA.md 是需求鏈的 Layer 0 起點，記錄「在 BRD 之前的原始想法與推理過程」。

gendoc 的 IDEA.md 目前**缺少推理過程**（為什麼做這個、問題是什麼、風險在哪），只記錄了「是什麼」，沒有記錄「為什麼」。

**建議補充的最高優先節**（不論輸入類型都應有）

| 節次 | 內容 | 為什麼必要 |
|------|------|----------|
| §2 Problem Statement | 5 Whys + 問題量化 | 沒有「問題陳述」的 IDEA 是不完整的 |
| §4 Value Hypothesis | Value Proposition Canvas | BRD §5 的前置輸入 |
| §8 Risk Assessment | 風險矩陣 + Kill Conditions | 早期識別可能失敗的原因 |
| §11 Quality Score | 5 維度評分 | 量化 IDEA 的完整度 |
| §14 Handoff Checklist | 12 項確認清單 | 確保移交 BRD 前所有資訊齊備 |

| 決策 | |
|------|--|
| **你的決策**（可逐節決定是否加入） | 產生邏輯都要依文件TEMPLATE要求的項目去產生，不能TEMPLATE，說的跟實際跑的不同，例如產生idea就要依template/idea.md, 產生BRD就要依template/BRD.md, 其他文件以此類推，若其他文件沒有這麼做也要改尤其在autodev, autogen裡沒有尊守此原則都要改，因為維護只在template而不是skill, 精練項目要尊從統一來源，template|
|

---

### CQ-03：IDEA Quality Score（gendoc 新增）

**影響**：gendoc｜**類型**：內容品質（建議加）

**現況**

idea 有 5 維度評分（目標清晰 / 使用者具體 / 痛點可量化 / 範圍邊界 / 可行性），各 0-1 分，轉為 0-5 星 + 質性評語。gendoc 無此步驟。

**第一性原則分析**

Quality Score 的作用：提示 AI 和使用者「這個 IDEA 的資訊是否足夠生成高品質 BRD」。如果分數低，應在繼續前補充資訊。

對 gendoc 而言，當輸入是完整規格文件時，分數通常會高；當輸入是截圖或模糊文字時，分數會低，此時可提示使用者補充。

**建議**：gendoc 加入 Quality Score，在 IDEA.md 生成後、BRD 生成前執行。若分數 ≤ 2★，互動模式下提醒使用者補充資訊，full-auto 模式下標記警告繼續。

| 決策 | |
|------|--|
| **你的決策** | 目前idea.md 在devsop-idea, devsop-gendoc 都沒有review loop 要加，而且共用devsop-config的reviw 條件|

---

### CQ-04：BRD Review Loop 詳細化（gendoc）

**影響**：gendoc｜**類型**：內容品質（建議改）

**現況**

gendoc 的 BRD Reviewer 有 8 個基本檢查點（Executive Summary / Objectives / Metrics / Stakeholders / Scope / Constraints / Timeline / Risk）。idea 有 20+ 個檢查點（逐節驗證，含 RTM / Benefits / Data Governance / Vendor Risk / OCM）。

**第一性原則分析**

Reviewer 的檢查點數量應與 BRD 的章節數量一致（CQ-01 決定後才能確定）。

**建議**：CQ-01 決定 BRD 章節數量後，Reviewer 的檢查點同步對齊。不需要在 CQ-01 之前獨立決策。

**附加項目（不依賴章節數量）**

| 項目 | 現況 | 建議 |
|------|------|------|
| `_FINDINGS_TREND` 趨勢追蹤 | gendoc 無 | 加入（每輪記錄 findings 數，判斷是否改善） |
| Per-Round Summary 輸出 | gendoc 無 | 加入（`【Round N/MAX】findings=X` 格式） |
| 每輪修復後 git commit | gendoc 無 | 加入 |
| Review 完成報告 | gendoc 無 | 加入 |

| 決策 | |
|------|--|
| **你的決策** | 產生邏輯都要依文件TEMPLATE要求的項目去產生，不能TEMPLATE，說的跟實際跑的不同，例如產生idea就要依template/idea.md, 產生BRD就要依template/BRD.md, 其他文件以此類推，若其他文件沒有這麼做也要改尤其在autodev, autogen裡沒有尊守此原則都要改，因為維護只在template而不是skill, 精練項目要尊從統一來源，template review 也是要依template的完整項目review 目標document|
|

---

### CQ-05：idea 的 IDEA.md 補充 Key Features 獨立節

**影響**：idea｜**類型**：內容品質（小幅調整）

**現況**

idea 的 Key Features 目前在 BRD §5（Proposed Solution → MoSCoW 優先序中）。gendoc 的 IDEA.md 有獨立的 Key Features 節。

**第一性原則分析**

Key Features 在 IDEA.md（Layer 0）中應是最早可見的，不應等到 BRD §5 才出現。IDEA.md 代表「在決定技術方案之前，功能需求是什麼」，屬於 Layer 0 的核心資訊。

**建議**：idea 在 IDEA.md 加入獨立的 Key Features 節（≥5 項，含 Must/Should/Could 優先度）。同時保留 BRD §5 的 MoSCoW 精細化版本（兩者有不同的詳細程度）。

| 決策 | |
|------|--|
| **你的決策** | 產生邏輯都要依文件TEMPLATE要求的項目去產生，不能TEMPLATE，說的跟實際跑的不同，例如產生idea就要依template/idea.md, 產生BRD就要依template/BRD.md, 其他文件以此類推，若其他文件沒有這麼做也要改尤其在autodev, autogen裡沒有尊守此原則都要改，因為維護只在template而不是skill, 精練項目要尊從統一來源，template review 也是要依template的完整項目review 目標document|

---

### CQ-06：codebase 複製清單補充（gendoc E5）

**影響**：gendoc｜**類型**：功能完整性（小幅修正）

**現況**

gendoc 的 codebase 複製清單缺少 `README.txt`、`pom.xml`、`build.gradle`。idea 有這三個。

（注意：讀取 gendoc 原始碼後確認，`codebase_local` 路徑已有 `README.txt`、`pom.xml`、`build.gradle`；但 `codebase_git` 的複製迴圈缺少這三個檔案。）

**建議**：僅修正 `codebase_git` 的 for 迴圈，補上缺少的三個檔案。改動極小。

| 決策 | |
|------|--|
| **你的決策** | 要加|

---

## Tier 4：UX / 介面決策

---

### UX-01：idea 新增 Handoff Banner

**影響**：idea｜**類型**：UX 改善

**現況**

gendoc DOC-03-B 在移交前顯示結構化 Handoff 表格（專案名稱 / 輸入類型 / BRD 行數 / IDEA 行數 / Review 輪次 / 最終 findings）。idea 無此 Banner。

**第一性原則分析**

Handoff Banner 讓使用者在移交給 autodev 前有一個「確認點」，知道即將啟動的是什麼規模的工作。這對 idea 同樣有價值，因為 autodev（含程式碼+部署）比 autogen 代價更高。

**建議**：idea 在 Step 8（呼叫 autodev 前）加入相同格式的 Handoff Banner。

| 決策 | |
|------|--|
| **你的決策** | 要加 |

---

### UX-02：gendoc 在移交前加入 BRD 摘要展示

**影響**：gendoc｜**類型**：UX 改善

**現況**

gendoc 在 DOC-03-B 直接跳到 Handoff，無摘要展示。idea 在 Step 6 有結構化的 BRD 摘要展示（專案 / 使用者 / 痛點 / 技術 / 功能 / 研究 / 評分）。

**建議**：gendoc 在 DOC-03-B 的 Handoff Banner 前加入等效的 BRD 摘要展示（可以是 Handoff Banner 的一部分，而非獨立步驟）。

| 決策 | |
|------|--|
| **你的決策** | idea, gendoc 要有一致的顯示，可以共用這個流程，避免長期維護兩邊不一樣 |

---

### UX-03：使用者確認選項統一（gendoc DOC-03-C）

**影響**：gendoc｜**類型**：UX 改善

**現況**

gendoc 確認選項：「立即啟動 autogen」/「先查看 BRD」/「稍後手動執行」。

idea 確認選項：「開始 autodev」/「查看/修改 BRD」/「重新整理 BRD」。

差異：gendoc 無「重新整理 BRD」（回 Step 2 重新問 Q1-Q5，不重建目錄）。

**建議**

gendoc 選項改為：「開始 autogen」/「查看/修改 BRD」/「重新整理 BRD（重回 Q 澄清）」/「稍後手動執行」。

注意：「重新整理 BRD」的行為應是：不重建目錄、不重跑素材保存、不重跑研究，只重跑 Q1-Q5 → BRD 生成 → Review Loop。

| 決策 | |
|------|--|
| **你的決策** | gendoc 要跟idea一樣，可以的話共用流程, 尤其不要有什麼手動執行，我就是要串起來，不要有莫名的中斷條件|

---

### UX-04：idea 補充 Handoff State 寫入

**影響**：idea｜**類型**：功能完整性（應改）

**現況**

gendoc DOC-03-A 完整寫入 Handoff State（`brd_path` / `idea_path` / `req_dir` / `gendoc_handoff=True` / `gendoc_handoff_source`）。idea 在 Step 8 呼叫 autodev 時無明確的 Handoff State 寫入。

**第一性原則分析**

autodev 需要知道「BRD 和 IDEA.md 已存在」才能跳過自己的前置步驟（否則會重新生成 BRD）。這個資訊必須透過 state 傳遞。

**建議**：idea 在 Step 8 呼叫 autodev 前，寫入等效的 Handoff State：

```json
{
  "brd_path": "docs/BRD.md",
  "idea_path": "docs/IDEA.md",
  "req_dir": "docs/req",
  "idea_handoff": true,
  "idea_handoff_source": "devsop-idea"
}
```

起始步驟（autodev 從哪步開始）由 autodev 自己讀取 state 決定（不在 idea 內硬編碼），維持關注點分離。

| 決策 | |
|------|--|
| **你的決策** | autodev, autogen 這段要一致，但兩邊對START_STEP處理方式不同 autogen要正確轉換|

---

### UX-05：git commit 時機與命名統一

**影響**：兩者｜**類型**：工程實踐

**現況**

idea 有 `git commit` 在工作空間建立後（`chore: init`）和 BRD 生成後（`docs(devsop-idea): init BRD`）。gendoc 目前無任何 git commit。

**建議**：gendoc 補充以下 git commit 時機點：

| 時機 | commit message |
|------|---------------|
| 工作空間建立後 | `chore(devsop-gendoc): init workspace` |
| IDEA.md 生成後 | `docs(devsop-gendoc): init IDEA` |
| BRD 生成後 | `docs(devsop-gendoc): init BRD` |
| 每輪 BRD Review 修復後 | `docs(brd-review): round N fixes` |

| 決策 | |
|------|--|
| **你的決策** | 
 時機  commit message 
--------------------
 工作空間建立後  `chore(devsop-gendoc): init workspace` 
 IDEA.md 生成後 `docs(devsop-gendoc): init IDEA` 
 每輪 IDEA Review 修復後 | `docs(idea-review): round N fixes` 
 BRD 生成後  `docs(devsop-gendoc): init BRD` 
 每輪 BRD Review 修復後  `docs(brd-review): round N fixes` 

|

---

## 決策彙總表

填寫完上方各項後，在此彙總確認的修改清單：

### Tier 1 架構基礎

| 項目 | 決策 |
|------|------|
| TF-01：gendoc 工作空間創建模式 | |
| TF-02：斷點續跑狀態機設計 | |
| TF-03：Config 委派邊界 | |
| TF-04：BRD/IDEA.md 生成邏輯共用策略 | |
| TF-05：State File 命名統一 | |
| TF-06：Session Config 執行時機 | |
| TF-07：Step 命名規範 | |

### Tier 2 流程設計

| 項目 | 決策 |
|------|------|
| PD-01：idea 移除 Step -0.5 | |
| PD-02：gendoc Q1-Q5 策略 | |
| PD-03：gendoc Web Research 策略 | |
| PD-04：Loop Count 來源 | |
| PD-05：起始步驟選擇 | |
| PD-06：idea 補充 PM Expert + 衝突檢查 | |
| PD-07：舊版文件處理邏輯 | |

### Tier 3 內容品質

| 項目 | 決策 |
|------|------|
| CQ-01：BRD 章節對齊 | |
| CQ-02：IDEA.md 章節對齊 | |
| CQ-03：IDEA Quality Score | |
| CQ-04：BRD Review Loop 詳細化 | |
| CQ-05：idea IDEA.md Key Features 獨立節 | |
| CQ-06：codebase 複製清單補充 | |

### Tier 4 UX / 介面

| 項目 | 決策 |
|------|------|
| UX-01：idea 新增 Handoff Banner | |
| UX-02：gendoc 移交前 BRD 摘要展示 | |
| UX-03：使用者確認選項統一 | |
| UX-04：idea 補充 Handoff State | |
| UX-05：git commit 時機與命名 | |

---

*此文件由 MYDEVSOP 框架維護。最後更新：2026-04-22*
