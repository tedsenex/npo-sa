# M7 募款銷售頁模組｜規格書 v1.0

**產出日期：2026-08-05**
**規格來源**：漸凍人協會 × 2026 美國 3,100 英里公益挑戰（1shop 內嵌版）
**上游文件**：[`architecture.md`](./architecture.md)

---

## 0. 這份頁面的真正價值

拆完之後，最值錢的不是 CSS，是**底層的十二拍募款敘事結構**。
這個結構跟漸凍症無關，換成視障、自閉症、弱勢兒少都成立——**它就是產品要賣的東西**。

```
① 英雄行動      有人正在為你做一件很難的事
② 動機轉折      他為什麼要做（手心向上 → 手心向下）
③ 問題放大      你不知道的規模（7,800 個社福團體）
④ 情境沉浸      具體的日常語句（打字機跑馬燈）
⑤ 知識鋪底      這個病／這個問題是什麼
⑥ 困境結構化    三道牆，每道附「破除迷思」
⑦ 參與感        進度條 + 我要成為其中一個
⑧ 組織可信度    協會在做什麼（三張服務卡）
⑨ 真人證言      受益者第一人稱長敘事（五章）
⑩ 執行者        替他們跑的人是誰
⑪ 資金用途      善款具體投到哪（三站路線）
⑫ 換算與收單    把捐款換算成單位（1 英里 = NT$1,000）
```

**⑥ 的「破除迷思」與 ⑫ 的「換算」是這頁最強的兩個設計**，多數 NPO 募款頁都沒有。

---

## 1. 技術約束（決定整個模組怎麼寫）

### 1.1 這是內嵌頁，不是獨立頁

```css
#brt-root, #brt-root * { all: revert; box-sizing: border-box; }
```

所有樣式包在 `#brt-root` 下，用 `all: revert` 隔離 1shop 母站的 CSS。
**產出格式必須是「單一 HTML 片段」**（`<style>` + `<div id="brt-root">` + `<script>`），不是完整 HTML 文件。

### 1.2 CTA 不自己收單，而是找到母站的方案區

```js
function findPlansSection(){
  // 1. 先試 id：plans / options / sponsor-plans / proposals / reward-list / rewards
  // 2. 再用關鍵字掃標題：點燃愛的火花 / 支持方案 / 選擇方案 / 回饋方案
  // 3. 都找不到 → 捲到頁面底部
}
```

三層 fallback。金流留在 1shop，頁面只負責說服。
**模組必須把這段邏輯保留並可設定**（id 清單與關鍵字清單要能改）。

### 1.3 圖片託管在 1shop CDN

`https://img.1shop.tw/{store}/{asset}/original.png`

產品需要一個上傳流程把協會的圖片推上去，或允許填外部 URL。

---

## 2. 設計 Token

```json
{
  "colors": {
    "bg":           "#FFFDF5",
    "bg_alt":       "#FFF8E1",
    "bg_dark_from": "#2C2416",
    "bg_dark_to":   "#3D2F18",
    "text":         "#2C2416",
    "text_body":    "#5a4e30",
    "primary":      "#FF8F00",
    "primary_alt":  "#F9A825",
    "primary_deep": "#C2620F",
    "accent":       "#FFD54F",
    "accent_soft":  "#FFF3CD",
    "secondary":    "#4A90D9",
    "secondary_lt": "#7BBCF5",
    "secondary_tx": "#4A6FA5",
    "hero_from":    "#5B8FC9",
    "hero_to":      "#7FB2E0"
  },
  "fonts": {
    "display": "Noto Serif TC, serif",      // 標題，weight 900
    "body":    "Noto Sans TC, sans-serif",
    "number":  "Bebas Neue, sans-serif"     // 所有數字
  },
  "radius": { "chip": "100px", "card": "20px", "panel": "32px" }
}
```

**換品牌只需換這份 token。** 主色／副色從 `org_voice.primary_color` / `accent_color` 帶入。

---

## 3. 區塊規格（`landing_pages.blocks` JSON Schema）

每個區塊一個 type，前端依 type 渲染。以下為 12 種。

### B01 `hero`

```ts
{
  type: "hero",
  video_id?: string,          // YouTube ID，桌機自動播放
  still_image: string,        // 手機靜態底圖（必填，video 可省）
  logo?: string,
  kicker?: string,            // 一行前導句
  lead: string,               // 支援 <strong> 與 <br>
  slogan?: string,            // 上下橫線包夾的標語
  big_number?: { value: number, unit: string },
  countdown?: { target_date: string, label: string },
  ctas: CTA[]
}
```

桌機播影片、`max-width:768px` 切靜態圖。`hero-road` 是底部跑動虛線，可關。

### B02 `quote_band`

```ts
{ type: "quote_band", image: string, quote: string, cite: string }
```

左圖右引言，引言中的 `<em>` 會渲染成描邊黃字。

### B03 `panel`（合併卡容器）

```ts
{ type: "panel", parts: Block[] }   // 淺橘漸層長條卡，內含多個子區塊，中間 divider
```

用於把「動機轉折」與「問題放大」包成同一張視覺卡。

### B04 `narrative`

```ts
{ type: "narrative", tag: string, title: string, paragraphs: string[], pull_quote?: string }
```

### B05 `icon_rows`（左右交錯圖文列）

```ts
{ type: "icon_rows", tag, title, items: [{ image, heading, body }] }
```

奇數列圖在左、偶數列圖在右並右對齊。

### B06 `typewriter`

```ts
{ type: "typewriter", tag, title, desc, bg_image, badge_logo?, label, lines: string[] }
```

打字機逐字輸出 → 停 2.4s → 逐字刪除 → 換下一句，無限循環。
`prefers-reduced-motion` 時只顯示第一句。**lines 建議 4–6 句，每句 ≤ 20 字。**

### B07 `explainer`

```ts
{
  type: "explainer", tag, title, desc,
  video_id?: string,
  paragraphs: string[],
  chain?: { items: string[], final: string, note: string }   // 退化順序鏈
}
```

`chain` 是「A → B → C → 終點」的膠囊串，終點用主色實心。

### B08 `wall_cards`（困境三卡）★

```ts
{
  type: "wall_cards", tag, title, desc,
  cards: [{
    image, icon, stage,        // "01 · 確診期"
    eyebrow,                   // "確診的牆"
    heading,                   // 支援 <em> 標主色
    bullets: string[],
    myth: { label, body }      // ★ 破除迷思，這欄不可省
  }],
  closing_quote?: string
}
```

**`myth` 是這個模組的靈魂**：每張卡都要回應一個外界常見的錯誤認知。深色底、卡片浮起。

### B09 `progress`（見 §4，需重新設計）

### B10 `service_cards`

```ts
{ type: "service_cards", tag, title, desc,
  cards: [{ image, logo?, heading, body, chips: string[] }] }
```

### B11 `testimony`（受益者長敘事）★

```ts
{
  type: "testimony", tag, title, desc,
  portrait: string, portrait_caption: string,   // sticky 側欄
  opening_quote: string,
  chapters: [{ no: string, heading: string, paragraphs: string[], quote?: string }],
  cta?: CTA,
  consent_note: string,        // ★ 必填：肖像與內容刊登授權聲明
  footer?: string
}
```

原檔的 `※ 本段整理自訪談影片逐字稿，經本人同意後刊出` 要升級成**必填欄位**。

### B12 `runners` / `allocation` / `equation`

```ts
{ type: "runners", cards: [{ image, badge, name, body }], closing_quote? }

{ type: "allocation", tag, title, lead,       // 三站路線
  stops: [{ no, image, tab, quote, body, fund_use }] }

{ type: "equation", tag, title, desc,
  carousel?: string[],
  formula: { a: {num, unit, label}, op1: "×", b: {...}, op2: "=", result: {...} },
  cta: CTA }
```

---

## 4. ⚠️ 進度條必須改掉

### 4.1 原檔怎麼做的

```js
var START_DATE = new Date(2026, 6, 2);
var WEEKDAY_INCREMENT = 5;   // 平日每天 +5
var WEEKEND_INCREMENT = 10;  // 假日每天 +10
// 依「今天距離起算日幾天」推算出目前人數
```

**這是模擬數字，不是真實參與數。** 頁面對捐款人展示一個持續上升的進度，但那個數字沒有任何實際來源。

### 4.2 為什麼不能進產品

你這個產品的核心賣點是合規守門。系統內建一個「自動生成假進度」的功能，跟賣點直接矛盾——
一旦有協會用它出事，你賠掉的是整條產品線的信任。

### 4.3 三種合法作法（模組只提供這三種）

| 模式 | 說明 | 適用 |
|---|---|---|
| `manual` | 協會自行輸入實際數字，顯示「資料更新於 YYYY/MM/DD」 | 最通用，預設 |
| `api` | 串 1shop / 金流後台實際訂單數 | 有 API 時最佳 |
| `milestone` | 不顯示數字，只顯示已達成的里程碑（如「已募得第一台呼吸器」） | 數字不好看時的誠實解法 |

```ts
{
  type: "progress", tag, title, desc,
  mode: "manual" | "api" | "milestone",
  goal: { value: number, unit: string, label: string },
  current?: number,              // manual 模式必填
  updated_at?: string,           // manual 模式必填，前端要顯示
  source?: string,               // api 模式的資料來源
  milestones?: [{ label, reached: boolean }],
  cta: CTA
}
```

**Guardrail G7（新增）**：`mode: "manual"` 且 `updated_at` 距今超過 30 天 → 前端顯示「資料更新中」而非舊數字；後台提醒協會更新。

---

## 5. 圖片佔位系統（直接沿用，很好用）

原檔的 `.illus-slot` 是一套完整的待補圖佔位框：編號、比例標示、圖示、名稱、說明。

```html
<div class="illus-slot">
  <span class="is-no">IL-03</span>
  <span class="is-ratio">16:10</span>
  <span class="is-ic">🩺</span>
  <span class="is-name">轉診單堆疊</span>
  <span class="is-desc">一張張轉診單，答案在最後一張</span>
</div>
```

**產品化作法**：每個圖片欄位未填時自動渲染佔位框，並在編輯器側欄列出「待補圖清單」（含編號、比例、建議內容）。
協會可以把這份清單直接交給攝影師或設計。**這是一個能直接省掉來回溝通的功能。**

---

## 6. 行為與無障礙（照抄）

| 行為 | 說明 |
|---|---|
| `.rv` 淡入 | IntersectionObserver，threshold .1，卡片群組 `transitionDelay = i*.08s` 錯落 |
| `data-count-to` | 數字滾動，`data-format="comma"`，`data-loop` 可循環 |
| 打字機 | 95ms/字打入、32ms/字刪除、停 2.4s |
| 倒數 | 距目標日天數 |
| 圓形輪播 | 4 slide 循環，2s 換頁 |

無障礙的部分原檔做得完整，**全部保留**：
`skip-link`、`focus-visible` 3px 主色外框、`aria-label` / `aria-labelledby`、
`role="progressbar"` + `aria-valuenow`、`.sr-only`、
以及 `prefers-reduced-motion` 全面關閉動畫。

> NPO 頁面的無障礙不是加分項是必要項——**服務身心障礙者的組織，官網不能有障礙。**
> 這一點可以直接寫進你的行銷文案。

---

## 7. 生成流程

```
選敘事模板（十二拍 / 精簡八拍 / 活動型六拍）
  → 從協會腦帶入：全名、勸募字號、CI 色、核准事實、紅線
  → 逐區塊生成文案（LLM，每個 block type 一個 prompt）
  → 過 Guardrail（G1 事實 / G2 紅線 / G3 字號 / G5 用語 / G7 進度時效）
  → 預覽（iframe 沙箱）
  → 匯出：單一 HTML 片段，可直接貼進 1shop
```

**LLM 只寫文案，不寫 HTML。** 版型是固定模板，AI 產出的是結構化 JSON 填進去。
這樣才可控、才能過守門、才不會每次輸出都不一樣。

---

## 8. 這個模組的新增守門規則

| # | 規則 |
|---|---|
| G7 | 進度數字須有 `updated_at`，逾 30 天不顯示數字 |
| G8 | `testimony` 區塊沒有 `consent_note` 不准發布 |
| G9 | 「換算式」的數字（如 1 英里 = NT$1,000）須有協會確認來源 |
| G10 | 頁面中所有服務數據須來自 `org_facts` |
| G11 | 若使用 AI 生成圖，須帶 `is_ai: true` 並自動附標示 |

---

## 9. 實作優先序

M7 排在 P4，但這份規格現在就該定，因為：

1. **十二拍敘事結構**可以立刻回頭用在 M1 健診的診斷維度（協會的現有募款頁缺了哪幾拍）
2. **B08 破除迷思**與 **B12 換算式**可以立刻抽出來給 M4 廣告文案用
3. 進度條那個問題，**現在不修，之後會變成產品的信任缺口**

---

## 10. 待確認

- [ ] 1shop 的方案區是否有穩定 id 可用（目前靠關鍵字猜）
- [ ] 圖片上傳走 1shop CDN 或自建 Storage
- [ ] 三種敘事模板（十二拍／八拍／六拍）的區塊組合
- [ ] 這份頁面上線後的實際轉換數據——**若有，就是你最強的銷售證據**
