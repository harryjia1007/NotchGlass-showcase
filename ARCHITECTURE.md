# NotchGlass 架構導讀 / Architecture Deep-Dive

> 本文件節錄 NotchGlass 的核心工程決策與關鍵程式碼，作為作品集的技術展示。
> Curated engineering highlights from the NotchGlass codebase.

## 專案結構

```
NotchGlass/
├── App/            App 入口與生命週期（含試用/授權閘門）
├── Core/           視窗管理、拖曳偵測、授權系統、設計系統
├── Features/
│   ├── Music/      音樂來源整合（Spotify/Apple Music）與面板
│   └── Convert/    轉換引擎與格式定義
├── UI/             瀏海容器、拖放格、付費牆
└── Resources/      圖示與 Info.plist
```

---

## 1. 零干擾視窗架構

**問題**：瀏海 widget 需要一個常駐視窗，但固定大視窗會形成「隱形屏障」——
桌面上靠近瀏海的檔案會被看不見的視窗擋住，無法拖曳。

**解法**：視窗尺寸跟著狀態走。閒置時縮到藥丸大小（只覆蓋瀏海本身），
偵測到檔案拖曳接近時才放大成捕捉區，面板收合後再縮回。

```swift
/// 閒置：只佔瀏海藥丸的範圍，桌面完全不被覆蓋
private func idleFrame() -> NSRect { /* pill-sized rect around the notch */ }

/// 展開/捕捉：放大到面板尺寸（340pt 高）
private func openFrame() -> NSRect { /* full panel rect */ }
```

另一個關鍵：**NSPanel 層級設在系統拖曳影像之下**（`.popUpMenu`，level 101），
所以使用者拖著的檔案縮圖永遠浮在面板上方，不會被面板蓋住。

## 2. 拖曳偵測三重保障

macOS 沒有「全域檔案拖曳開始」的公開 API。NotchGlass 用三條互補路徑：

1. **快速路徑** — 全域 `leftMouseDragged` 事件監聽（即時反應）
2. **備援** — 0.25s 輪詢拖曳剪貼簿（事件監聽收不到的情境）
3. **最終防線** — AppKit `draggingEntered`（游標真的進到視窗）

**防誤觸的核心**：拖曳剪貼簿在拖曳結束後仍殘留舊資料，導致「框選桌面」也會誤觸發。
解法是追蹤 `changeCount` 的「結算值」，只有數值改變才視為新的拖曳：

```swift
func dragHasFiles() -> Bool {
    let pb = NSPasteboard(name: .drag)
    guard pb.changeCount != settledChangeCount else { return false }  // 殘留資料 → 不是新拖曳
    return pb.canReadObject(forClasses: [NSURL.self],
                            options: [.urlReadingFileURLsOnly: true])
}
```

再配合「游標接近瀏海才展開、離開即收回」的距離判定，靈敏度剛剛好。

## 3. NSPanel 的第一下點擊問題

`NSPanel` 預設 `canBecomeKey = false`，第一下點擊會被吃掉當作視窗激活，
按鈕要點兩次才有反應。三件事一起修：

```swift
final class KeyableNotchPanel: NSPanel {
    override var canBecomeKey: Bool { true }
}

final class NotchHostingView: NSHostingView<AnyView> {
    override func acceptsFirstMouse(for event: NSEvent?) -> Bool { true }
}

panel.becomesKeyOnlyIfNeeded = true   // 不搶其他 App 的焦點
```

## 4. 免權限的音樂整合

三層資料來源，依權限需求由低到高：

| 層 | 技術 | 權限 | 提供 |
|---|---|---|---|
| 1 | `DistributedNotification`（Spotify 播放狀態廣播）| 免 | 曲名/演出者/位置，即時 |
| 2 | `MediaRemote` 私有框架 | 免 | 播放/暫停/切歌控制 |
| 3 | AppleScript | 需自動化授權 | 封面 URL、精準 seek |

AppleScript 在使用者未授權時會**無限懸置**（TCC 對話框 pending），
若用序列佇列會永久卡死。解法：並行佇列＋6 秒看門狗：

```swift
// busySince: Date? 取代 busy: Bool —— 超過 6 秒視為懸置，自動解鎖
if let since = busySince, Date().timeIntervalSince(since) < 6 { return }
```

封面圖另有 **iTunes Search API 備援**（免權限），使用者完全不授權也看得到封面。

## 5. 轉換引擎

- **執行期能力偵測**：`CGImageDestinationCopyTypeIdentifiers()` 查詢系統實際支援的編碼格式，而非寫死清單
- **原生 GIF**：`AVAssetImageGenerator` 逐幀 → `CGImageDestination` 合成，不依賴 ffmpeg
- **路由設計**：`(來源類別, 目標副檔名)` 雙維 switch，新格式只要加一條 case
- **防呆**：重複副檔名自動剝除（`報告.html.txt → 報告.html`）、輸出重名自動編號

## 6. 授權系統

- 3 天試用：首啟日期存 **Keychain**（重裝不歸零）
- 金鑰驗證：Gumroad License API，檢查 `uses` 實作**啟用台數上限**、拒絕已退款訂單
- 通過後金鑰存 Keychain，離線啟動不需重驗

```swift
guard json["success"] as? Bool == true else { return .invalid }
if purchase["refunded"] as? Bool == true   { return .refunded }
if let uses = json["uses"] as? Int, uses > maxActivations { return .tooManyActivations }
```

---

## 工程原則

1. **權限是 UX 成本** — 能免權限就免權限，授權只用來「增強」而非「啟用」功能
2. **每條關鍵路徑都有備援** — 拖曳偵測 ×3、封面來源 ×3、拖曳結束偵測 ×3
3. **狀態決定資源** — 視窗大小、輪詢頻率都跟著使用狀態縮放，閒置時近乎零成本

> 完整原始碼為私有。技術交流或面試展示請聯繫：harryjia1007@gmail.com
