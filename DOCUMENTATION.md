# 客製化浴缸互動設計系統 — 完整說明文件

> 版本：2026-07-04｜維護基準：`index.html`（單一檔案，無後端）
> 線上測試網址：https://lyricann-stack.github.io/bathtub-designer-test/
> GitHub Repo：https://github.com/lyricann-stack/bathtub-designer-test

---

## 一、設計概念

讓客戶「依照自己的想像設計浴缸」，並讓設計結果直接變成工廠可用的生產資料，形成完整閉環：

```
客戶端（網頁）                設計端（AutoCAD）              製造端（工廠）
──────────────              ──────────────              ──────────────
選造型 / 調參數 / 手繪形狀  →  DXF 三視圖（可直接修改）   →  模具開發與生產
3D 即時預覽 360° 旋轉          JSON 規格表（系統可讀）
      ↑                            │
      └────── 上傳 CAD 檔案還原設計 ┘   ← 閉環：設計師改完傳回即渲染
```

核心價值：客戶不需要會 CAD，設計師不需要重新畫圖，參數在整條鏈路上不失真。

---

## 二、技術架構

| 項目 | 內容 |
|---|---|
| 形式 | 單一 `index.html`，HTML + CSS + JS 全部內含 |
| 3D 引擎 | Three.js r128，CDN：`https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js`（唯一外部依賴） |
| 後端 | 無。所有計算（建模、DXF 產生、輪廓辨識、STL 解析）都在瀏覽器完成 |
| 託管 | GitHub Pages（`main` 分支根目錄，push 後自動部署，約 1 分鐘） |
| 相容性 | 桌機 / 手機瀏覽器；<700px 寬自動改上下堆疊版面 |

---

## 三、功能總覽（按開發順序）

1. **參數化建模**：圓角矩形／圓端形／橢圓形三種基本缸口，長、寬、高、壁厚、底厚、圓角半徑滑桿即時重建 3D。
2. **不對稱造型**（對應 38.JPG 照片款）：
   - 靠背增高 `dH`：後端缸緣比前端高，缸緣呈斜面。
   - 增高弧度 `arc` 0–100%：0%＝直線增高，越高 S 曲線越彎（冪次混合函數 `u^p/(u^p+(1-u)^p)`，p∈[1,4]）。
   - 蛋形係數 `egg`：前端變窄、後端變寬。
   - 底部收縮 `taper`：缸底佔缸口比例，側壁以貝茲曲線外鼓內收。
   - 「★ 套用照片款」一鍵預設組合。
3. **缸緣造型**：平面／圓弧（半圓滾邊，隆起＝壁厚一半）／斜角（三角脊）。
4. **✏️ 手繪模式**：
   - 俯視缸口：畫布手繪或上傳紙上手稿照片（灰階 → Otsu 閾值 → 邊界淹水填充 → Moore 邊界追蹤 → RDP 簡化 → Chaikin 平滑 → 重取樣 96 點）。
   - 側面剖面：畫單邊外牆曲線取代參數剖面。
5. **水位模擬**：沿內壁放樣的水體、水面平坦、固定內部深度 80%，藍色半透明，可開關。
6. **360° 自由旋轉**：拖曳全方位（含仰視，地板單面材質不擋視線）、滾輪縮放、觸控雙指縮放。
7. **三語介面**：繁中／简中／EN，左上角切換，涵蓋畫布導引文字與錯誤訊息；`?lang=en` 可直接分享。
8. **輸出**：
   - DXF：俯視（含底部輪廓虛線）＋前視（缸緣弧線取樣）＋側視三視圖、標註、AC1009/R12 格式（AutoCAD 直接開）、PARAMS 圖層內嵌參數 JSON。
   - JSON 規格表：全部設計參數＋計算規格（容量、重量）＋手繪輪廓座標。
9. **⬆ CAD 上傳渲染**：
   - 本系統 DXF → 讀 PARAMS 完整還原（閉環）。
   - 任意 2D DXF（LINE/ARC/LWPOLYLINE 含 bulge/CIRCLE，mm）→ 線段鏈接取最大封閉迴路當缸口。
   - STL（二進位＋ASCII，自製解析器）→ 直接渲染檢視，自動置中落地框取。
   - 本系統 JSON → 完整還原含手繪資料。
10. **即時規格計算**：內部尺寸、底部尺寸、滿水容量（截面積 k(v)² 數值積分）、八成滿水量、壓克力估重（密度 1.19 g/cm³）。
11. **網址參數**：`?preset=photo`（照片款）、`?lang=zhT|zhS|en`、`?phi=`／`?theta=`（視角）。

---

## 四、程式碼結構導覽（index.html 內區塊順序）

| 區塊（搜尋註解關鍵字） | 職責 |
|---|---|
| `<style>` | 版面。`.row` 兩行式滑桿排版；`@media (max-width:700px)` 響應式 |
| `多語系 i18n` | `I18N` 字典（繁中字串為 key，`[简中, EN]` 為值）、`I18N_HTML`（含標記區塊）、`t()`、`applyLang()`、`setLang()` |
| `Three.js 場景` | renderer / camera（球座標 `orbit`）/ 燈光 / 地板 / `updateCamera()` |
| `輪廓產生` | `rawOutline()` 基本形 → `resample()` 均勻弧長重取樣 → `outlinePts()`（套蛋形、custom 分支）；`rimH()` 缸緣高度曲線；`bez()`/`wallK()` 牆壁剖面；`baseK()` 缸底比例 |
| `曲面放樣建模` | `loftGeometry()` 外殼/內壁、`stripGeometry()` 缸緣（平面/圓弧/斜角）、`capGeometry()` 封蓋、`buildTub()` 總組裝（含水體放樣、排水孔） |
| `規格計算` | `computeSpec()`（數值積分容量/重量）、`updateSpec()` 表格 |
| `DXF 輸出` | 實體產生器 `E.line/circle/text/poly/path`、`endProfile()` 剖面線、`exportDXF()`（三視圖＋PARAMS 內嵌） |
| `JSON 規格表輸出` | `exportJSON()`、`download()` |
| `手繪模式` | `makePad()` 畫布、`processTopSketch()`/`processSideSketch()` 幾何處理、`rdp()`/`chaikinClosed()`、`traceImage()` 照片輪廓辨識、`confirmSketch()` |
| `CAD 檔案匯入` | `importSpecJSON()`、`importDXF()`（解析＋`chainLoops()` 鏈接）、`importSTL()`、`applyParams()`、`clearExt()` 外部模型模式 |
| `UI 綁定` | `sliderMap` 滑桿、`setShape/setDrain/setRim`、`applyPhotoPreset()`、`syncUI()`、`updateRowVis()` |
| 檔尾啟動區 | 網址參數解析、`collectI18nNodes()`、`buildTub()`、`animate()` |

**狀態中心**：全域物件 `P` 保存所有設計參數（含 `customPts` 手繪輪廓、`customProfile` 手繪剖面）。任何參數變更 → 呼叫 `buildTub()` 全量重建（幾何都會 dispose，不會漏記憶體）。

---

## 五、資料格式

### JSON 規格表（exportJSON 產出、importSpecJSON 可讀回）
```json
{
  "設計參數": {
    "shape_code": "ellipse | rect | stadium | custom",
    "外部長度_mm": 1700, "外部寬度_mm": 850, "缸緣高度_前端_mm": 520,
    "靠背增高_mm": 130, "增高弧度_pct": 60, "蛋形係數_pct": 12, "底部收縮_pct": 72,
    "缸壁厚度_mm": 15, "缸底厚度_mm": 40, "rim_code": "round | flat | bevel",
    "排水孔位置": "center | back | front",
    "手繪俯視輪廓_normalized": [[x,y]...96點, "±0.5 正規化，可為 null"],
    "手繪側牆剖面_k": [25個 k 值或 null]
  },
  "計算規格": { "滿水容量_L": "...", "估計重量_kg": "..." }
}
```

### DXF 圖層
| 圖層 | 內容 |
|---|---|
| OUTLINE | 可見輪廓（俯視內外緣、剖面、前/側視外形） |
| HIDDEN | 虛線（底部輪廓、缸內底面、內壁） |
| CENTER | 中心線 |
| TEXT / DIM / TITLE | 視圖標題、尺寸文字、標題欄 |
| PARAMS | 一行 JSON（`{"_bathtub":1,...}`），供本系統上傳還原。**設計師修改圖面時請保留此文字** |

---

## 六、部署與更新流程

- Repo：`lyricann-stack/bathtub-designer-test`（Public），本機工作副本在 `~/Documents/bathtub-designer-test/`。
- 開發母版在專案資料夾 `3D渲染圖轉換成CAD圖/index.html`（本資料夾），改完後：
  ```bash
  cp "…/3D渲染圖轉換成CAD圖/index.html" ~/Documents/bathtub-designer-test/
  cd ~/Documents/bathtub-designer-test && git add -A && git commit -m "說明" && git push
  ```
  push 後 GitHub Pages 約 1 分鐘自動部署。若部署失敗（GitHub 偶發），到 repo → Actions → 失敗的 run → Re-run jobs，或推一個空 commit 重新觸發。
- 本機預覽：資料夾內 `python3 -m http.server 8123` 後開 `http://localhost:8123/index.html`（直接雙擊 html 也可）。

---

## 七、嵌入其他網頁

### 方法 A：iframe（推薦，最快）
```html
<iframe
  src="https://lyricann-stack.github.io/bathtub-designer-test/?lang=en"
  style="width:100%; height:90vh; border:none;"
  allow="fullscreen"
  title="Bathtub Designer"></iframe>
```
- 要指定預設語言/造型就在 src 加參數：`?lang=en&preset=photo`。
- 正式上線建議另建正式 repo（如 `bathtub-designer`）與網域 CNAME，測試 repo 留給內部。

### 方法 B：整頁直接放進網站
把 `index.html` 改名（如 `designer.html`）放入網站根目錄即可，無其他檔案依賴。注意：
1. 需要能連到 cdnjs 載入 three.js；若網站要完全自託管，下載 `three.min.js` 放同目錄並改 `<script src>`。
2. 頁面自帶 `<header>` 與整頁版型；嵌入既有版型時建議仍用 iframe，避免 CSS 衝突。

### 方法 C：與後端整合（未來）
- 「送出訂單」：在 `exportJSON()` 產出的物件直接 `fetch()` POST 到你的 API 或 Email 服務即可，格式見第五節。
- 客戶登入/存檔：把 `P` 物件序列化存 DB，載入時走 `applyParams()` + `buildTub()`。

---

## 八、維護指南（常見修改怎麼改）

| 想做的事 | 改哪裡 |
|---|---|
| 改參數預設值/範圍 | HTML 中對應 `<input type="range">` 的 min/max/value ＋ JS 開頭 `P` 物件 |
| 新增基本缸型 | `rawOutline()` 加分支 → 加 shape 按鈕 → `I18N` 加翻譯 |
| 改照片款預設 | `applyPhotoPreset()` 內的參數組 |
| 加一種語言 | `I18N` 每個 key 的陣列加一欄、`I18N_HTML` 同步、header 加按鈕、`t()`/`applyLang()` 的索引邏輯加語言代碼 |
| 改 DXF 內容/標註 | `exportDXF()`，實體用 `E.*` 產生器；驗證可用 Python `ezdxf.readfile()` |
| 改容量/重量公式 | `computeSpec()`（目前壓克力 1.19 g/cm³，taperF 為 k(v)² 數值積分） |
| 改水位比例 | `buildTub()` 水體區塊 `inn.D*0.8` 與 `computeSpec()` 的 `*0.8` 要一起改 |
| 除錯 | 瀏覽器 DevTools Console；幾何問題先看 `P` 物件目前值；`node --check` 可驗語法（把 `<script>` 內容抽出） |

### 已知限制
- DWG／STEP 無法在純網頁解析（AutoCAD 專有／需 WASM 大型函式庫）→ 請設計師另存 DXF 或 STL。
- DXF 匯入假設單位為 mm；SPLINE 實體目前略過。
- 手稿照片辨識需深色筆＋白紙＋均勻光線；輪廓有缺口會失敗（會提示重畫）。
- 內壁厚度用縮放近似（非精確等距 offset），對製造圖以設計師修訂後的 DXF 為準。
- 估重/容量為概念參考值，非工程精算。

### 未來擴充建議
- 嵌入式浴缸類型與安裝開孔尺寸。
- STL/STEP 輸出（供模具 3D 軟體，可考慮 occt-import-js WASM）。
- 訂單送出直接 Email 給設計師＋客戶資料庫。
- 正式 repo + 自訂網域 + 版本標籤（tag）管理。

---

## 九、檔案清單（本資料夾）

| 檔案 | 說明 |
|---|---|
| `index.html` | 系統本體（開發母版，與線上版同步） |
| `README.md` | 簡要使用說明 |
| `系統說明文件.md` | 本文件 |
| `38.JPG` | 不對稱蛋形缸參考照片 |
| `測試檔/` | CAD 匯入測試檔（designer_edit.dxf 參數閉環／outline_hex.dxf 輪廓／cylinder.stl），可刪 |
