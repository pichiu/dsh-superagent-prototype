# LLM Catch-up: dsh-superagent-prototype

接手摘要，給下一個要在這個 repo 工作的 LLM。圖解版見同目錄 `superagent-explainer.html`。
分析基準：commit `6337fc8`（2026-10-31，單一匯入 commit）。

## 一句話

DeepSeek Harness（DSH）的 bundle：一個只服務一個 GitHub repo 的主動型 coding agent。它監聽 GitHub（以及選用的飛書），有權限的人請它做事時，開一張 Ticket，在獨立的 Executor Session 與目錄裡完成，再回到原本的對話交付（通常是一個 PR）。核心原則：**盡量少佔用人的注意力，盡量多給價值**（docs/000）。

## 基線（實測）

- `pnpm install --frozen-lockfile` 可直接安裝，所有 `@deepseek-ai/*` 依賴都在 npm 上。
- `pnpm typecheck` 通過；`pnpm test`：59 個 test file、1,197 個測試全過，約 25 秒。
- `pnpm check` = typecheck + test + build（tsdown）。
- 規模：src 約 15.6k 行（app 5.3k、client 4.4k、core 2.1k、sources 2.1k、executor 0.8k、infra 0.3k、頂層 0.5k）；tests 約 15.0k 行；docs 約 2.0k 行；prompts 438 行。

## 閱讀順序

1. `docs/000-philosophy.md`：所有取捨的判準。
2. `docs/001-domain.md`：概念、規則、**被拒絕的概念**、中英詞彙表。
3. `docs/002-authorization.md` + `src/policy.ts`（`settingsPolicy().gate()`、`revalidate()`）。
4. `docs/003-execution.md` + `src/core/stage.ts`。
5. `docs/007-architecture.md`：程式碼索引（Engine 方法表、模組表、測試表、SQLite 表、工具表）。
6. `src/core/engine.ts` → `intake.ts` → `evals.ts` → `tickets.ts` → `assignments.ts` → `replies.ts`。
7. 需要時：`docs/004`（prompts）、`005`（panel）、`006`（部署）、`008`（新增 Source 的 cookbook）。

編號文件只引用編號更小的文件（測試會檢查）。

## 心智模型

```
Source ──Trigger──▶ Intake ──▶ Thread
                       │           ├─ 已有 Ticket 訂閱 → fan-out 給 Worker（人的訊息不進 Eval）
                       │           └─ 排入 Eval
                       ▼
             Gate（確定性，policy.ts）→ explicit / inferred / 不准入
                       ▼
             Main Session = Router（LLM，只路由，orchestrator_route）
                       ▼ accept
             Ticket + Mandate + Subscriptions + Assignment（同一個交易）
                       ▼ effect: launch
             Executor Session "worker-<ticket id>" + 目錄（prepare 指令，通常是 git worktree）
                       ▼ orchestrator_ack / orchestrator_report
             Outbox → Channel → 回到原對話
```

- 文章中的 Router = 程式碼的 Main Session（`orchestrator`）；Worker = Executor Session（`worker`）。
- Accept mode：`off` / `observe`（預設，正面評估只記成 ignore「would accept」）/ `auto`。explicit 請求在任何 mode 都有效。
- Ticket stage（first match wins）：ended → person/delivery → person/answer → person/review（ready）→ person/blocked·failed（之後沒有人留言）→ working/queued → idle → working。
- 結束的 Ticket 保留 Mandate 和訂閱；不是人叫停的，人在其 Thread 留言就直接 resume（先 revalidate）。

## 不變式（不要破壞）

- `Engine` 是 Trigger、Thread、Decision、Ticket、Assignment 唯一的寫入者。方法全部同步，一個指令一個交易。
- `src/core` 不做 I/O，只 import 自己與 Node hashing。外部動作在交易內 `enqueue` 成 effect，commit 後才執行。
- `src/sources` 不 import `src/app`。面板只 import 型別與少數純值模組。
- 每種狀態只有一個寫入者（Settings→Settings、Prompts→Prompts、Effects→Effects、Outbox receipts→Outbox、Source meta→該 Source）。
- 呼叫者只拿自己角色的 Engine `Pick`：`EngineForSources`、`EngineForMainSession`、`EngineForPeople`、`EngineForSettings`、`EngineForExecutor`。
- 命名：要求用祈使句（`stopTicket`），可能被拒絕；事實用過去式（`assignmentStarted`）。
- 可由人或 agent 觸發的拒絕一律是 `UserError(code)`（`src/core/errors.ts`）。
- Mandate 只有三種改變：accept 建立、人的 stop 撤回、revalidation 失敗撤銷。Policy 變更**不**取消 live Ticket，只送 policy-changed Input。
- Subscription 只帶上下文、不帶授權；Correlation 只加 context、不路由。
- 外部文字（留言、引用、repo 檔案）是資料，不能改權限。不跨 Source 對應身分。
- Trigger id、Thread key、Channel 位址會持久化，改格式會斷掉舊資料。`SCHEMA_VERSION` 只增不減。
- 不新增領域概念：新需求先變成既有概念的屬性，或一段 prompt 文字。

## 會擋住你的測試

| 測試 | 擋什麼 |
| --- | --- |
| `tests/docs.spec.ts` | `docs/007` 的 Modules 表與 Tests 表必須列出每個 src 和 test 檔；文件連結必須能解析；編號文件只能引用更早的 |
| `tests/class-docs.spec.ts` | `src` 中每個 class 前都要有 JSDoc |
| `tests/model-surface.spec.ts` | 模型看得到的工具、prompt、skill、input 與 schema 一致，並且每個角色有字數預算 |
| `tests/cards.spec.ts` | 內建 prompt 文字必須陳述的規則，逐檔檢查 |
| `tests/locales.spec.ts` | 面板文案中英都要有 |
| `tests/store.spec.ts` | `KV` 的每個 key 只在 `src/kv.ts` 與 migration 出現 |

## 常見改動

| 想做的事 | 程式碼 | 測試 | 文件 |
| --- | --- | --- | --- |
| 授權 / gate | `src/policy.ts`、`src/admission.ts` | `policy.spec` | docs/002 |
| Ticket stage / 結束條件 | `src/core/stage.ts`、`src/core/tickets.ts` | `stage.spec`、`core-engine.spec` | docs/003 |
| 新 Source | `src/sources/<kind>.ts`、`tsdown.config.ts`、`package.json` exports、`cordis.patch.yml` | `sources.spec`、`registry.spec` | docs/008 全部步驟 + checklist；docs/007 模組表 |
| Agent 行為文字 | `prompts/*.md`（一行一句）、`skills/` | `cards.spec`、`model-surface.spec`、`prompts.spec` | docs/004 |
| 新工具 | `src/app/tools/main.ts` / `executor.ts` | `tools.spec`、`model-surface.spec` | docs/007 工具表 |
| 新 effect kind | `src/app/effects.ts` + handler | `effects.spec`、`queue.spec` | docs/007 Effects 表 |
| 儲存形狀 | `src/app/store.ts`、`src/app/migration.ts`、`src/kv.ts` | `store.spec`、`migration.spec` | docs/007 Persistence |
| 面板 | `src/client/*`、`locales.ts` | `panel-*.spec.tsx` | docs/005 |

Domain 規則可以用 `src/core/memory.ts` 的 in-memory repositories 測；整合測試用 `tests/fixtures/host.ts` 模擬 DSH host。

## 文章與原型的差距

依 repo 搜尋結果：

- 已實作：Router–Worker 雙層、事件驅動 Trigger、Gate + Assessment 兩道篩網、後續訊息直達 Worker、持久化 effect 佇列與重啟恢復。
- 部分：主動行動（需團隊寫 rule，預設 observe）、背景非同步（worker 背景 job/subagent，30 分鐘上限）、存算分離（狀態在 SQLite，但沒有遠端 daemon）。
- 找不到：Watchdog、chained recall / 長程記憶。
- 已知缺口（docs/007）：沒有資料清理，trigger、decision、outbox、各版本設定會無限增長；Trigger 欄位的信任邊界與部署隔離仍是開放問題。

## 接手開場 prompt

```
你要接手 dsh-superagent-prototype（TypeScript，pnpm）。
先讀 analysis/LLM-CATCHUP.md、docs/000、docs/001，再讀與任務相關的 docs/002–004，用 docs/007 當程式碼索引。
遵守 LLM-CATCHUP.md 的不變式。新增或改名 src/tests 檔案要同步 docs/007 的表格。收尾跑 pnpm check。
任務：<在這裡寫>。先回報會動哪些檔案、哪些文件要同步，再動手。
```
