# NPO 行銷系統｜Skill 統整清單 v1.0

**產出日期：2026-08-05**
**上游**：[`architecture.md`](./architecture.md)、[`m7-landing-page.md`](./m7-landing-page.md)、[`../CLAUDE.md`](../CLAUDE.md)
**用途**：定義整個系統要哪些 skill、哪些現成、哪些新建，供 skill-creator 依序開工

---

## 0. 一個關鍵前提

系統裡有兩種「skill」，不要混：

| | 對話用 Skill | 系統用 Prompt Pack |
|---|---|---|
| 在哪 | `/mnt/skills/user/*/SKILL.md` | repo 的 `prompts/{module}/v*.md` |
| 誰跑 | 你在 Claude 對話裡手動跑 | Web App 的 server 端自動跑 |
| 格式 | 含流程、問答、判斷分支 | 純模板 + 變數插槽 + 輸出 schema |
| 關係 | **是 Prompt Pack 的來源與母體** | 從 Skill 蒸餾出來，但獨立維護 |

**原則**：Skill 是你的資產庫與實驗場；Prompt Pack 是把驗證過的 Skill 邏輯「凍結+瘦身」成機器能穩定跑的版本。兩者不共用檔案（`polytask-threads-writer` 拆分又刪掉的教訓）。

本清單同時交代兩邊：先列 Skill 盤點，再列每個模組要蒸餾出的 Prompt Pack。

---

## 1. Skill 盤點（對話層）

### 1.1 直接複用，不動（6 支）

| Skill | 系統用途 | 餵給哪個模組 |
|---|---|---|
| `npo-fundraising-audit` | 募款健檢主邏輯（Step 0 逆向偵察 + 曝光×認同×捐款率） | M1 |
| `npo-fundraising-ad-prompt` | 六型募款單圖 prompt | M5 |
| `npo-social-card` | NPO 深色輪播出圖 | M3 輔助出圖 |
| `npo-meta-writer` | FB／Threads 文案 | M3 / M4 |
| `service-catalog` | 產出對外服務型錄（你要賣這個系統本身時用） | 商務 |
| `brand-positioning-architect` | 新協會入場定調 | M-1 建檔前置 |

### 1.2 需要修改（2 支）

| Skill | 改什麼 | 動機 |
|---|---|---|
| `npo-fundraising-audit` | 末頁加「缺口交給我們每天顧／用工具自己顧」雙導流；把診斷維度對齊 M7 十二拍敘事結構 | 讓健檢變成產品漏斗入口 |
| `threads-method-tbca` → `threads-method-npo` | 從 TBCA 專屬泛化：協會名、紅線、語氣全部變數化；對齊七型範式 | M3 要服務多協會 |

### 1.3 需要新建（5 支）★

| Skill | 職責 | 對應模組 | 優先序 |
|---|---|---|---|
| `npo-org-brain` | 協會腦建檔訪談：把一家協會的身分／勸募／事實／紅線／語氣結構化 | M-1 | **P0** |
| `npo-annual-fundraising-plan` | 年度募款規劃：目標拆解、四大檔期、管道配置 | M2 | P1 |
| `npo-landing-builder` | 十二拍募款頁區塊化生成（產出 1shop 內嵌 HTML 片段） | M7 | P2 |
| `npo-adgrants-ops` | Ad Grants 週檢與調整建議 | M6 | P2 |
| `npo-compliance-guard` | 合規守門：事實核對、字號、15% 上限、用語、AI 標示 | M0 橫切 | **P0** |

---

## 2. 模組 → Prompt Pack 對照（系統層）

系統要凍結成 Prompt Pack 的清單。每個 pack 有版本號，`generations` 記錄用了哪版。

| 模組 | Prompt Pack | 從哪支 Skill 蒸餾 | P幾建 |
|---|---|---|---|
| M-1 協會腦 | `org_brain/v1` | `npo-org-brain`（新） | P1 |
| M0 守門 | 純程式為主，語意判斷用 `guard_semantic/v1` | `npo-compliance-guard`（新） | P1 |
| M1 健診 | `audit/v1` | `npo-fundraising-audit` | P2 |
| M2 年度規劃 | `annual_plan/v1` | `npo-annual-fundraising-plan`（新） | P3 |
| M3 內容 | `content/v1`（七型） | `threads-method-npo` + `npo-meta-writer` | **P1** |
| M4 廣告文案 | `ad_copy/v1` | `npo-meta-writer` | P2 |
| M5 廣告圖 | `ad_image/v1` | `npo-fundraising-ad-prompt` | P4 |
| M7 銷售頁 | `landing/v1` | `npo-landing-builder`（新） | P4 |

M6 儀表板走純資料，無 Prompt Pack。

---

## 3. 新建 Skill 規格（給 skill-creator 開工）

以下五支照優先序。每支給到「description 觸發語 + 核心流程 + 紅線」，細節開工時展開。

---

### 3.1 `npo-org-brain` ★ P0 地基

**description 觸發**：當使用者要替某 NPO 建立「協會腦／品牌事實檔／募款素材基礎資料」，或說「幫某協會建檔」「整理這個協會的基本資料」「做協會的事實庫與紅線」時觸發。

**核心流程**

1. 輸入：協會名稱或官網 URL
2. 逆向偵察（web_search + web_fetch）：立案字號、勸募字號、服務對象、既有募款管道、社群佈局
3. 結構化訪談（缺的欄位逐項問，不臆測）
4. 產出五區塊 JSON：`identity / permits / facts / redlines / voice`
5. **核准事實庫**每一條數字都要問出處，無出處標 `[待查證]`

**紅線**

- 立案字號、勸募字號一律引用官方文件，查不到就留空不編
- 事實庫數字沒出處不得標為已確認
- 紅線欄至少要問出「這個協會什麼話不能說」

**輸出對接**：直接寫入 DB 的 `orgs / org_permits / org_facts / org_redlines / org_voice`

---

### 3.2 `npo-compliance-guard` ★ P0 護城河

**description 觸發**：當要檢查一段 NPO 對外文案／募款素材是否合規，或說「幫我過一下守門」「這段能不能發」「檢查有沒有踩到勸募法規」時觸發。也作為其他 skill 的內部收尾步驟被呼叫。

**六道檢查**

| # | 檢查 | 純程式 or LLM |
|---|---|---|
| G1 事實 | 數字須在該協會 `org_facts` | 程式（掃數字 + 比對） |
| G2 紅線 | 比對 `org_redlines`，block 級直接擋 | LLM 語意 |
| G3 字號 | 引用的勸募字號存在且未過期 | 程式 |
| G4 AI 人像 | 生成人像強制附示意標示 | 程式 |
| G5 用語 | 簡體字＋大陸用語字典 | 程式 |
| G6 句型 | 「不是X而是Y」等 AI 味（listicle 除外） | LLM 語意 |
| G7 進度時效 | 進度數字逾 30 天不顯示 | 程式 |
| G8 授權 | 真人證言須有 consent_note | 程式 |

**紅線**

- 寧可誤擋不可放行：不確定就標 warn 給人看
- 守門報告要能被存進 `generations.guardrail_report`
- 大陸用語字典是活檔，遇到新詞就加

**內建字典（起始版，持續擴充）**

`掙來→賺來、個性化→個人化、身份→身分、真論壇→論壇、視頻→影片、質量→品質、信息→資訊、默認→預設、用戶→使用者、渠道→管道、優化→最佳化(視情境)`

---

### 3.3 `npo-annual-fundraising-plan` P1 最缺

**description 觸發**：當要替 NPO 做年度募款規劃／年度目標拆解／檔期配置，或說「排這個協會明年的募款計畫」「年度募款目標怎麼分」「幫我配全年募款檔期」時觸發。

**核心流程**

1. 輸入：年度目標金額、現有管道現況、人力、既有檔期
2. 目標分月拆解（不是平均分，要對齊捐款季節性）
3. 四大檔期配置：春節前、年中、**年底抵稅（最重）**、週年／議題日
4. 各管道目標佔比（定期定額／單次／企業／義賣）
5. 月度執行清單 + 風險提示

**紅線**

- 定期定額是留存主力，規劃要優先鞏固既有月捐再談拉新
- 年底抵稅檔期是台灣 NPO 全年最大峰，不可弱化
- 所有目標數字標為「規劃值」，不與已募金額混淆

**特殊定位**：這支是**年度顧問服務的數位載體**——工作坊線下做，成果存這裡，季度回來檢視。輸出要能存成可追蹤、可回顧的計畫，不是一次性文件。

---

### 3.4 `npo-landing-builder` P2

**description 觸發**：當要替 NPO 做募款落地頁／捐款頁／專案頁／1shop 內嵌頁，或說「做一個募款頁」「這個專案要一頁銷售頁」「把這個案子變成捐款落地頁」時觸發。

**核心流程**

1. 選敘事模板：十二拍（完整）／八拍（精簡）／六拍（活動型）
2. 從協會腦帶入全名、字號、CI 色、事實、紅線
3. 逐區塊生成文案（12 種 block type，見 [`m7-landing-page.md`](./m7-landing-page.md) §3）
4. 過 `npo-compliance-guard`
5. 產出**單一 HTML 片段**（`#brt-root` 隔離 + 內嵌 style + script）

**紅線**

- LLM 只生文案 JSON，不生 HTML；版型是固定模板
- 進度條只能 manual／api／milestone 三模式，禁止自動遞增假數字
- 困境卡的「破除迷思」欄、證言的 consent_note 欄不可省
- CTA 不自己收單，用 `findPlansSection()` 找母站方案區

**依賴**：十二拍結構與 token 全部依 [`m7-landing-page.md`](./m7-landing-page.md)

---

### 3.5 `npo-adgrants-ops` P2

**description 觸發**：當要維運或健檢 Google Ad Grants 帳號，或說「檢查 Ad Grants」「這個協會的公益廣告帳號怎麼優化」「Ad Grants 被停權怎麼辦」時觸發。

**核心流程**

1. 讀 Metricool / Google Ads 數據
2. 週檢清單：CTR 5% 門檻、單一關鍵字政策、地理定位、轉換追蹤、$2 出價上限
3. 產出調整建議：關鍵字增刪、否定字、RSA 文案改寫
4. 合規風險標記（哪一項再不改會被停權）

**紅線**

- $2 出價上限是 Ad Grants 硬規則（除非用 Maximize Conversions）
- CTR 連續低於 5% 兩個月會停權，這是最高優先警示
- 只給建議，實際帳號操作一律回到人工放行

---

## 4. 開工順序（對齊 CLAUDE.md 的 Phase）

```
P1（先能生一則合規貼文）
  Skill:   npo-org-brain、npo-compliance-guard、threads-method-npo(泛化)
  Pack:    org_brain/v1、content/v1、guard 程式 + guard_semantic/v1

P2（可收月費）
  Skill:   npo-fundraising-audit(改)、npo-adgrants-ops(新)
  Pack:    audit/v1、ad_copy/v1

P3（顧問服務有載體）
  Skill:   npo-annual-fundraising-plan(新)
  Pack:    annual_plan/v1

P4（全模組）
  Skill:   npo-landing-builder(新)
  Pack:    ad_image/v1、landing/v1
```

---

## 5. 一頁總表

| Skill | 狀態 | 模組 | Prompt Pack | Phase |
|---|---|---|---|---|
| `npo-org-brain` | 🆕 新建 | M-1 | org_brain/v1 | P1 |
| `npo-compliance-guard` | 🆕 新建 | M0 | guard_semantic/v1 | P1 |
| `threads-method-npo` | ✏️ 改（從 tbca 泛化） | M3 | content/v1 | P1 |
| `npo-meta-writer` | ✅ 複用 | M3/M4 | content/v1, ad_copy/v1 | P1/P2 |
| `npo-fundraising-audit` | ✏️ 改（加導流） | M1 | audit/v1 | P2 |
| `npo-adgrants-ops` | 🆕 新建 | M6 | — | P2 |
| `npo-annual-fundraising-plan` | 🆕 新建 | M2 | annual_plan/v1 | P3 |
| `npo-fundraising-ad-prompt` | ✅ 複用 | M5 | ad_image/v1 | P4 |
| `npo-landing-builder` | 🆕 新建 | M7 | landing/v1 | P4 |
| `npo-social-card` | ✅ 複用 | M3 出圖 | — | P4 |
| `service-catalog` | ✅ 複用 | 商務 | — | — |
| `brand-positioning-architect` | ✅ 複用 | M-1 前置 | — | — |

新建 5、修改 2、複用 5。

---

## 6. 待確認

- [ ] `threads-method-tbca` 目前是獨立檔還是內嵌在某支 content-os？（決定泛化方式）
- [ ] 大陸用語字典你既有那批的完整清單，併進 `npo-compliance-guard`
- [ ] 五支新 skill 要不要先在對話層各跑一次 Mode A 驗證，再蒸餾成 Pack（建議要）

> 針對本清單的交叉比對結果與待決事項，見 [`open-questions.md`](./open-questions.md) §E。
