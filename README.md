# npo-workbench

給台灣非營利組織用的行銷 AI 工作台。多租戶 SaaS。

協會在系統裡建一次「協會腦」（全名、勸募字號、核准事實、紅線、品牌語氣），
之後所有 AI 產出都自動帶入這些資料，並且**必須通過合規守門才能交付**。

> 這個產品的核心不是「生成」，是「守門」。
> 生成品質好是基本盤，守門守得住才是產品價值。

---

## 現況

**尚未開始實作。** 本 repo 目前只有文件，程式碼從 P1 的 T1 開始（見 `CLAUDE.md` 的任務清單）。

| Phase | 範圍 | 狀態 |
|---|---|---|
| P1 | 骨架 + 協會腦 + M3 內容 + Guardrail G1/G5 | 未開始 |
| P2 | M1 健診 + M4 文案 + 產出歷史 + 額度 | — |
| P3 | M2 年度規劃 + M6 儀表板 | — |
| P4 | M5 生圖 + M7 銷售頁 | — |
| P5 | 多使用者權限、顧問視角、白標 | — |

P1 是唯一有硬性驗收的一期：用 TBCA 的資料生出一則 Threads 貼文，且守門報告全綠。

---

## 文件

| 檔案 | 內容 |
|---|---|
| [`CLAUDE.md`](./CLAUDE.md) | **開發契約**。硬規則、目錄結構、P1 任務清單與驗收條件。開工前先讀這份。 |
| [`docs/architecture.md`](./docs/architecture.md) | 系統架構 v1.0。分層、技術選型、完整資料模型、Generator 抽象、Guardrail Pipeline、分期。 |
| [`docs/service-architecture.md`](./docs/service-architecture.md) | 服務架構。四層服務、三方分工、服務年曆、方案分級、續約鉤子。主詞是人，不是系統。 |
| [`docs/functional-architecture.md`](./docs/functional-architecture.md) | 系統功能架構。功能全景圖、六個功能層、七個模組的規格與依賴、功能 × 分期 × 角色對照。 |
| [`docs/module-reference.md`](./docs/module-reference.md) | **模組對照表**。每個模組解決什麼問題、吃什麼資料、靠哪支 skill 與哪些外部規範。含資料流與參考資料索引。 |
| [`docs/module-functions.md`](./docs/module-functions.md) | **七個模組的功能清單**。共用形狀、每個模組能做什麼、功能總表。 |
| [`docs/m7-landing-page.md`](./docs/m7-landing-page.md) | M7 募款銷售頁規格 v1.0。十二拍敘事結構、12 種區塊 schema、進度條的合規改法。 |
| [`docs/skills-inventory.md`](./docs/skills-inventory.md) | Skill 統整清單 v1.0。對話層 Skill 與系統層 Prompt Pack 的分工、盤點、五支新建 skill 規格。 |
| [`docs/gap-analysis.md`](./docs/gap-analysis.md) | 系統自檢：目前設計缺什麼。上游（事實庫冷啟動）與下游（產出動線）兩頭都是空的。 |
| [`docs/open-questions.md`](./docs/open-questions.md) | 交叉比對兩份文件後尚未收斂的決定，動工前要先拍板。 |

---

## 技術棧

Next.js 15（App Router）· TypeScript strict · Tailwind + shadcn/ui ·
Supabase（Postgres / Auth / Storage / RLS）· Drizzle ORM · Anthropic SDK（**server only**）· Zod · Vercel

**刻意不用**：微服務、Redis、Kafka、自建 auth、GraphQL。單人專案，複雜度就是敵人。

---

## 快速開始

專案尚未初始化。第一步是 `CLAUDE.md` 任務清單的 T1：

```bash
npx create-next-app@latest . --typescript --tailwind --app --eslint
```

之後每個 task 完成都要跑：

```bash
npm run typecheck && npm run lint
```

---

## 環境變數

複製 `.env.example` 成 `.env.local` 後填值。變數集中在 `src/lib/env.ts` 以 Zod 驗證，缺變數就啟動失敗。

```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
ANTHROPIC_API_KEY=
RESEND_API_KEY=
```

`ANTHROPIC_API_KEY` 與 `SUPABASE_SERVICE_ROLE_KEY` **只能存在 server 端**。
任何 `"use client"` 檔案裡出現 `@anthropic-ai/sdk` 就是嚴重錯誤。

---

## 三條最容易寫錯的規則

1. **RLS**：除 `users` 外每張表都要有 `org_id` 與 RLS policy。沒有 RLS 的表不准 merge——這條寫錯就是資料外洩。
2. **不准繞過守門**：`api/generate` 的流程固定是「組 context → LLM → guardrail → 寫 generations → 回傳」，包括除錯用的路徑也不准繞。
3. **無 seed 不生成**：`seed` 為空時 API 直接回 400，不呼叫 LLM。模板是骨，肉一定要是真的。
