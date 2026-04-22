# MYDEVSOP 共用邏輯遷移追蹤（#22）

> 追蹤 devsop-shared 標準邏輯遷移至各 SKILL 的進度。
> 原則：不動已達標項目，逐步遷移未統一部分。
> 最後更新：2026-04-19

---

## 遷移項目清單

### ✅ 已完成

| 項目 | 說明 | 完成版本 |
|------|------|---------|
| Session Config §0 | execution_mode + review_strategy 問一次全程共用 | v2.1 |
| review_strategy 讀取 | 9 個 review SKILL 統一讀取 _MAX_ROUNDS | v2.2 |
| _write_state() 原子寫入 | mktemp + mv 防 crash 截斷，devsop-autodev 已實作 | v2.3 |
| 雙層輸出格式 | [技術] + [白話說明] 並列，9 個 review SKILL | v2.4 |
| AskUserQuestion 取代 _ask() | devsop-lang-select、devsop-autodev STEP 02 | v2.4 |

---

### 🔄 進行中

| 項目 | 說明 | 目標版本 | 受影響 SKILL |
|------|------|---------|------------|
| 版本檢查邏輯統一 | 統一 check-update.sh 呼叫方式 | v2.5 | 所有入口 SKILL |
| STEP signal 格式統一 | STEP_COMPLETE / STEP_FAILED 格式 | v2.5 | 所有 STEP Agent |
| _write_state() 遷移 | idea、change、repair 使用原子寫入 | v2.5 | 3 個入口 SKILL |

---

### ⏳ 待規劃

| 項目 | 說明 | 優先度 |
|------|------|-------|
| scripts/devsop-lib.sh | 提取 Bash 函式為真實 shell 腳本，可 bats 測試 | 中 |
| state file 命名空間 | 確認 idea/change/repair 使用 user+branch 命名 | 中 |
| 三層測試覆蓋 | front-matter schema + unit + smoke test | 低 |

---

## 不遷移項目（已達標，不動）

以下項目已在各 SKILL 中正確實作，**不需遷移**：

- devsop-autodev：_spawn_agent_with_retry()、_review_loop()、STEP signal
- 9 個 review SKILL：review_strategy 讀取、Per-Round Summary 格式
- devsop-repair：Session Config、保護 PASS 原則

---

*此文件由 MYDEVSOP 開發者手動維護，反映截至目前的遷移狀態。*
