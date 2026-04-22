# MYDEVSOP 實施工作清單 v2.0

> **版本**：v2.0 — 2026-04-22（重寫修正 v1.0 的根本錯誤）
> **來源**：modification-decisions.md 所有決策

---

## 核心架構原則（所有修改必須遵守）

```
templates/*.md        → 唯一真理，改這裡等於改所有 skill 的輸出
skills/devsop-gen-*   → 執行時 Read templates，依結構產生文件，永不硬編碼章節
skills/devsop-*-review→ 執行時 Read templates，對照現有文件做 review
skills/idea/gendoc/*  → 只做流程編排：何時呼叫哪個 gen-* / review-* skill
```

**改 template → 不需改任何 skill。改 skill → 只改流程邏輯，不改文件結構。**

---

## 現有基礎架構盤點（已存在，不需建立）

| 類型 | 現有項目 |
|------|---------|
| **Templates**（已有 14 個） | `templates/IDEA.md`, `BRD.md`, `PRD.md`, `PDD.md`, `EDD.md`, `API.md`, `ARCH.md`, `SCHEMA.md`, `BDD.md`, `RTM.md`, `test-plan.md`, `README.md`, `LOCAL_DEPLOY.md`, `UML-CLASS-GUIDE.md` |
| **共用** | `skills/devsop-shared/SKILL.md`（§-1 版本檢查、§0 Session Config、§1 State 命名）|
| **文件生成 skills**（已有 15 個） | `devsop-gen-brd`, `devsop-gen-prd`, `devsop-gen-pdd`, `devsop-gen-edd`, `devsop-gen-arch`, `devsop-gen-api`, `devsop-gen-schema`, `devsop-gen-bdd`, `devsop-gen-test-plan`, `devsop-gen-readme`, `devsop-gen-diagrams`, `devsop-gen-html`, `devsop-gen-k8s`, `devsop-gen-cicd`, `devsop-gen-secrets` |
| **文件 Review skills**（已有 8 個）| `devsop-brd-review`, `devsop-prd-review`, `devsop-pdd-review`, `devsop-edd-review`, `devsop-arch-review`, `devsop-api-review`, `devsop-test-review`, `devsop-product-review` |

---

## 真正的缺口與需修正項（新發現）

| # | 缺口 | 動作 |
|---|------|------|
| N-1 | `devsop-gen-idea` 不存在 — IDEA.md 生成無獨立 skill | **新建** |
| N-2 | `devsop-idea-review` 不存在 — IDEA.md review loop 無 skill | **新建** |
| N-3 | `devsop-shared §0` 讀取固定名 `.devsop-state.json`（違反 TF-05）| **更新** |
| N-4 | `devsop-shared §1` 建立 symlink（違反 TF-05，與 `.devsop-state-{user}-{branch}.json` 矛盾）| **移除** |
| N-5 | 所有 `gen-*` / `review-*` skills 讀取固定名 `.devsop-state.json` | **批次修正** |
| N-6 | `autogen` 缺 autodev↔autogen 步驟對照表（導致 `start_step` 無法跨 skill 共用）| **新增** |
| N-7 | `idea` 和 `gendoc` 的文件生成用 inline prompt，未委派給 gen-* skills | **改為 Skill tool 呼叫** |
| N-8 | `idea` 和 `gendoc` 都缺 IDEA.md Review Loop（違背文件階層設計）| **兩者均需補** |
| N-9 | `idea` 和 `gendoc` 都沒有對的順序要先 IDEA.md IDEA Review Loop，才有BRD.md BRD review loop（違背文件階層設計）| **兩者均需補** |

---

## Part A：基礎架構更新（所有 Part 前提，先完成）

---

### A-01：新建 `devsop-gen-idea` skill

**目標檔案**：`skills/devsop-gen-idea/SKILL.md`（新建）
**來源決策**：TF-04（A）、N-1

**模式**：與 `devsop-gen-brd` 完全相同的框架：
- Step -1：版本檢查（同 devsop-shared §-1）
- Step 0：讀取 state（動態名 `$_STATE_FILE`）
- Step 1：`Read templates/IDEA.md` → 了解期望結構（**不在 skill 裡列章節，template 改了自動跟上**）
- Step 2：依 template 結構 + state 的 Q1-Q5 + `_RESEARCH_SUMMARY` + 輸入素材 → 生成 `docs/IDEA.md`
- Step 3：state 寫入 `idea_generated: true`，`completed_steps` 追加

---

### A-02：新建 `devsop-idea-review` skill

**目標檔案**：`skills/devsop-idea-review/SKILL.md`（新建）
**來源決策**：TF-02 user 備註、CQ-03、N-2

**模式**：與 `devsop-brd-review` 完全相同的框架：
- Step 1：`Read templates/IDEA.md` → 取得期望章節（動態，改 template 自動適應）
- Step 2：`Read docs/IDEA.md` → 對照每個章節是否存在且完整
- 輪次上限：從 state 的 `review_strategy` 換算（rapid=1 / standard=2 / thorough=3 / exhaustive=5）
- 每輪：輸出 REVIEW_JSON + FINDINGS_TREND + Per-Round Summary
- 每輪修復後：`git commit -m "docs(idea-review): round N fixes"`
- 通過後：state 寫入 `idea_review_passed: true`

---

### A-03：更新 `devsop-shared/SKILL.md`

**目標檔案**：`skills/devsop-shared/SKILL.md`（更新，不是新建）
**來源決策**：TF-05、N-3、N-4

**具體動作**：

| 位置 | 修改內容 |
|------|---------|
| **§0 Session Config** | 把所有 `'.devsop-state.json'` 固定名改為 `'$_STATE_FILE'`。各 skill 在呼叫前自行設好 `$_STATE_FILE` 變數 |
| **§1 State File 命名** | 移除 `ln -sf "$_STATE_FILE" ".devsop-state.json"` 整行。只保留動態命名邏輯，格式統一為 `.devsop-state-{git_user}-{branch}.json` |
| **新增 §HANDOFF-DISPLAY** | 統一的 Handoff Banner + BRD 摘要展示邏輯（idea 和 gendoc 共用，呼叫時傳入 project_name / input_type / BRD 行數 / IDEA 行數 / review 輪次 / 最終 findings 數）|
| **新增 §STATE-SCHEMA** | 記錄所有 skill 共用的標準 state key（見 A-04）|

---

### A-04：定義 State Schema 統一 key（寫入 devsop-shared §STATE-SCHEMA）

**來源決策**：TF-02、TF-03、TF-05

| Key | 類型 | 說明 |
|-----|------|------|
| `skill_source` | string | `devsop-idea` / `devsop-gendoc`（防止跨 skill 誤用 state）|
| `execution_mode` | string | `interactive` / `full-auto` |
| `review_strategy` | string | `rapid` / `standard` / `thorough` / `exhaustive` |
| `project_name` | string | 英文小寫連字號 |
| `project_type` | string | PM Expert 提取 |
| `input_type` | string | 7 種輸入類型之一 |
| `input_source` | string | URL 或路徑 |
| `input_summary` | string | 摘要文字 |
| `req_dir` | string | `docs/req` |
| `completed_steps` | array | 已完成的 STEP 名稱清單 |
| `idea_generated` | bool | IDEA.md 已生成 |
| `idea_review_passed` | bool | IDEA Review Loop 已通過 |
| `brd_generated` | bool | BRD.md 已生成 |
| `brd_review_passed` | bool | BRD Review Loop 已通過 |
| `handoff` | bool | 已移交下游 skill |
| `brd_path` | string | `docs/BRD.md` |
| `idea_path` | string | `docs/IDEA.md` |
| `handoff_source` | string | `devsop-idea` / `devsop-gendoc` |
| `start_step` | int | autodev 31 步編號（autogen 透過對照表翻譯）|

---

### A-05：批次修正 gen-* / review-* skills 的 state file 讀取名稱

**目標**：所有 gen-* 和 review-* skills
**來源決策**：TF-05、N-5

**具體動作**：
- 搜尋所有 `skills/devsop-gen-*/SKILL.md` 和 `skills/devsop-*-review/SKILL.md`
- 找出 `'.devsop-state.json'` 固定名稱
- 替換為動態偵測邏輯：
  ```bash
  _STATE_FILE=$(ls .devsop-state-*.json 2>/dev/null | head -1 || echo ".devsop-state.json")
  ```
  （fallback 維持向後相容）

---

### A-06：在 autogen 建立 autodev↔autogen 步驟對照表

**目標檔案**：`skills/devsop-autogen/SKILL.md` — 文件最前端新增 `§STEP-MAPPING`
**來源決策**：TF-03、TF-07

**說明**：讓 `start_step`（autodev 31 步格式）可跨兩個 skill 共用。autogen 自行翻譯。

| autodev STEP | 說明 | autogen 文件 STEP |
|-------------|------|-----------------|
| STEP-01 | Workspace + State | 無（前置）|
| STEP-02 | PM Expert / Project Init | 無（前置）|
| STEP-03 | BRD | `autogen-BRD` |
| STEP-04 | PRD | `autogen-PRD` |
| STEP-05 | PDD | `autogen-PDD` |
| STEP-06 | EDD | `autogen-EDD` |
| STEP-07 | SDD | `autogen-SDD` |
| STEP-08 | API Design | `autogen-API` |
| STEP-09 | Schema | `autogen-SCHEMA` |
| STEP-10 | Test Plan | `autogen-TESTPLAN` |
| STEP-11 | BDD Features | `autogen-BDD` |
| STEP-12 | RTM | `autogen-RTM` |
| STEP-13+ | Code / Deploy / Test | 不在 autogen 範圍 → 找 mapping 表中前一個 autogen STEP |

**翻譯邏輯**：若 `start_step` 指向非文件 STEP（STEP-01、02、13+），往前找最近的文件 STEP 開始。

---

## Part B：devsop-idea 修改清單

---

### B-01：移除 Step -0.5（輸入模式選單）

**目標**：`~/.claude/skills/devsop-idea/SKILL.md`
**來源決策**：PD-01

- 刪除整個 `## Step -0.5` 區塊
- 刪除 `_QUICK_START` / `_FULL_AUTO_INPUT` 旗標所有引用
- Step 2 澄清提問的執行條件改為純粹依 `execution_mode`：`full-auto` → 跳過；`interactive` → 執行

---

### B-02：移除 Step 1.5 的 symlink

**目標**：`~/.claude/skills/devsop-idea/SKILL.md`
**來源決策**：TF-05

- 刪除 `ln -sf "$_STATE_FILE" ".devsop-state.json"` 這行
- 所有引用 `.devsop-state.json` 固定名稱的地方改為引用 `$_STATE_FILE`

---

### B-03：在 Step 1.5 加入 TF-02 守衛邏輯

**目標**：`~/.claude/skills/devsop-idea/SKILL.md` — `mkdir` 之前插入
**來源決策**：TF-01（A）、TF-02（A）

```
if $_STATE_FILE 不存在：
  → 全新建路徑
  → 驗證 $_PROJECT_DIR 不存在
  → 驗證 GitHub repo 不存在（gh api repos/$_GH_OWNER/$_PROJECT_NAME）
  → full-auto 衝突時：自動加日期後綴
  → interactive 衝突時：AskUserQuestion（加後綴 / 指定新名 / 中止）
  → 通過後才執行 mkdir

elif $_STATE_FILE 存在：
  → 讀取 state["skill_source"]
  → 若非 "devsop-idea" → ABORT（說明衝突原因）
  → 讀取 completed_steps → 找最後完成 STEP
  → 驗證最後 STEP 對應的文件是否存在（不做內容細節比對，交給對應的 gen-* skill 負責）
  → 若文件不存在 → 刪除該 STEP 記錄，重做該 STEP
  → 從下一 STEP 繼續
```

---

### B-04：在 Session Config 後加入 PM Expert Agent

**目標**：`~/.claude/skills/devsop-idea/SKILL.md` — Session Config 之後
**來源決策**：PD-06

移植 gendoc DOC-01 的完整 PM Expert Agent 邏輯：
- PM Expert 分析 `_INPUT_SUMMARY`，提取 `project_name` / `project_type` / `tech_stack_hints` / `key_features` / `target_users` → 寫入 state
- 寫入 `skill_source: "devsop-idea"` → state

---

### B-05：IDEA.md 生成改為呼叫 Skill tool `devsop-gen-idea`

**目標**：`~/.claude/skills/devsop-idea/SKILL.md` — IDEA.md 生成步驟
**來源決策**：TF-04（A）、N-7

- 移除現有 inline IDEA.md 生成 prompt（無論多少行）
- 改為：透過 Skill tool 呼叫 `/devsop-gen-idea`
- `devsop-gen-idea` 讀取 `templates/IDEA.md`，template 改了 skill 自動適應，**idea 不需做任何修改**

---

### B-06：新增 IDEA.md Review Loop → 呼叫 `devsop-idea-review`

**目標**：`~/.claude/skills/devsop-idea/SKILL.md` — IDEA.md 生成後、BRD 前
**來源決策**：TF-02 user 備註、CQ-03、N-8

新增步驟，透過 Skill tool 呼叫 `/devsop-idea-review`：
- 輪次由 state 的 `review_strategy` 換算
- 通過後寫 state `idea_review_passed: true`

---

### B-07：BRD 生成確認/改為呼叫 `devsop-gen-brd`

**目標**：`~/.claude/skills/devsop-idea/SKILL.md`
**來源決策**：TF-04（A）

確認 idea 已透過 Skill tool 呼叫 `/devsop-gen-brd`。若有 inline BRD 生成 prompt 則移除，改為呼叫。

---

### B-08：BRD Review Loop 確認/改呼叫 `devsop-brd-review`

**目標**：`~/.claude/skills/devsop-idea/SKILL.md`

確認 BRD Review 已透過 Skill tool 呼叫 `/devsop-brd-review`。若有 inline review 邏輯則移除，改為呼叫。

---

### B-09：Handoff Banner 改呼叫 devsop-shared §HANDOFF-DISPLAY

**目標**：`~/.claude/skills/devsop-idea/SKILL.md` — 呼叫 autodev 前
**來源決策**：UX-01、UX-02

移除現有 Handoff Banner inline 顯示，改為引用 devsop-shared §HANDOFF-DISPLAY 的邏輯（與 gendoc 共用）。

---

### B-10：補充 Handoff State 完整寫入

**目標**：`~/.claude/skills/devsop-idea/SKILL.md` — 呼叫 autodev 前
**來源決策**：UX-04

寫入 state：
```json
{
  "handoff": true,
  "brd_path": "docs/BRD.md",
  "idea_path": "docs/IDEA.md",
  "req_dir": "docs/req",
  "handoff_source": "devsop-idea"
}
```
`start_step` 由 autodev 讀 state 自行決定，不在 idea 內硬編碼。

---

### B-11：使用者確認選項統一

**目標**：`~/.claude/skills/devsop-idea/SKILL.md` — 確認段落
**來源決策**：UX-03

選項改為（移除「稍後手動執行」）：
1. 開始 autodev
2. 查看/修改 BRD
3. 重新整理 BRD（重回 Q1-Q5，不重建目錄）

---

### B-12：PD-07 舊版文件歸檔邏輯

**目標**：`~/.claude/skills/devsop-idea/SKILL.md` — 各文件生成步驟前
**來源決策**：PD-07

```
if 目標文件存在 AND state 記錄此文件為「已生成」:
    mv 舊版 → docs/req/old-{filename}-{timestamp}.md
    重新呼叫 gen-* skill 生成新版

elif 目標文件存在 AND state 無此文件記錄:
    interactive → AskUserQuestion 如何處理
    full-auto   → mv 到 docs/req/old-{filename}-{timestamp}.md → 重新呼叫 gen-* skill
```

---

### B-13：每個 STEP 完成後寫入 completed_steps + git commit

**目標**：`~/.claude/skills/devsop-idea/SKILL.md`
**來源決策**：TF-02、UX-05

| 時機 | git commit message |
|------|-------------------|
| 工作空間建立後 | `chore(devsop-idea): init workspace` |
| IDEA.md 生成後（gen-idea 返回後）| `docs(devsop-idea): init IDEA` + state `idea_generated: true` |
| IDEA Review 每輪 | `docs(idea-review): round N fixes` |
| IDEA Review 通過 | state `idea_review_passed: true` |
| BRD 生成後（gen-brd 返回後）| `docs(devsop-idea): init BRD` + state `brd_generated: true` |
| BRD Review 每輪 | `docs(brd-review): round N fixes` |
| BRD Review 通過 | state `brd_review_passed: true` |

---

## Part C：devsop-gendoc 修改清單

---

### C-01：前段架構重寫（新增工作空間建立流程）

**目標**：`~/.claude/skills/devsop-gendoc/SKILL.md` — Step 0 之前新增前置步驟
**來源決策**：TF-01（A）、TF-06（A）

新增（對齊 idea Step 0 + Step 1 + Step 1.5）：

```
§PRE-1：輸入來源偵測（同 idea Step 0-A/0-B：偵測 _INPUT_TYPE，遠端來源立即 WebFetch）

§PRE-2：專案目錄偵測（同 idea Step 1）
  - 偵測慣用根目錄（~/projects / ~/workspace / ~/code）
  - interactive → AskUserQuestion 3 選項
  - full-auto   → $(pwd)/$_PROJECT_SLUG

§PRE-3：工作空間建立 + Session Config（同 idea Step 1.5）
  - mkdir -p $_PROJECT_DIR/docs && cd $_PROJECT_DIR
  - git init
  - _STATE_FILE=".devsop-state-${_GIT_USER}-${_GIT_BRANCH}.json"（不要 symlink）
  - 寫入 skill_source: "devsop-gendoc" → state
  - Session Config（execution_mode + review_strategy）

§PRE-4：TF-02 守衛（同 B-03，但 skill_source 改為 "devsop-gendoc"）
```

---

### C-02：移除 Step 0-B（gendoc_loop_count）

**來源決策**：PD-04、TF-03

- 刪除整個 `## Step 0-B` 區塊
- 刪除所有 `gendoc_loop_count` 引用
- Review Loop 的 `max_rounds` 改為從 state `review_strategy` 換算（rapid=1 / standard=2 / thorough=3 / exhaustive=5）

---

### C-03：移除 Step 0-C（起始步驟詢問）

**來源決策**：PD-05、TF-02

- 刪除整個 `## Step 0-C` 區塊
- 刪除所有 `gendoc_start_step` 引用
- 起始步驟由 TF-02 狀態機自動判斷（讀 `completed_steps`）

---

### C-04：移除 Step 0-E（skip/overwrite/abort），改為 TF-02 守衛

**來源決策**：TF-02、PD-07

- 刪除 Step 0-E 整個區塊
- 守衛邏輯已在 C-01 §PRE-4 實施
- 文件覆蓋改為 C-16 的 REQ 歸檔邏輯

---

### C-05：State File 命名修正

**來源決策**：TF-05

全文搜尋 `.devsop-state.json` 固定名稱 → 改為引用 `$_STATE_FILE`（已在 C-01 §PRE-3 初始化）

---

### C-06：Session Config 移至工作空間建立後

**來源決策**：TF-06

- 從 Step 0 刪除 Session Config 段落
- Session Config 已在 C-01 §PRE-3 中實施

---

### C-07：補充 Q1-Q5 澄清步驟

**修改位置**：素材保存（Step 0-D）之後，IDEA.md 生成前
**來源決策**：PD-02（B）

新增 `## Step Q：需求澄清（Q1-Q5）`，與 idea Step 2 完全相同：
- Q1 主要使用者、Q2 核心痛點、Q3 技術限制、Q4 使用規模、Q5 補充
- full-auto → 跳過，AI 自動推斷
- **所有輸入類型均執行**（素材是參考資料，不是答案）

---

### C-08：補充 Web Research 步驟（3 次 WebSearch）

**修改位置**：Q1-Q5 澄清後，IDEA.md 生成前
**來源決策**：PD-03（B）

新增 `## Step R：網路背景研究`，與 idea Step 3 完全相同：
- 3 次 WebSearch（競品+開源 / 技術最佳實踐 / 已知挑戰）
- 整合為 `_RESEARCH_SUMMARY`（注入後續 IDEA.md §8 + BRD §0）
- **所有輸入類型均執行**

---

### C-09：IDEA.md 生成改為呼叫 Skill tool `devsop-gen-idea`

**來源決策**：TF-04（A）、N-7

- 移除現有 DOC-01.5 inline 生成 prompt
- 改為透過 Skill tool 呼叫 `/devsop-gen-idea`
- 與 B-05 相同邏輯，template 改了自動適應

---

### C-10：新增 IDEA.md Review Loop → 呼叫 `devsop-idea-review`

**修改位置**：DOC-01.5 後，DOC-02 前
**來源決策**：TF-02 user 備註、N-8

新增 `## DOC-01.6：IDEA.md Review Loop`，透過 Skill tool 呼叫 `/devsop-idea-review`

---

### C-11：BRD 生成確認/改為呼叫 Skill tool `devsop-gen-brd`

**來源決策**：TF-04（A）

確認 gendoc DOC-02 已透過 Skill tool 呼叫 `/devsop-gen-brd`。若有 inline BRD 生成 prompt 則移除，改為呼叫。

---

### C-12：BRD Review Loop 確認/改呼叫 `devsop-brd-review`

確認 BRD Review 已透過 Skill tool 呼叫 `/devsop-brd-review`。若有 inline review 邏輯則移除。

---

### C-13：修正 codebase_git 複製清單

**修改位置**：`codebase_git` 的 for 迴圈
**來源決策**：CQ-06

補充缺少的三個檔案：
```bash
for _FILE in README.md README.rst README.txt CONTRIBUTING.md \
             package.json pyproject.toml go.mod Cargo.toml \
             pom.xml build.gradle; do
```

---

### C-14：Handoff Display 改引用 devsop-shared §HANDOFF-DISPLAY

**來源決策**：UX-02、UX-01

移除現有 Handoff Banner inline 顯示邏輯，改為引用 `devsop-shared §HANDOFF-DISPLAY`（與 idea B-09 共用同一段邏輯）。

---

### C-15：使用者確認選項統一

**來源決策**：UX-03

選項改為（與 idea B-11 對齊，移除「稍後手動執行」）：
1. 開始 autogen
2. 查看/修改 BRD
3. 重新整理 BRD（重回 Q1-Q5，不重建目錄）

---

### C-16：PD-07 舊版文件歸檔邏輯（gendoc 版）

**來源決策**：PD-07

與 B-12 相同邏輯。

---

### C-17：每個 STEP 完成後寫入 completed_steps + git commit

**來源決策**：TF-02、UX-05

| 時機 | git commit message |
|------|-------------------|
| 工作空間建立後 | `chore(devsop-gendoc): init workspace` |
| IDEA.md 生成後 | `docs(devsop-gendoc): init IDEA` + state `idea_generated: true` |
| IDEA Review 每輪 | `docs(idea-review): round N fixes` |
| IDEA Review 通過 | state `idea_review_passed: true` |
| BRD 生成後 | `docs(devsop-gendoc): init BRD` + state `brd_generated: true` |
| BRD Review 每輪 | `docs(brd-review): round N fixes` |
| BRD Review 通過 | state `brd_review_passed: true` |

---

## Part D：autogen 修改清單

---

### D-01：新增 autodev↔autogen 步驟對照表（已在 A-06 定義）

**目標**：`skills/devsop-autogen/SKILL.md` — 文件最前端新增 `§STEP-MAPPING`

詳見 A-06 的對照表。讀取 state 的 `start_step` → 翻譯為 autogen STEP → 開始執行。

---

### D-02：修正 Handoff State 讀取

**來源決策**：UX-04

- 讀取 state `handoff_source`（`devsop-idea` 或 `devsop-gendoc`）
- 若 `brd_review_passed=true` → 跳過 BRD 生成步驟
- 若 `idea_review_passed=true` → 跳過 IDEA.md 生成步驟

---

### D-03：確認所有文件生成呼叫 gen-* skills

**來源決策**：TF-04（A）

確認 autogen 已透過 Skill tool 呼叫對應的 gen-* skill（`/devsop-gen-prd`、`/devsop-gen-pdd` 等）。若有 inline 生成邏輯則移除，改為呼叫。**template 改了，gen-* skill 會自動適應，autogen 不需修改。**

---

## Part E：autodev 修改清單

---

### E-01：確認所有文件生成呼叫 gen-* skills

**來源決策**：TF-04（A）

與 D-03 相同。確認 autodev 文件生成步驟已呼叫 gen-* skills，若有 inline 生成邏輯則移除。

---

### E-02：補充 IDEA.md Review Loop

**來源決策**：TF-02 user 備註、N-8

autodev 的 IDEA.md 生成後，透過 Skill tool 呼叫 `/devsop-idea-review`（與 B-06、C-10 相同）。

---

### E-03：State file 命名確認

**來源決策**：TF-05

確認 autodev 使用動態命名 `.devsop-state-{git_user}-{branch}.json`，無 symlink，無固定名稱讀取。若有則修正（同 A-05）。

---

## 實施路徑

| 優先序 | 工作項目 | 前提條件 |
|--------|---------|---------|
| **P0（基礎）** | A-01, A-02, A-03, A-04, A-05, A-06 | 無 |
| **P1（idea 核心）** | B-01, B-02, B-03, B-04 | A-03, A-04 |
| **P1（gendoc 核心）** | C-01, C-02, C-03, C-04, C-05, C-06 | A-03, A-04 |
| **P2（文件生成委派）** | B-05, B-07, C-09, C-11, D-03, E-01 | A-01（gen-idea 存在後 B-05/C-09 才能呼叫）|
| **P2（Review Loop）** | B-06, B-08, C-10, C-12, E-02 | A-02（idea-review 存在後才能呼叫）|
| **P3（流程補充）** | C-07, C-08, B-12, C-16 | P1 完成 |
| **P3（UX 對齊）** | B-09, B-11, C-14, C-15 | A-03（HANDOFF-DISPLAY 存在後）|
| **P4（下游 skill）** | D-01, D-02 | A-06 |
| **P5（收尾）** | B-13, C-13, C-17, E-03 | P1, P2, P3 |

---

*此文件由 MYDEVSOP 框架維護。最後更新：2026-04-22*
