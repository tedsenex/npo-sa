# 待決事項

**建立日期：2026-08-06**

交叉比對 `CLAUDE.md`、`docs/architecture.md`、`docs/m7-landing-page.md` 後，
以下是三份文件之間不一致、或實作時一定會撞上但目前沒寫的地方。
**每一項在動到相關 task 之前都要先拍板**，否則會寫出彼此矛盾的程式。

---

## A. 三份文件互相衝突的地方

### A1. P1 到底要不要做紅線守門（G2）？★ 影響 T7

- `CLAUDE.md` 的 **P1 驗收**列了「沒有觸犯 TBCA 紅線（不得出現直接募款語言）」。
- 但 `CLAUDE.md` 的 **T7** 只寫「Guardrail pipeline 骨架 + G1 事實核對 + G5 用語字典」，
  `architecture.md` §8 的 P1 範圍同樣只有 G1/G5。

驗收條件要 G2，任務清單沒有 G2。**驗收條件比較大，兩者必須擇一調整。**

建議：P1 做「關鍵字比對版 G2」。`org_redlines` 這張表 P1 本來就要建（T4 的五個區塊之一），
純字串／正則比對不需要 LLM，成本接近零，就能滿足驗收。
語意判斷版（architecture 說的 Haiku）留到 P2。

### A2. 「不是 X 而是 Y」是 block 還是 warn？

- `CLAUDE.md`「M3 核心規則」寫**禁用句型**，語氣是硬禁止。
- `architecture.md` §6 的 G6 處置寫的是 **warn**。

兩者對 A4／listicle 型除外的認定一致，只有嚴重度不同。要選一個寫進 pipeline。

建議：`warn` + 自動改寫建議。這是文風問題不是合規問題，擋掉會讓使用者很煩。

### A3. 守門本身的 LLM 成本要不要記進 `generations.cost_usd`？

`architecture.md` §6 說 G2、G6 用 Haiku 做語意判斷——**守門自己也會燒 token**。
但 `generations` 的 `tokens_in / tokens_out / cost_usd` 讀起來像只記生成那一次。

要決定：`cost_usd` 是「生成成本」還是「這趟總成本」。
建議記總成本，另外拆 `cost_breakdown jsonb` 存各段明細，否則毛利算不準。

### A4. M7 規格書的上游文件檔名

M7 規格書原本寫的上游是 `npo-saas-architecture.md`，但 `CLAUDE.md` 指定的路徑是 `docs/architecture.md`。
**已統一為 `docs/architecture.md`**，M7 文件內的連結也已改。若之後檔名要改回，兩處要一起改。

---

## B. 文件沒寫但實作一定會撞到的

### B1. `memberships` 的 RLS 會無限遞迴 ★ 影響 T2，這題不解決 T2 過不了

兩份文件都寫 policy 一律是：

```sql
org_id in (select org_id from memberships where user_id = auth.uid())
```

這條套在**其他表**上沒問題，但套在 `memberships` 自己身上時，
Postgres 會在評估 policy 時再次觸發同一條 policy，直接噴
`infinite recursion detected in policy for relation "memberships"`。

解法是把查詢包進 `security definer` 函式（函式內不再受 RLS 管轄，所以不遞迴）：

```sql
create or replace function public.current_org_ids()
returns setof uuid
language sql
stable
security definer
set search_path = public
as $$
  select org_id from public.memberships where user_id = auth.uid()
$$;
```

之後所有表統一寫 `org_id in (select public.current_org_ids())`。
順帶一提這樣每張表的 policy 都一模一樣，migration 可以用迴圈產生，不容易漏。

### B2. `orgs` 這張表沒有 `org_id`

硬規則說「除 `users` 外，每張表都必須有 `org_id`」，但 `orgs` 自己就是租戶本體，
它的 `id` 就是 `org_id`。policy 要寫成 `id in (select public.current_org_ids())`。

**這是規則的合法例外，要寫進註解**，否則下一個人 review 時會以為漏了。
`org_voice` 用 `org_id` 當 pk，同樣要注意 policy 寫的是 pk 欄位。

### B3. `users` 表要自建還是直接用 `auth.users`？

`architecture.md` §4.1 列了 `users (id uuid pk, email text)`，
但 `CLAUDE.md` 硬規則寫「不自建 auth」，而 Supabase Auth 已經有 `auth.users`。

要決定是「直接 FK 到 `auth.users(id)`」還是「建一張 `public.users` 鏡像表用 trigger 同步」。
建議前者，少一層同步就少一種不一致。真的需要 profile 欄位時再開 `public.profiles`。

### B4. P1 的 `content_items` 有 `campaign_id`，但 `campaigns` 是 P3 才建

`architecture.md` §4.2 的 `content_items` 帶 `campaign_id`，
而 `CLAUDE.md` 的 P1 資料模型沒有 `campaigns` 這張表。

建議 P1 的 `content_items` 先不要這個欄位，P3 建 `campaigns` 時再一起加。
留一個指向不存在的表的欄位，只會讓 T2 的 migration 卡住。

### B5. A4「每週上限 1」的「週」怎麼算？誰來擋？

`CLAUDE.md` 寫 A4 每週上限 1，但沒說週界在哪、也沒說在哪一層擋。

要決定兩件事：

1. 週界：建議以 `Asia/Taipei` 的週一 00:00 為界（NPO 的工作週）。
2. 擋的位置：這是**生成前**的配額檢查，不是守門（守門檢查的是輸出）。
   應該放在 `/api/generate` 呼叫 LLM 之前，跟 seed 檢查同一段，超過就回 409。

### B6. G1 的「中性數字」白名單邊界

硬規則寫「非日期／**價格**等中性數字者」替換成 `【請補資料】`。
但對 NPO 來說「每月 500 元」這種價格**就是**募款訴求的核心數字，放行反而危險。

要把白名單列清楚。建議只放行：日期／時間／年份、序數與清單編號、電話與郵遞區號、
法規條號、勸募字號本身。**金額一律不放行**，要引用就進 `org_facts`。
硬規則說「寧可漏掉也不可放行」，那金額就該從嚴。

### B7. 守門判定 block 之後，API 回什麼？

流程寫的是「→ 過 guardrail → 寫 generations → 回傳」，但沒定義 block 時的回傳形狀。

建議：

- **不論守門結果如何，只要 LLM 被呼叫過就一定要寫 `generations`**（含成本）。
  否則「被擋下的產出不記帳」會變成成本黑洞。
- API 回 200 + 已遮蔽的文字 + 完整守門報告，讓使用者看得到問題在哪。
  用 HTTP 錯誤碼表達守門結果會讓前端拿不到報告。
- 但 `content_items` 的存檔要擋：verdict = block 時 status 只能停在 `draft`，不得排程或發布。

### B8. `generations` 要不要存 LLM 的原始輸出？

守門會改寫文字（G1 替換數字、G5 改用語）。目前只有一個 `output` 欄位，
存的是改寫後的結果，那就**無法稽核守門到底改了什麼**。

建議加 `raw_output`，並且在 UI 上限制只有 `owner` / `consultant` 看得到——
原始輸出裡可能有未查證的數字，不該讓一般編輯直接複製走。

---

## C. 從兩份文件搬過來的外部待辦

來自 `architecture.md` §9「待你提供」：

- [ ] 3100 英里最新版 HTML 原始檔（M7 的規格來源）
- [ ] TBCA 的 Metricool brand_id
- [ ] 定價（工具月費／年度顧問費）
- [ ] 產品與網域名稱（`npo-workbench` 目前是暫定代號）
- [ ] 大陸用語字典的既有清單

來自 `m7-landing-page.md` §10：

- [ ] 1shop 的方案區是否有穩定 id 可用
- [ ] 圖片上傳走 1shop CDN 或自建 Storage
- [ ] 三種敘事模板（十二拍／八拍／六拍）的區塊組合
- [ ] 3100 英里頁面上線後的實際轉換數據

---

## D. 動工前的最小決策集

只有這五題會擋住 T2（建 schema 與 migration），其餘可以邊做邊定：

1. **B1** RLS 遞迴的解法 → 決定 `current_org_ids()` 要不要用
2. **B2** `orgs` / `org_voice` 的 policy 寫法
3. **B3** `users` 用 `auth.users` 還是自建鏡像
4. **B4** `content_items.campaign_id` P1 要不要留
5. **A3 / B8** `generations` 的欄位要不要加 `cost_breakdown` 與 `raw_output`
