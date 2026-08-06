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

## E. Skill 層（比對 `skills-inventory.md` 後補，2026-08-06）

### E1. `threads-method-tbca` 這支 skill 不存在 ★ 這回答了清單 §6 第一題

清單 §1.2 要把 `threads-method-tbca` 泛化成 `threads-method-npo`，
§6 也問「它是獨立檔還是內嵌在某支 content-os」。

查過目前可用的 skill 清單，**沒有這個名字的 skill**。
七型範式的邏輯內嵌在 `ad-engine-content-os` 裡——那支的說明明確寫著
「Threads 從七型範式隨機挑 3 型」，NPO 線走它的 §2.1-C。

→ §1.2 的來源要改成「從 `ad-engine-content-os` 的 NPO 線抽出七型範式邏輯」。
泛化的難度也不同：不是改一支專屬 skill 的變數，是從一支很大的週排程 skill 裡**切出**內容生成那段。

### E2. `npo-meta-writer` 已被取代，而且品牌是寫死的 ★

清單把它列為「✅ 複用不動」，同時餵給 M3 與 M4。兩個問題：

1. `ad-engine-content-os` 的說明寫著它「**取代並合併** ai-daily-recap-writer +
   npo-threads-writer + **npo-meta-writer**」。清單把一支已被取代的 skill 當成 M3/M4 的主要來源。
2. 更要緊的是，`npo-meta-writer` 是**專為「8件小事」品牌**寫的，協會名寫死在裡面。
   M3/M4 要服務多協會，它不可能「複用不動」。

→ 要嘛改從 `ad-engine-content-os` 抽，要嘛把它一起列入「需泛化」。

### E3. `npo-social-card` 的配色也是寫死的

版型是「黑底白字 × **焰橘**」。但 `org_voice` 有 `primary_color` / `accent_color`，
多協會共用時一定要吃這兩個欄位，否則所有協會的輪播圖長得一模一樣。

→ 從「✅ 複用」改成「✏️ 修改（配色變數化）」。

### E4. 因此 §5 的統計要改

「新建 5、修改 2、複用 5」→ 實際上是 **新建 5、修改 4、複用 3**。

| | |
|---|---|
| 修改 | `npo-fundraising-audit`、七型範式（來源改為 `ad-engine-content-os`）、`npo-meta-writer`、`npo-social-card` |
| 複用 | `npo-fundraising-ad-prompt`、`service-catalog`、`brand-positioning-architect` |

「真正能原封不動複用的只有三支」是個值得知道的事實——泛化的工作量比清單預估的多一倍。

### E5. `npo-fundraising-audit` 同時出現在「不動」與「要改」兩張表

§1.1 第一列與 §1.2 第一列都是它，§5 總表寫的是「✏️ 改（加導流）」。
→ §1.1 該刪掉這列。（刪掉後複用剛好 5 支，正好對上 §5 的統計，可以確定是 §1.1 多列了。）

### E6. §1.3 的「優先序」欄用 P0/P1/P2，會被讀成 Phase

§1.3 標 org-brain / guard = P0、annual-plan = P1、landing / adgrants = P2。
但 §2、§4、§5 三處的 Phase 是：org-brain / guard = P1、adgrants = P2、annual-plan = **P3**、landing = **P4**。

那一欄看起來是排序序號（第 0、第 1、第 2 順位），但寫成 `P0/P1/P2` 跟 Phase 撞名。
→ 改成 ①②③，或直接對齊 Phase。

### E7. `npo-compliance-guard` 標題寫「六道檢查」，表列八道

G1～G8。標題沒更新而已。

### E8. guard 只做到 G8，但 M7 需要 G9～G11

`m7-landing-page.md` §8 定義了 G9（換算式數字須有來源）、G10（服務數據須來自 `org_facts`）、
G11（AI 生成圖須標示）。而 `npo-landing-builder` 的流程第 4 步是「過 `npo-compliance-guard`」，
但 guard 的清單只到 G8。

→ guard 補三關，或 landing-builder 自帶。前者比較好，守門集中在一個地方。

### E9. 對話層的 guard 和系統層的 guard，嚴格度根本不同 ★

清單 §2 說 M0「純程式為主，語意判斷用 `guard_semantic/v1`」。
但**對話層的 `npo-compliance-guard` skill 沒有程式可以跑**——
G1 掃數字比對事實庫、G5 查字典、G3 查字號效期，在對話裡全部只能靠 LLM 盡力做。

這代表：**對話層跑過 guard ≠ 系統層會過**，反過來也是。

→ 影響 §6 第三題（先在對話層各跑一次驗證再蒸餾）。這個建議是對的，但要講清楚它驗的是什麼：
**對話層驗證的是文案品質與判斷邏輯，不是守門正確性。**
守門正確性只能在系統層用黃金測試集驗（見 `gap-analysis.md` C4）。
兩件事不要混，否則會以為 skill 跑得順就等於守門做好了。

### E10. 大陸用語字典擴充了兩條，但有一條純字典做不到

清單新增 `渠道→管道`、`優化→最佳化(視情境)`，CLAUDE.md 硬規則 5 的清單要同步補上。

但 **「優化→最佳化(視情境)」帶條件，純字典比對做不到**。
「優化」在台灣的技術與商業語境其實很常用，無條件替換會改錯。

→ `mainland-terms.json` 的 schema 要支援三種處置，不能只有「替換」：

```json
{ "from": "視頻", "to": "影片", "action": "auto" }
{ "from": "優化", "to": "最佳化", "action": "warn", "note": "技術語境可保留，交人工判斷" }
```

`auto` 自動改、`warn` 只提示不改、`block` 直接擋。「優化」歸 `warn`。

### E11. Prompt Pack 命名少了模組前綴

CLAUDE.md 的目錄結構與 `architecture.md` §5.2 都是 `prompts/m3_content/v1.md`（**有**模組前綴），
清單 §2 寫的是 `content/v1`、`audit/v1`（**沒有**前綴）。

→ 統一用有前綴的版本，排序時自然按模組分組。

另外清單新增了 `org_brain/v1` 與 `guard_semantic/v1` 兩個 pack，
`architecture.md` §5.2 的目錄樹沒有它們，要補。

### E12. M6 說「無 Prompt Pack」，但 adgrants-ops 要改寫 RSA 文案

§2 說「M6 儀表板走純資料，無 Prompt Pack」，§5 總表 adgrants 的 pack 欄也是「—」。
但 §3.5 的流程第 3 步是「產出調整建議：關鍵字增刪、否定字、**RSA 文案改寫**」——文案改寫一定要 LLM。

→ 要決定：Ad Grants 建議只在對話層做（顧問跑 skill、系統只顯示數據），
還是系統層也要做（那就需要一個 pack）。

### E13. `brand-positioning-architect` 與 `npo-org-brain` 的分工要界定

兩支都在做協會定調，重疊在「語氣 / voice」那塊。

建議分法：`brand-positioning-architect` 做**策略層**（定位、使命、品牌人格），
`npo-org-brain` 做**資料層**（身分、字號、事實、紅線）。
`org_voice.tone` 到底由誰產出，要講明。

### E14. `npo-org-brain` 補上了自檢 A1 的一半——而且這是好消息 ★

`gap-analysis.md` A1 說「事實庫沒有匯入機制，是產品的單點失敗」。
`npo-org-brain` 正好做這件事：逆向偵察 + 結構化訪談 + 每條數字都要問出處。

但它是**對話層 skill**，跑的人是你或顧問，不是協會自己。

這其實**讓 P1 變小了**：如果接受「P1 的協會腦由顧問用 skill 建檔後匯入 DB」，
那 `gap-analysis.md` 的 A1（web app 的年報匯入 UI）與 A2（onboarding 引導）都可以往後排，
P1 只需要一個「匯入 JSON」的後台功能就夠了。

**但要明確承認一件事：這代表 P1 不是自助 SaaS，是顧問陪跑工具。**
這個定位決定了 P2 要不要補自助建檔，也決定了定價頁怎麼寫。這題值得單獨拍板。

---

## D. 動工前的最小決策集

會擋住 T2（建 schema 與 migration）的：

1. **B1** RLS 遞迴的解法 → 決定 `current_org_ids()` 要不要用
2. **B2** `orgs` / `org_voice` 的 policy 寫法
3. **B3** `users` 用 `auth.users` 還是自建鏡像
4. **B4** `content_items.campaign_id` P1 要不要留
5. **A3 / B8** `generations` 的欄位要不要加 `cost_breakdown` 與 `raw_output`

會擋住 T6／T7（Generator 與守門）的：

6. **E1 / E2** 七型範式從哪裡抽——沒有 `threads-method-tbca` 這支，來源要重新指定
7. **E10** `mainland-terms.json` 的 schema 要不要支援 `auto` / `warn` / `block` 三種處置
8. **E11** Prompt Pack 的命名慣例

會影響產品定位與定價的：

9. **E14** P1 是自助 SaaS 還是顧問陪跑工具
