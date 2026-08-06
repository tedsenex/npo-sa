# NPO 行銷 AI 工作台｜系統架構 v1.0

**產出日期：2026-08-05**
**產品形態**：多租戶 Web 應用（SaaS）
**商業模式**：工具月費 + 年度顧問服務
**專案代號**：`npo-workbench` `[暫定]`

---

## 0. 產品定位

> 給 NPO 的行銷作業台。協會建檔一次，七個工具共用同一份「協會腦」，
> 產出全部經過合規守門，再由顧問每年帶著走一輪。

**雙層收費**

| 層 | 內容 | 頻率 |
|---|---|---|
| 工具月費 | 七個模組使用權、產出額度、成效看板 | 月 |
| 年度顧問 | 年度募款規劃工作坊、季度檢視、專案陪跑 | 年 |

工具負責日常產能，顧問負責方向與續約黏著度。**工具讓你可規模化，顧問讓你有定價權。**

---

## 1. 服務 → 模組對應

| 你現有的服務／Skill | 模組 | 狀態 |
|---|---|---|
| `npo-fundraising-audit` | **M1 行銷健診** | Skill 成熟，需簡化成自填問卷 |
| （新，來自健檢框架延伸） | **M2 年度募款規劃** | 需新建，最缺 |
| `ad-engine-content-os` / `nitro-content-os` / 七型範式 | **M3 社群內容排程** | Skill 成熟 |
| `npo-meta-writer` | **M4 廣告文案生產** | Skill 成熟 |
| `npo-fundraising-ad-prompt` v2.0 + Higgsfield | **M5 廣告圖片生產** | Skill 成熟 |
| Metricool 串接 + 成效報告 | **M6 廣告數據儀表板** | 已接通 |
| 3100 英里募款頁（1shop 內嵌 HTML） | **M7 募款銷售頁** | **需要你提供 HTML 原始檔** |
| 勸募法規知識 | **M0 合規守門員**（橫切層，非獨立模組） | 需程式化 |
| — | **M-1 協會腦**（資料層地基） | 需新建 |

---

## 2. 系統分層

```
┌── 呈現層 Next.js App Router ──────────────────────┐
│  儀表板 · 七個模組頁 · 產出歷史 · 設定             │
└───────────────────▲──────────────────────────────┘
┌── API 層 Route Handlers / Server Actions ────────┐
│  認證 · 租戶隔離 · 額度控管 · 生成排程            │
└───────────────────▲──────────────────────────────┘
┌── 生成核心 Generator ────────────────────────────┐
│  Prompt Pack ＋ 輸入 Schema ＋ 輸出 Schema         │
│  ↓                                                │
│  Guardrail Pipeline（守門，見 §5）★                │
└───────────────────▲──────────────────────────────┘
┌── 資料層 Supabase Postgres（RLS 多租戶）─────────┐
│  orgs · org_facts · org_redlines · generations …  │
└───────────────────▲──────────────────────────────┘
┌── 外部服務 ─────────────────────────────────────┐
│  Anthropic API · Higgsfield · Metricool · Resend  │
└──────────────────────────────────────────────────┘
```

---

## 3. 技術選型

| 項目 | 選擇 | 理由 |
|---|---|---|
| 框架 | Next.js 15 App Router + TypeScript | 前後端一體，單人可維護 |
| UI | Tailwind + shadcn/ui | 不自己刻元件 |
| 資料庫 | Supabase Postgres | 內建 Auth、Storage、RLS，多租戶隔離免自己寫 |
| ORM | Drizzle | 型別安全、遷移簡單 |
| 認證 | Supabase Auth（Email Magic Link） | NPO 使用者不擅長記密碼 |
| LLM | Anthropic API（**只在 server 端**） | 內容用 Sonnet，檢查用 Haiku |
| 生圖 | Higgsfield API | M5 才接 |
| 廣告數據 | Metricool API | 已驗證可用 |
| 寄信 | Resend | 週報、邀請信 |
| 部署 | Vercel | |
| 背景任務 | Vercel Cron + DB 佇列 | 不引入 Redis／MQ |

**刻意不用**：微服務、Redis、Kafka、自建 auth、GraphQL。單人專案，複雜度就是敵人。

---

## 4. 資料模型

### 4.1 核心表

```sql
-- 租戶
orgs (
  id uuid pk, slug text unique,
  full_name text not null,        -- 圖上要寫死的協會全名
  short_name text,
  reg_number text,                -- 立案字號
  authority text,                 -- 主管機關
  plan text,                      -- trial / standard / pro
  consulting_until date,          -- 年度顧問到期日
  created_at timestamptz
)

-- 勸募許可（一個協會可有多張）
org_permits (
  id uuid pk, org_id uuid fk,
  permit_number text,             -- 勸募字號
  valid_from date, valid_to date,
  approved_use text,              -- 核准用途
  target_amount numeric
)

-- ★ 核准事實庫：唯一可對外引用的數字來源
org_facts (
  id uuid pk, org_id uuid fk,
  key text,                       -- "累計服務人次"
  value text,                     -- "12,480"
  source text,                    -- "2025 年報 p.14"
  verified_at date,
  is_active boolean
)

-- ★ 紅線
org_redlines (
  id uuid pk, org_id uuid fk,
  rule text,                      -- "不得出現直接募款語言"
  severity text                   -- block / warn
)

-- 品牌調性
org_voice (
  org_id uuid pk fk,
  tone text, banned_words text[],
  sample_posts text[],            -- 3 則範例貼文
  primary_color text, accent_color text,
  logo_url text, photo_library_url text
)

-- 使用者與權限
users (id uuid pk, email text)
memberships (user_id, org_id, role)   -- owner / editor / viewer / consultant
```

### 4.2 業務表

```sql
-- M1 健診
audits (id, org_id, answers jsonb, result jsonb, score int, created_at)

-- M2 年度募款規劃
fundraising_plans (
  id, org_id, year int,
  annual_target numeric,
  channels jsonb,                 -- 各管道目標與佔比
  status text
)
campaigns (
  id, org_id, plan_id, name text,
  type text,                      -- 定期定額/單次/年底抵稅/活動/義賣
  starts_on date, ends_on date,
  target_amount numeric, raised_amount numeric,
  permit_id uuid                  -- 掛哪張勸募許可
)

-- M3 內容
content_items (
  id, org_id, campaign_id,
  platform text,                  -- fb/ig/threads
  archetype text,                 -- 七型範式代號
  seed text,                      -- 真材料，無 seed 不生成
  body text, status text,         -- draft/approved/scheduled/published
  scheduled_for timestamptz
)

-- M4 / M5 廣告
ad_copies (id, org_id, campaign_id, angle text, headline text, body text, cta text)
ad_images (id, org_id, campaign_id, ad_type text, prompt text, image_url text, has_ai_disclaimer bool)

-- M6 數據
metric_snapshots (id, org_id, source text, date date, payload jsonb)

-- M7 銷售頁
landing_pages (
  id, org_id, campaign_id, slug text,
  blocks jsonb,                   -- 區塊化內容
  html text,                      -- 產出的內嵌 HTML
  published_at timestamptz
)

-- 橫切
generations (
  id, org_id, user_id, module text,
  input jsonb, output jsonb,
  guardrail_report jsonb,         -- ★ 守門結果
  model text, tokens_in int, tokens_out int, cost_usd numeric,
  created_at
)
usage_quotas (org_id, month text, generations_used int, images_used int)
```

**RLS 原則**：除 `users` 外每張表都有 `org_id`，policy 一律 `org_id in (select org_id from memberships where user_id = auth.uid())`。**這條寫錯就是資料外洩，Phase 1 就要寫對。**

---

## 5. 生成核心 ★ 系統的靈魂

### 5.1 Generator 抽象

每個模組都是一個 Generator，介面統一：

```ts
interface Generator<TIn, TOut> {
  id: string                       // "m4_ad_copy"
  inputSchema: z.ZodType<TIn>
  outputSchema: z.ZodType<TOut>
  promptPack: PromptPack           // 從 skill 轉來的版本化 prompt
  guardrails: GuardrailId[]
  estimateCost(input: TIn): number
}
```

新增模組 = 新增一個 Generator，不動框架。

### 5.2 Prompt Pack

你的 Skill 是給人在對話裡跑的；Prompt Pack 是給機器跑的。**兩者不共用檔案**（`polytask-threads-writer` 那次拆分又刪掉的教訓：跨檔耦合成本大於模組化好處）。

```
prompts/
  m1_audit/v1.md
  m2_annual_plan/v1.md
  m3_content/v1.md          ← 七型範式
  m4_ad_copy/v1.md
  m5_ad_image/v1.md         ← 六型募款廣告
  m7_landing/v1.md
```

每個 pack 有版本號，`generations` 記錄用了哪版——**改 prompt 後品質變差可以回溯**。

### 5.3 三段式呼叫

```
組裝 context（協會腦 + 事實庫 + 紅線 + 語氣）
  → 呼叫 Anthropic（結構化輸出，強制 JSON）
  → 過 Guardrail Pipeline
  → 存 generations（含守門報告）
```

---

## 6. Guardrail Pipeline ★ 護城河

每一次產出都要過這六關。這是別家做不到的地方——**因為他們不知道公益勸募條例有這些條。**

| # | 守門 | 規則 | 違反處置 |
|---|---|---|---|
| G1 | **事實核對** | 輸出中的數字須存在於 `org_facts`；否則替換成 `【請補資料】` | block |
| G2 | **紅線比對** | 掃 `org_redlines`，severity=block 者直接擋 | block / warn |
| G3 | **勸募字號** | 引用的字號須存在於 `org_permits` 且未過期 | block |
| G4 | **AI 人像標示** | M5 生圖若含人像，強制附「示意畫面，人物為 AI 生成非真實服務對象」 | 自動補 |
| G5 | **用語檢查** | 簡體字、大陸用語（掙來／個性化／身份／真論壇…）字典比對 | 自動改 + 提示 |
| G6 | **禁用句型** | 「不是 X 而是 Y」等 AI 味句型（listicle 型除外） | warn |

G1 與 G5 用純程式即可（字典 + 正則），不需要 LLM。G2、G6 用 Haiku 做語意判斷。

**守門報告存進 `generations.guardrail_report`**，前端顯示成「本次檢查通過 6/6」——這是協會理監事會最想看的東西，也是你的行銷素材。

---

## 7. 模組規格摘要

### M1 行銷健診

問卷約 20 題 → 曝光／認同／捐款率三段診斷 → 缺口排序 + 前三優先動作。
**產品作用**：新客第一站，免費，導流到付費方案與年度顧問。

### M2 年度募款規劃 ★ 最缺也最值錢

輸入：年度目標、現有管道現況、人力、檔期。
輸出：年度分月目標拆解、四大檔期配置（春節／年中／年底抵稅／週年）、各管道目標佔比、月度執行清單、風險提示。

**這個模組是年度顧問服務的數位載體**——工作坊在線下做，成果存在這裡，季度回來檢視。

### M3 社群內容排程

沿用七型範式。**核心規則：無 seed 不生成**——使用者必須先填「今天發生什麼真事」，才給模板。
輸出接內容日曆，可推 Metricool。

### M4 廣告文案生產

角度選擇 → 生成 headline / body / CTA 多版本 → 過守門 → 存庫。

### M5 廣告圖片生產

六型募款廣告（創辦人敘事／破疑慮／贈品回禮／小錢大愛／稅務誘因／能見度補位）。
自動帶入協會全名、勸募字號、CI 色。Higgsfield 生成，G4 強制標示。

### M6 廣告數據儀表板

Metricool 拉 Google Ads / Meta Ads，加 Ad Grants 合規檢查（CTR 5% 門檻、單一關鍵字、$2 出價），月底自動彙整理監事會可用的成效報告。

### M7 募款銷售頁

區塊化編輯（Hero／痛點／方案／信任／FAQ／CTA），輸出可貼進 1shop 的內嵌 HTML。
**依賴**：需要 3100 英里那份 HTML 原始檔，抽出 `#brt-root` 隔離結構與方案區抓取邏輯後參數化。

完整規格見 [`m7-landing-page.md`](./m7-landing-page.md)。

---

## 8. 分期

| Phase | 範圍 | 目標 |
|---|---|---|
| **P1** | 骨架 + 協會腦 + M3 內容 + Guardrail G1/G5 | **能生出第一則合規貼文** |
| **P2** | M1 健診 + M4 文案 + 產出歷史 + 額度 | 可開始收月費 |
| **P3** | M2 年度規劃 + M6 儀表板 | 顧問服務有數位載體 |
| **P4** | M5 生圖 + M7 銷售頁 | 全模組上線 |
| **P5** | 多使用者權限、顧問視角、白標 | 規模化 |

**P1 是唯一有硬性驗收的一期**：TBCA 用它生出一則貼文，且守門報告全綠。

---

## 9. 待你提供

- [ ] **3100 英里最新版 HTML 原始檔**（M7 的規格來源，我不憑印象重寫）
- [ ] TBCA 的 Metricool brand_id
- [ ] 定價（工具月費／年度顧問費）
- [ ] 產品與網域名稱
- [ ] 大陸用語字典的既有清單（你已累積過一批）
