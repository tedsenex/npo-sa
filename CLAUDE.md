> 這個檔案放在 repo 根目錄。Claude Code 每次開工都會先讀它。
> 修改專案慣例時請同步更新本檔。

---

## 專案是什麼

`npo-workbench` — 給台灣非營利組織用的行銷 AI 工作台。多租戶 SaaS。

協會在系統裡建一次「協會腦」（全名、勸募字號、核准事實、紅線、品牌語氣），
之後所有 AI 產出都自動帶入這些資料，並且必須通過合規守門才能交付。

商業模式：工具月費 + 年度顧問服務。

---

## 為什麼這個產品存在（讀懂這段才寫得對）

NPO 做行銷最怕三件事：

1. **寫錯數字** — 小編為了寫得動人，順手編了一個沒人查證過的服務人次，被捐款人抓到。
2. **踩到法規** — 台灣《公益勸募條例》規定勸募活動必要支出上限 15%，且對外募款須有勸募字號。字號過期或誤植是行政違規。
3. **AI 味太重** — 產出一看就是機器寫的，反而傷害信任。公益組織的資產就是信任。

**所以本專案的核心不是「生成」，是「守門」。**
生成品質好是基本盤，守門守得住才是產品價值。寫任何功能時都要記得這個優先序。

---

## 技術棧

- Next.js 15（App Router）+ TypeScript strict
- Tailwind + shadcn/ui
- Supabase（Postgres / Auth / Storage / RLS）
- Drizzle ORM
- Anthropic SDK（**只能在 server 端呼叫，絕不可出現在 client component**）
- Zod（所有輸入輸出都要 schema）
- 部署 Vercel

---

## 目錄結構

```
src/
  app/
    (auth)/login/
    (app)/
      dashboard/
      org/setup/            # 協會腦建檔
      m3-content/           # P1 唯一的生成模組
      settings/
    api/
      generate/route.ts     # 統一生成入口
  lib/
    db/
      schema.ts             # Drizzle schema
      queries/
    generators/
      base.ts               # Generator 介面
      m3-content.ts
    guardrails/
      index.ts              # Pipeline
      g1-facts.ts           # 事實核對
      g5-lexicon.ts         # 用語檢查
      dictionaries/
        mainland-terms.json # 大陸用語對照
    ai/
      client.ts             # Anthropic wrapper（含成本記錄）
    auth/
  components/
prompts/
  m3_content/v1.md
```

---

## 硬規則（違反就是 bug）

### 1. 多租戶隔離

除 `users` 外，**每張表都必須有 `org_id`**，且必須寫 RLS policy：

```sql
org_id in (select org_id from memberships where user_id = auth.uid())
```

新增任何表格時，migration 必須同時包含 RLS。沒有 RLS 的表不准 merge。

### 2. 生成一律走 server

Anthropic API key 只能存在 server 環境變數。
任何 `"use client"` 檔案裡出現 `@anthropic-ai/sdk` 就是嚴重錯誤。

### 3. 產出必過守門

`api/generate` 的流程固定為：

```
組 context → 呼叫 LLM → 過 guardrail pipeline → 寫 generations 表 → 回傳
```

**不准有繞過 guardrail 的路徑**，包括除錯用的。

### 4. 數字必須來自事實庫

G1 守門：掃描輸出中的所有數字，凡不在該 org 的 `org_facts` 內、且非日期／價格等中性數字者，
替換為 `【請補資料】` 並在報告中標示。寧可漏掉也不可放行。

### 5. 繁體中文（台灣）

所有 UI 文案、錯誤訊息、AI 產出一律繁體中文台灣用語。

G5 字典要擋的既有清單：`掙來→賺來`、`個性化→個人化`、`身份→身分`、`真論壇→論壇`、`視頻→影片`、`質量→品質`、`信息→資訊`、`默認→預設`、`用戶→使用者`。
（此清單持續擴充，遇到新的就加進 `mainland-terms.json`）

### 6. 不做的事

- 不引入 Redis、訊息佇列、微服務
- 不自建 auth
- 不做即時協作
- 不在 P1 碰生圖、儀表板、銷售頁

---

## 資料模型（P1 只建這些）

```
orgs            id, slug, full_name, short_name, reg_number, authority, plan
org_permits     org_id, permit_number, valid_from, valid_to, approved_use
org_facts       org_id, key, value, source, verified_at, is_active
org_redlines    org_id, rule, severity(block|warn)
org_voice       org_id, tone, banned_words[], sample_posts[], primary_color, accent_color
memberships     user_id, org_id, role(owner|editor|viewer|consultant)
content_items   org_id, platform, archetype, seed, body, status, scheduled_for
generations     org_id, user_id, module, input, output, guardrail_report,
                model, tokens_in, tokens_out, cost_usd
```

完整欄位見 `docs/architecture.md`。

---

## M3 內容模組的核心規則

七型範式（archetype），使用者選一型：

| 代號 | 名稱 | 引擎 |
|---|---|---|
| A1 | 導流長文＋輪播 | 換名單 |
| A2 | 圖一句迷因 | 低成本拉新 |
| A3 | 大字二選一 | 逼留言 |
| A4 | 工具單 listicle | 乾貨（AI 味最高，每週上限 1） |
| A5 | 素樸實作碎念 | 養信任，AI 味最低 |
| A6 | 五步驟＋重複節奏 | 講方法 |
| A7 | 認知反轉大字 | 蹭常識 |

**最重要的規則：無 seed 不生成。**
使用者必須先填「今天實際發生了什麼真事」（真數字／真對象／真事件），
`seed` 欄位為空時 API 直接回 400，不呼叫 LLM。模板是骨，肉一定要是真的。

**禁用句型**：「不是 X 而是 Y」在 A4 以外全部禁止。

---

## P1 任務清單（照順序做）

```
[ ] T1  初始化 Next.js + TS + Tailwind + shadcn，設定 lint / format
[ ] T2  接 Supabase，建 Drizzle schema，寫上表所有 migration（含 RLS）
[ ] T3  Magic Link 登入 + 建立第一個 org + membership
[ ] T4  協會腦建檔頁：orgs / permits / facts / redlines / voice 五個區塊的 CRUD
[ ] T5  lib/ai/client.ts：Anthropic wrapper，含 retry、token 與成本記錄
[ ] T6  Generator 介面 + m3-content generator + prompts/m3_content/v1.md
[ ] T7  Guardrail pipeline 骨架 + G1 事實核對 + G5 用語字典
[ ] T8  /api/generate route：組 context → LLM → 守門 → 寫 generations
[ ] T9  M3 頁面：選 archetype、填 seed、產出、顯示守門報告、存成 content_item
[ ] T10 產出歷史頁：列出 generations，可看守門報告與成本
[ ] T11 種一組 TBCA 假資料，端到端跑通
```

### P1 驗收

用 TBCA 的資料生出一則 Threads 貼文，且滿足：

- 貼文中沒有任何不在 `org_facts` 的數字
- 沒有觸犯 TBCA 紅線（不得出現直接募款語言）
- 沒有簡體字或大陸用語
- 守門報告完整存進 `generations`
- 整趟花費有被記錄

**這一關過了，產品的技術風險就過了。其餘六個模組都是複製 M3 的形狀。**

---

## 開發習慣

- 每個 task 完成後跑 `npm run typecheck && npm run lint`
- Migration 一律用 Drizzle 產生，不手改 SQL 檔
- 環境變數集中在 `src/lib/env.ts`，用 Zod 驗證，缺變數就啟動失敗
- 不寫沒有人會讀的註解；寫「為什麼這樣做」而非「這行在做什麼」
- 提交訊息用繁體中文或英文皆可，但要說清楚動機

---

## 環境變數

```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
ANTHROPIC_API_KEY=
RESEND_API_KEY=
```

---

## 後續模組（P1 完成前不要碰）

M1 行銷健診 · M2 年度募款規劃 · M4 廣告文案 · M5 廣告圖片（Higgsfield）·
M6 廣告數據儀表板（Metricool）· M7 募款銷售頁（需 1shop 內嵌 HTML 規格）
