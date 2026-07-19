# NotchGlass — 規格書

> macOS 瀏海（notch）懸浮小工具
> 設計語言：Apple **Liquid Glass**（液態玻璃）
> 文件版本：v1.0 — 2026-06-10

---

## 0. 文件用途

本文件定義 NotchGlass 的**完整功能、互動、樣式與技術規格**，作為開發與驗收的單一依據。
標記說明：
- 🟢 已實作　🟡 部分完成／待驗證　🔴 未完成／失效中
- ⭐ 最高優先

---

## 1. 產品定位

| 項目 | 內容 |
|---|---|
| 形態 | 常駐於 MacBook 瀏海處的懸浮小工具，平時縮成藥丸狀，靠近時展開 |
| 核心價值 | ① 音樂控制　② 拖曳檔案做快速操作（暫存／轉檔／AirDrop／iCloud） |
| 參考對象 | Apple Dynamic Island、macOS 26 Liquid Glass |
| 專案路徑 | `/Users/harry/Desktop/NotchGlass/` |
| 最低系統 | macOS 13（Ventura）以上，相容含/不含實體瀏海的機型 |

---

## 2. 設計語言：Liquid Glass

整體要呈現 Apple Liquid Glass 的「**一塊會折射環境光的液態玻璃懸浮在桌面上**」的感覺。

### 2.1 材質（Material）
- **即時背景模糊**：真實 backdrop blur（`NSVisualEffectView`，material 視狀態切換），透出後方桌布與視窗的模糊色彩。
- **半透明玻璃本體**：玻璃層之上疊一層可調明度的暗色蓋板
  - 收合（瀏海狀態）：蓋板近全黑（opacity ≈ 1.0），看起來像實體瀏海。
  - 展開：蓋板降到 opacity ≈ 0.42–0.46，透出毛玻璃的透明感。
  - ⚠️ 毛玻璃層**必須永遠存在、絕不條件式插入/移除**，否則會出現「黑頻」（NSVisualEffectView 首次插入/尺寸變動閃黑一幀）。
- **鏡面高光（Specular highlight）**：玻璃頂部一道由白到透明的細微漸層（模擬光打在玻璃上緣），上緣最亮。
- **折射色調（Refraction tint）**：左上→右下方向，疊一層極淡的品牌色漸層（accent1→透明），製造玻璃帶色的折射感。
- **頂部彩色細線**：展開時，面板頂端淡入一條 0.5pt 的水平漸層線（透明→accent1→accent2→透明），寬度內縮，像光被玻璃邊緣截斷。
- **玻璃邊框**：展開時，整塊面板描一圈由上而下變淡的白色細邊（lineWidth ≈ 0.75，opacity 0.22→0.03），凸顯玻璃厚度。

### 2.2 幾何（Geometry）
- **連續圓角（continuous / squircle）**：所有圓角一律使用 `.continuous` style，不可用一般圓角。
- **同心圓角原則（Concentric）**：外層容器圓角大、內層元件圓角依比例縮小，邊距視覺一致（Apple HIG 同心律）。
- 收合圓角 = `idleH / 2`（完整藥丸）；展開圓角 = `DL.R.xl`。
- **懸浮陰影**：面板帶柔和投影，強化「浮在桌面上」的層次（注意不要過重，保持輕盈）。

### 2.3 色彩（Color）
- **固定暗色**（已定稿）：永遠暗色玻璃，不跟隨系統明暗 — Liquid Glass 在深色下質感最好、實作最單純。
- 強調色 accent1 / accent2（紫→藍漸層系，沿用 DropTarget 的 `#a78bfa`～`#38bdf8` 色域）。
- 文字階層：`t1`（主）→ `t4`（最弱），對應標題、內文、次要、時間碼。
- **玻璃強度：中等**（已定稿）— 展開時暗色蓋板 opacity ≈ 0.42–0.46，兼顧透明感與可讀性。

### 2.4 字體（Typography）
- 系統字（SF）為主；標題用 title 字重、內文用 body、時間碼用等寬 mono。
- 字級精簡：標題 13、副標 10、時間碼 9、提示 10–11。

### 2.5 動態（Motion）— Liquid Glass 的靈魂
- **流體彈簧**：展開/收合用 `spring(response: 0.42, dampingFraction: 0.86)`，要有「果凍般」的滑順收放。
- **形變（Morph）而非淡出**：尺寸由單一 frame 同時驅動寬高動畫，內容隨之 morph，不可有寬黑條殘留或 700ms 延遲。
- **內容轉場**：分頁/拖曳內容用 opacity + 微小 y 位移（±3pt），duration ≈ 0.16。
- **光隨動而走**：高光、彩色細線、邊框皆隨展開狀態淡入淡出（純色動畫，保證不黑頻）。
- **狀態切換無縫**：閒置↔hover↔拖曳↔各分頁之間轉換要連貫，不可閃爍或跳格。

---

## 3. 視窗與架構規格

| 參數 | 值 | 說明 |
|---|---|---|
| 視窗型別 | `NSPanel` | `.borderless` + `.nonactivatingPanel` |
| 視窗層級 | `.screenSaver` | 蓋過一般視窗 |
| collectionBehavior | `.canJoinAllSpaces`, `.fullScreenAuxiliary` | 跨 Space、全螢幕輔助 |
| 背景 | `.clear`、`isOpaque=false`、`hasShadow=false` | 透明，陰影自繪 |
| 展開寬 `openW` | 460 | |
| 視窗高 `panelH` | 340 | 取最高面板狀態(300)+邊距；縮小覆蓋桌面範圍 |
| 閒置高 `idleH` | = 安全區上緣（無瀏海則 37） | |
| 閒置寬 `idleW` | = 瀏海實際寬（無瀏海則 162） | 有音樂時 +92 露出封面與音波 |
| 位置 | 螢幕頂端置中 | `x = midX - openW/2`, `y = maxY - panelH` |

### 3.1 點擊穿透（click-through）核心規則 ⭐
這是「靠近 notch 的檔案抓不起來/移不動」問題的關鍵，必須嚴格遵守：

| 狀態 | 行為 |
|---|---|
| 點在**可見瀏海/展開面板**範圍內 | 正常互動（按鈕、滑桿、hover） |
| 點在**透明區**且**沒有在拖曳檔案** | `hitTest` 回 `nil` → 事件穿透 → 桌面/Finder 收到 → 檔案可點可抓可移動 |
| **正在拖曳檔案**（系統拖曳剪貼簿帶有 file URL） | 整欄捕捉 → 作為 drop 目的地 |

- 判斷「拖曳中」**只能用系統拖曳剪貼簿**（`NSPasteboard(name:.drag)` 的 changeCount 比對），
  **不可**用 `NSEvent.pressedMouseButtons`（按下尚未拖曳就誤判，會擋住抓檔）。
- 啟動時把當下 drag pasteboard 的 changeCount 記為「已處理」，避免殘留資料造成開機即誤擋。
- 每次拖曳結束（mouseUp）都要把 changeCount 標記為已處理，恢復穿透。

---

## 4. 狀態機（States）

```
              hover 0.16s            偵測到檔案拖曳
  [閒置藥丸] ───────────► [展開·分頁] ◄───────────── [展開·拖曳格]
      ▲   ▲                  │   │                        │
      │   └── 0.6s 無 hover ──┘   │ 點 ✕ / 0.6s 離開        │ 放開
      │                          ▼                        ▼
      └──────────────── 收合 ◄──── 1.2s 後（除非 keepOpen）─┘
```

| 狀態 | 觸發 | 內容 |
|---|---|---|
| 閒置藥丸 | 預設 | 瀏海外觀；有音樂時左封面＋右音波 |
| 展開·分頁 | 滑入 0.16s | 音樂／轉檔分頁＋頂部 Tab bar＋關閉鈕 |
| 展開·拖曳格 | 全域偵測到檔案拖曳 | 四個落點格＋提示文字 |
| 收合 | 離開 0.6s／拖曳結束 1.2s／點關閉 | 回藥丸；`keepOpenForConvert` 為真時不收 |

---

## 5. 功能規格

### 5.1 音樂控制 🟡

**資料來源**
- 主來源：Spotify 的 `DistributedNotification`（`com.spotify.client.PlaybackStateChanged`）→ 免權限取得 曲名／演出者／專輯／長度（注意 Duration 為**毫秒**，需 /1000）。
- 補強：AppleScript（`with timeout of 3 seconds`）→ 取**專輯封面 URL**、**精準播放位置**、trackId（需自動化授權）。
- Apple Music：MediaRemote。
- 兩來源自動切換（`.spotify` / `.mediaRemote` / `.none`）。

**顯示**
- 曲名（lineLimit 1）、`演出者 · 專輯`（lineLimit 1）。
- 專輯封面：44×44，連續圓角，細白邊；無封面時顯示漸層＋音符 placeholder。
- 播放時右側音波跳動，暫停時靜止。

**進度條（可 seek）**
- 顯示 `已播放 / 總長度` 時間碼（mono）。
- 可拖拉：Spotify → `SpotifyController.seek(to:)`（AppleScript）；Apple Music → MediaRemote `setElapsed`。
- 拖拉時 thumb 放大（10→14）、軌道加粗（3→4）。
- 進度每 ~0.5s 內插平滑前進，每 ~1s 對時校正。

**控制列**
- 上一首 / 播放暫停（主鈕，漸層底）/ 下一首 — 皆須真正有效。
- 透過 MediaRemote `SendCommand`（Spotify 免權限）。

**音量**
- 滑桿調整系統輸出音量（CoreAudio，免權限）。

**樣式要求** ⭐
- 音樂面板**精簡**：只放 封面列＋進度條＋控制列，展開高度 ≈ 240，不要留白過多。

---

### 5.2 拖曳檔案操作 🔴 ⭐（最高優先，目前失效）

拖曳檔案靠近瀏海 → 面板展開出現**四個落點格**，放到對應格執行：

| 格子 | rawValue | 功能 | 落點行為 | 完成回饋 |
|---|---|---|---|---|
| **NotchClip** | 0 | 暫存 | 複製到 `~/Downloads/NotchClip/`（重名自動加序號），並把最後一個寫入剪貼簿 | 格子變綠 2s |
| **Convert** | 1 | 格式轉換 | 載入檔案、自動切到轉檔分頁、保持展開 | 同上 |
| **AirDrop** | 2 | 無線傳送 | `NSSharingService(.sendViaAirDrop)`；暫時切 `.regular` 讓選人面板前景顯示，0.6s 後切回 `.accessory` | 同上 |
| **iCloud** | 3 | 上傳雲端 | 複製到 iCloud Documents（重名加序號） | 同上 |

**技術要求（為何不能用 SwiftUI `.onDrop`）**
- 面板是「偵測到拖曳後才展開並加入格子」，但 macOS 在拖曳「進入視窗」當下就鎖定 drop 目的地清單，事後加入的 SwiftUI onDrop 收不到 drop（檔案彈回）。
- 解法：在**視窗建立時就存在**的 `NotchHostingView` 上 `registerForDraggedTypes([.fileURL])`，由 AppKit 層接管 `draggingEntered/Updated/prepareForDragOperation/performDragOperation`，依游標 x 換算落在第幾格。

**落點判定**
- 四格等寬：`cellWidth = (openW - 2*14 - 3*8) / 4`，水平內距 14、格間距 8。
- drop 時依 x 算 cellIndex；**整欄高度皆可接收**（避免放開時落在死區彈回）。

**已修復並實測通過（2026-06-11 demo 驗證）** 🟢
1. ✅ 隱形屏障根治：視窗閒置時縮到藥丸大小（213×42pt），桌面完全不被覆蓋。
2. ✅ 靠近 notch 的檔案可正常點選、拖移（實機拖曳驗證）。
3. ✅ drop 不再彈回：真實拖曳 `draggingEntered` 觸發、`performDrop` 四格皆回傳 true
   （NotchClip 落檔、Convert 轉出 JPG/HEIC/45頁PNG、AirDrop 服務啟動、iCloud 落檔）。
4. ✅ 拖曳影像在面板上方：視窗層級 .screenSaver→.popUpMenu(101) < 系統拖曳層(~500)。
5. ✅ 拖曳偵測三重保障：DragCoordinator 全域監聽（快速路徑）＋ 0.25s 拖曳剪貼簿輪詢（備援）
   ＋ AppKit draggingEntered（最終防線）。

---

### 5.3 轉檔（Convert）規格 ✅ 已定稿

支援三類（**不含音訊**）：

| 類型 | 來源 → 目標 | 引擎 |
|---|---|---|
| 圖片 | HEIC/PNG/TIFF/JPG 互轉、→ WebP | ImageIO / `NSImage` |
| 影片 | MOV → MP4、短片 → GIF | AVFoundation 匯出 |
| PDF/文件 | 圖片 → PDF、PDF → 圖片（逐頁） | PDFKit / ImageIO |

- 自動依放入檔案類型，提供合理的目標格式選單（圖片給圖片目標、影片給影片目標…）。
- 多檔放入時**逐一轉換**（見 §5.2 多檔規則），逐檔顯示進度與結果。
- 轉檔 UI：放入後顯示來源資訊＋目標格式選擇＋轉換按鈕，展開高度 ≈ 300。
- 輸出位置：與原檔同目錄（重名自動加序號），或可改 `~/Downloads`（待小決定）。

---

## 6. 互動與手勢

| 互動 | 行為 |
|---|---|
| 滑入瀏海 0.16s | 展開分頁 |
| 移開 0.6s | 收合（拖曳中或 keepOpen 時不收） |
| 點 Tab | 切換 音樂／轉檔（easeInOut 0.18） |
| 點 ✕ | 立即收合 |
| 拖檔靠近 | 展開拖曳格 |
| 放到格子 | 執行操作＋回饋；Convert 後自動切分頁並保持展開（5s 備援收合） |
| 拖進度條 | seek |
| 拖音量條 | 調音量 |

---

## 7. 動畫清單

| 動畫 | 參數 |
|---|---|
| 展開/收合 | `spring(0.42, 0.86)` |
| 有音樂寬度變化 | `spring(0.5, 0.85)` |
| 分頁切換 | `easeInOut(0.18)` |
| 內容轉場 | opacity + y±3, `easeInOut(0.16)` |
| 閒置音樂指示淡入 | `easeInOut(0.25)` |
| 進度條 thumb 縮放 | `interactiveSpring(0.18)` |
| **黑頻** | **零容忍** — 毛玻璃常駐、只動蓋板 opacity |

---

## 8. 權限

| 功能 | 權限 | 狀態 |
|---|---|---|
| Spotify 曲目元資料（基本） | 免（DistributedNotification） | 🟢 |
| Spotify 封面/精準位置/seek | 自動化授權 | 🟢 已授權 |
| 播放控制 | 免（MediaRemote SendCommand） | 🟢 |
| 系統音量 | 免（CoreAudio） | 🟢 |
| AirDrop | 免（NSSharingService） | 🟡 |
| iCloud 容器 | App entitlement | 🟡 |
> 注意：面板在 `.screenSaver` 層會蓋住系統授權框，需要授權時應先讓出前景。

---

## 9. 驗收標準（Definition of Done）— 2026-06-11 實測

- [x] 靠近 notch 的桌面檔案可正常點擊、選取、拖動、移動位置。（閒置視窗僅 213×42pt）
- [x] 拖檔靠近瀏海面板會展開出四格。（真實拖曳，輪詢偵測 + 面板快照驗證）
- [x] 檔案放到 NotchClip/Convert/AirDrop/iCloud 任一格皆能完成操作，不彈回。
      （log：performDrop idx 0/1/2/3 全部 true；落檔全數驗證）
- [x] 放下後格子顯示成功（綠）/失敗（紅）回饋。
- [x] 音樂：曲名、演出者、專輯、封面（iTunes 備援，免權限）、真實時長、位置內插皆正確。
      （seek 需 Spotify 自動化授權；播放控制走 MediaRemote 免權限）
- [x] 轉檔實測：PNG→JPG、PNG→HEIC、3×PNG→合併3頁PDF、45頁PDF→45×PNG(2x)、
      MOV→GIF(原生)、MOV→MP4 全部成功，輸出與原檔同目錄不覆蓋。
- [x] 展開/收合呈現 Liquid Glass 流體質感（高光、邊框、折射色調、連續圓角、懸浮陰影）。
- [x] 固定暗色（§10 決策），玻璃蓋板 0.44。
- [ ] 黑頻：機制未變（毛玻璃常駐），請使用者於日常使用再確認。

---

## 10. 決策紀錄（已定稿）

| # | 項目 | 決定 |
|---|---|---|
| 1 | 轉檔格式 | ✅ 圖片互轉、影片轉檔、PDF/文件（**不含音訊**）— 見 §5.3 |
| 2 | Liquid Glass 強度 | ✅ 中等（蓋板 opacity 0.42–0.46） |
| 3 | 明暗模式 | ✅ 固定暗色 |
| 4 | NotchClip 暫存位置 | ✅ `~/Downloads/NotchClip` |
| 5 | 多檔拖曳 | ✅ 全部處理（NotchClip/iCloud/AirDrop 全收；Convert 逐一轉） |

### 仍可後續討論（非阻塞）
- 轉檔輸出位置：與原檔同目錄 vs `~/Downloads`。
- 是否新增其他瀏海元件：截圖、剪貼簿歷史、計時器、電量/網速等。
```
