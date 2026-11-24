# MultiVR Plugin

Unreal Engine 5 插件，提供 **大空間多人 VR（Location-Based VR）** 所需的定位、連線與 XR 週邊整合。
針對 Meta Quest 3 / 3S / Pro（Android arm64）與 Win64 編輯器開發環境設計。

本插件以 **預編譯（不含原始碼）** 形式交付，可在編輯器中使用、打包 Android APK，也可建置 Win64 content server。

> **第一次接觸本系統？** 先讀 [先認識這個系統](#先認識這個系統)，10 分鐘內就能知道整套系統怎麼運作、你要做哪些事。後面的章節是安裝步驟與技術參考，需要時再查。

主要能力：

- **Aruco / Markerless SLAM 重定位**：讀取頭盔左右眼透視相機影像，以 OpenCV + Aruco（或 RTAB-Map markerless）計算世界座標，把 VR 場景對齊到真實場地。
- **WebSocket / UDP 連線**：與後台（Local Server）交換房間、玩家位置、遊戲狀態、緊急事件等訊息。
- **鄰近玩家同步**：以 `proximity_update` 廣播他人位置，供 avatar 或防碰撞提示使用。
- **Meta XR 整合**：Passthrough 動態透明度（靠近牆、柱自動開透視）、Eye Tracking、Face Tracking。
- **效能控制**：藍圖一個節點調整定位頻率、log 等級、網路收送頻率。
- **WebRTC 串流信令**：頭盔畫面串流給營運端的 offer / answer / ICE 交換。
- **編輯器工具**：Galaxy 素材庫瀏覽 / 下載 / 匯入（EUW + `MultiVREditor` 模組）。

---

## 目錄

- [先認識這個系統](#先認識這個系統)
- [環境需求](#環境需求)
- [安裝步驟](#安裝步驟)
- [目錄結構](#目錄結構)
- [模組架構](#模組架構)
- [快速開始](#快速開始)
- [API 參考](#api-參考)
- [事件（Event Dispatcher）](#事件event-dispatcher)
- [通訊協定](#通訊協定)
- [SLAM 定位流程](#slam-定位流程)
- [效能設定](#效能設定)
- [Meta XR 功能](#meta-xr-功能)
- [Android 打包注意事項](#android-打包注意事項)
- [Log 說明](#log-說明)
- [疑難排解](#疑難排解)

---

## 先認識這個系統

### 這套系統在做什麼

玩家戴著 Meta Quest 頭盔，在一個**真實的場地**裡走動，頭盔裡看到的是**虛擬場景**，而且虛擬的牆、柱子跟真實的牆、柱子位置完全重疊；多位玩家在同一個場地裡，彼此看得到對方的位置。

要做到這件事，需要解決三個問題，本插件都處理好了：

| 問題 | 怎麼解決 |
|------|----------|
| 頭盔怎麼知道自己在場地的哪裡？ | 場地牆上、地上貼了很多**黑白方塊圖案（ArUco marker）**，每一顆的位置都事先量好存在「地圖」裡。頭盔的相機看到 marker，就能算出自己的精確位置，並把虛擬場景對齊到真實場地。這個過程叫**定位** |
| 多位玩家怎麼看到彼此？ | 每台頭盔不斷把自己的位置傳給**後台**，後台再轉發給其他頭盔 |
| 遊戲怎麼開始、結束？ | 工作人員在電腦上的**操作頁面**按「開始」，後台通知每台頭盔進入遊戲；按「結束」就全部退出 |

### 系統裡有哪些東西

```
                        ┌────────────────────────────┐
  工作人員 ── 操作頁面 ──▶│ 後台（Local Server）        │◀── 存放：場地地圖、相機標定檔
                        │ 電腦上的伺服器，port 5000   │
                        └───┬─────────────┬──────────┘
                 WebSocket  │             │  WebSocket
            ┌───────────────┘             └──────────────┐
            ▼                                            ▼
  ┌─────────────────── 頭盔 ───────────────────┐   ┌──────────────────────┐
  │ Lobby App ──(啟動)──▶ 遊戲 App（Content）    │──▶│ 遊戲伺服器            │
  │ 待機畫面               ★使用本插件           │UDP│（Content Server）     │
  │                        相機看 marker → 定位  │   │ ★使用本插件           │
  └─────────────────────────────────────────────┘   │ 電腦上，port 7777     │
                                                    └──────────────────────┘
```

| 名詞 | 是什麼 | 誰負責 |
|------|--------|--------|
| **後台**（Local Server / fiveg-local） | 跑在場地電腦上的伺服器。管理房間、轉發玩家位置、提供地圖與標定檔、派發開始 / 結束指令 | 系統方提供 |
| **操作頁面** | 工作人員用來建立房間、開始 / 暫停 / 結束遊戲的網頁 | 系統方提供 |
| **Lobby** | 頭盔上的待機 App，開機自動啟動，等後台通知後啟動遊戲 App。**不使用本插件** | 系統方提供 |
| **Content**（遊戲 App） | 你們開發的 UE 遊戲，打包成 Android APK 裝在頭盔上。**使用本插件**做定位與連線 | **你們開發** |
| **Content Server**（遊戲伺服器） | 你們遊戲的 UE Dedicated Server，跑在電腦上，負責多人遊戲同步。**使用本插件**連後台 | **你們開發** |
| **ArUco marker** | 貼在場地上的黑白方塊圖案，每一顆有自己的編號 | 現場佈置 |
| **地圖**（Aruco 地圖） | 記錄每一顆 marker 在場地裡的位置，存在後台，用一個代號（`MapName`）取用 | 系統方量測建立 |
| **標定檔** | 頭盔相機的鏡頭參數，存在後台，插件自動下載 | 系統方提供 |
| **首次定位** | 遊戲開始時第一次對齊場地，決定虛擬場景的朝向。玩家要**正對牆上的 marker 站穩 1～2 秒** | 插件自動處理 |
| **Passthrough（透視）** | 頭盔顯示真實世界的畫面。插件會在玩家靠近牆、柱時自動打開，避免撞到 | 插件自動處理 |
| **MultiVRGIS** | 插件的主要入口（藍圖搜尋「Multi VRGIS」），所有連線、定位功能都從這裡呼叫 | — |
| **MultiVR Component** | 掛在玩家 Pawn 上的元件，負責把定位結果套用到玩家位置 | — |

### 一場遊戲從頭到尾發生什麼事

1. **開場前**：工作人員啟動後台；玩家的頭盔開機，自動進入 **Lobby** 待機畫面，Lobby 連上後台。
2. **建立房間**：工作人員在操作頁面建立房間、選擇地圖與要參加的頭盔，後台啟動**遊戲伺服器**。
3. **進入遊戲**：後台通知每台頭盔的 Lobby，Lobby 啟動**遊戲 App**，並把「遊戲伺服器位址、房間、地圖」交給遊戲。
4. **連線與定位**：遊戲 App 連進遊戲伺服器，並呼叫插件的 `StartSystem`。插件自動向後台下載標定檔與地圖、打開相機；玩家正對牆上的 marker 站穩，**1～3 秒內完成首次定位**，插件發出 `OnSystemReady` 事件。
5. **遊戲進行中**：插件持續用 marker 校正玩家位置，並把位置傳給後台、接收其他玩家的位置（`OnDetectNearbyUsers`）。工作人員可以在操作頁面開始、暫停遊戲，或發出緊急警報。
6. **結束**：工作人員按結束，插件發出 `WSMessage_CloseGame` 事件，遊戲 App 結束，頭盔回到 Lobby。

### 你要做的事（開發者清單）

| 步驟 | 做什麼 | 看哪一節 |
|------|--------|----------|
| 1 | 準備原始碼版 UE、設定 BuildId、放入插件 | [環境需求](#環境需求)、[安裝步驟](#安裝步驟) |
| 2 | 在玩家 Pawn 加上 **MultiVR Component** | [快速開始](#快速開始) |
| 3 | 在 Pawn 的 BeginPlay 讀取命令列參數、連遊戲伺服器、呼叫 `StartSystem` | [Lobby → Content 流程](#4-lobby--content-流程多人遊戲) |
| 4 | 綁定事件：定位完成、附近玩家、關閉遊戲、緊急警報 | [事件](#事件event-dispatcher) |
| 5 | 遊戲伺服器的 GameMode 呼叫 `StartConnectWebSocket`，回報進度與狀態 | [Lobby → Content 流程](#4-lobby--content-流程多人遊戲) 的 Content Server 端 |
| 6 | 打包 Android APK 與 Win64 Server | [Android 打包注意事項](#android-打包注意事項) |
| 7 | （選用）調整效能設定 | [效能設定](#效能設定) |

> 有現成範例可以照著做：**example 專案**（UE 5.8）的 `BP_XRPawn`（頭盔端）、`BP_XRGameMode`（遊戲伺服器端）已經把上面 2～5 步都接好了。

---

## 環境需求

| 項目 | 需求 |
|------|------|
| Unreal Engine | **原始碼版** UE 5.3.2 或 UE 5.8.4（不支援 Epic Launcher 版）。交付包分版本提供，請使用與引擎相同版本的包 |
| 引擎 BuildId | 引擎的 `Engine\Build\Build.version` 必須加上交付包指定的 BuildId（見[安裝步驟](#安裝步驟)） |
| 專案 | C++ 專案；打包 content server 需要 Server Target（`<專案>Server.Target.cs`） |
| 目標平台 | Android（arm64-v8a）為主，Win64 可在編輯器內開發；content server 只支援 Win64 |
| HMD | Meta Quest 3 / 3S / Quest Pro；HTC 裝置走 LBS 模式（`IsHTCLBS`） |
| 相依插件 | `MetaXR (OculusXR)`、`AndroidPermission`、`WebSockets`、`EyeTracker` |
| Android | SDK、JDK 17；NDK 版本依引擎：**UE 5.3.2 用 r25b、UE 5.8.4 用 r27c** |

---

## 安裝步驟

> 交付包的 `MultiVR` 資料夾內附 `README.md` 與 `AGENT_INSTALL.md`，內含該版本**實際的 BuildId**，請以交付包內的文件為準。

1. **設定引擎 BuildId**
   打開 `<引擎>\Engine\Build\Build.version`，在最後一個欄位後面加上一行（前一行要補逗號）：

   ```json
   	"BuildId": "<交付包 README 內的 BuildId>"
   ```

   每台要開這個專案的電腦都要做。原始碼版引擎每台電腦編譯出的 BuildId 都不同，外掛的編輯器 DLL 只能在 BuildId 相同的引擎上載入；固定成交付包的值，就不需要外掛原始碼。

2. **放入外掛**
   把整個 `MultiVR` 資料夾複製到 `<專案>\Plugins\MultiVR`。已有舊版時，先整個刪除再放入，不要混用新舊檔案。

3. **修改 Target**
   在專案的 Game Target（`<專案>.Target.cs`）與 Server Target（`<專案>Server.Target.cs`）建構式內都加上：

   ```csharp
   BuildEnvironment = TargetBuildEnvironment.Shared;
   ```

   沒加的話，打包 APK 或 content server 時會出現 `Missing precompiled manifest`。

4. **在 `.uproject` 啟用插件**：`MultiVR`，以及 `OculusXR`（Passthrough / Eye / Face Tracking 需要）。

5. **建置一次編輯器**
   依序執行（先引擎、再專案；只建專案的話，引擎的 BuildId 不會更新）：

   ```
   <引擎>\Engine\Build\BatchFiles\Build.bat UnrealEditor Win64 Development -WaitMutex
   <引擎>\Engine\Build\BatchFiles\Build.bat <專案>Editor Win64 Development -Project="<專案>.uproject" -WaitMutex
   ```

   第一行通常只會更新清單檔，約數十秒。完成後 `<引擎>\Engine\Binaries\Win64\UnrealEditor.modules` 的 BuildId 應該等於交付包的值。

6. **開啟專案**，確認沒有出現「模組遺失或由不同引擎版本建置」的對話框。

### 不要做的事

- **不要刪除 `Plugins\MultiVR\Intermediate`**：裡面是 Android 與 Server 的預編譯物件檔，刪了就無法打包。清理專案時只刪專案自己的 `Intermediate`、`Binaries`。
- 不要對 MultiVR 執行 Rebuild，也不要在出現「是否重新編譯 MultiVR」時按「是」：交付包沒有原始碼，重新編譯一定失敗。
- 不要手動修改 `Plugins\MultiVR\Binaries\Win64\UnrealEditor.modules`。
- 不需要另外下載或更新 `Include` / `Library`：交付包已包含與預編譯檔相符的版本，自行替換可能造成連結錯誤或閃退。

---

## 目錄結構

```
Plugins/MultiVR/
├─ MultiVR.uplugin
├─ README.md / AGENT_INSTALL.md   本版本的安裝說明（含 BuildId）
├─ Binaries/Win64/                編輯器 DLL（預編譯）
├─ Intermediate/Build/            Android / Win64 / Server 預編譯物件檔（不可刪）
├─ Content/                       Curve、Materials、EUW（編輯器工具 Widget、Galaxy 素材庫）
├─ Include/ , Library/            第三方標頭檔與靜態庫（OpenCV / Aruco / RTAB-Map / PCL / Boost，已包含在交付包內）
└─ Source/
   ├─ MultiVR/Public/             MultiVRGIS.h / MultiVRComponent.h / MultiVRDatas.h / MultiVRPluginSettings.h
   ├─ MultiVR/Android/            MultiVRManifest_APL.xml(權限、WebRTC aar、Java 注入)
   ├─ MultiVR/Java/com/multivr/   相機橋接(CameraBridge / CameraViewModel)
   ├─ ThirdParty/Public/          SLAM 介面標頭
   ├─ PlatformUtility/Meta/       MetaXR 膠水層(含原始碼,由引擎編譯)
   └─ MultiVREditor/Public/       編輯器模組標頭
```

`MultiVR`、`ThirdParty`、`MultiVREditor` 三個模組不含 `Private` 原始碼。

---

## 模組架構

| 模組 | 型別 | 平台 | 說明 |
|------|------|------|------|
| `MultiVR` | Runtime（預編譯） | Android / Win64 | 核心：`UMultiVRGIS`（GameInstanceSubsystem）、`UMultiVRComponent`（SceneComponent） |
| `ThirdParty` | Runtime（預編譯） | Android / Win64 | SLAM 演算法與 OpenCV / Aruco / RTAB-Map / PCL / Boost 靜態庫連結；Android 額外載入 `libomp.so` |
| `MultiVRMetaModule` | Runtime（含原始碼） | 需 OculusXR | Passthrough 透明度、Eye / Face Tracking 封裝 |
| `MultiVREditor` | Editor（預編譯） | — | Galaxy 素材庫（登入、分類、下載、FBX 匯入）、編輯器通知、Http 封裝 |

執行期核心關係：

```
UMultiVRGIS (GameInstanceSubsystem)          UMultiVRComponent (掛在 VR Pawn 上)
 ├─ WebSocket / UDP 連線與收發               ├─ 訂閱定位結果 → 修正 Pawn 位置 / 旋轉
 ├─ 左右眼相機影像 → SLAM → 定位結果廣播      ├─ 每 0.05s 送出頭盔 transform
 ├─ Aruco 地圖參數（marker 尺寸、柱子、危險區） ├─ Recenter 偵測與重新定位、柱子重定位閃爍
 ├─ 效能設定（ApplyPerfSettings）             └─ 自動掛載 Meta 三個元件
 └─ Meta Passthrough 透明度 / Eye / Face
```

---

## 快速開始

### 1. 在 VR Pawn 上加入元件

於 VR Pawn（含 `UCameraComponent` 的 Pawn）加入 **MultiVR Component**。
元件會在第一次定位成功時自動抓取 Camera 並把 Pawn 設為 Aruco 中心 Actor。

### 2. 啟動系統

Blueprint 取得 `MultiVRGIS`（Get Game Instance Subsystem → MultiVRGIS，或直接搜尋「Multi VRGIS」getter）後呼叫：

```
StartSystem(
    IP        = "192.168.9.223",   // 後台電腦的 IP（正式流程由 Lobby 傳入 -wsserverip）
    Port      = "5000",            // 後台的 port（-wsport）
    ClientId  = "",                // 留空：自動用頭盔的 Wi-Fi IP 當頭盔 ID（建議一律留空）
    MapName   = "MapCodeName",     // 場地地圖的代號，要跟後台存的一樣（-mapName）
    RoomID    = "1234",            // 房間代號（-roomId）
    IsHTCLBS  = false,             // Quest 一律 false；true 只用於 HTC LBS 裝置
    SlamType  = ARUCO,             // 用 marker 定位填 ARUCO（MARKLESS / MIX 為無 marker 定位）
    HMDType   = "quest3s"          // 頭盔型號，用來向後台取對應的相機標定檔
)
```

呼叫之後插件會自動做完以下所有事，不需要另外寫程式：連上後台 → 下載相機標定檔 → 下載地圖 → 打開相機開始定位 → 首次定位完成後發出 `OnSystemReady`。

### 3. 綁定事件

在 Content 關卡（玩家 Pawn 的 BeginPlay）至少綁定以下事件：

| 事件 | 什麼時候發生 | 你通常要做什麼 |
|------|--------------|----------------|
| `OnSystemReady` | 首次定位完成 | 顯示開始畫面、允許玩家開始遊戲 |
| `OnDetectNearbyUsers` | 收到其他玩家的位置（持續觸發） | 用 `Clientid` 生成 / 移動其他玩家的角色 |
| `WSMessage_CloseGame` | 工作人員結束遊戲 | `Quit Game`，頭盔會回到 Lobby |
| `OnEmergencyAlert` / `OnEmergencyClear` | 工作人員發出 / 解除緊急警報 | 顯示 / 清除警報畫面（見[緊急警報 UI 建議](#緊急警報-ui-建議)） |
| `OnCameraBlock` | 頭盔相機被遮住 | 提示玩家不要遮住頭盔前方 |

### 4. Lobby → Content 流程（多人遊戲）

系統分成三個角色：

| 角色 | 執行於 | 說明 |
|------|--------|------|
| **Lobby**（`com.quest.lobby`） | 頭盔 | 獨立的 Android App（**不使用本插件**），開機自動啟動，等後台派發遊戲後用 Android Intent 啟動 Content APK |
| **Content** | 頭盔 | 使用本插件的 UE 遊戲 APK，以 `vr_headset` 身分連後台、以 client 身分連 content server |
| **Content Server** | Win64 Dedicated Server | 使用本插件的 UE 遊戲伺服器，以 `game_backend` 身分連後台，由後台啟動 |

流程：

1. 後台開始遊戲時，Lobby 收到 `headset_game_backend_command`（`command=connect`）：

   ```json
   {
     "type": "headset_game_backend_command",
     "command": "connect",
     "connectData": {
       "roomId": "room1",
       "ip": "192.168.9.102",
       "port": 7777,
       "packageName": "com.YourCompany.example",
       "mapName": "tt"
     }
   }
   ```

   Lobby 關閉自己的 WebSocket（讓出共用的 `clientId`），約 3 秒後用 Android Intent 啟動 `packageName` 指定的 Content APK，把參數組成 UE 命令列放進 Intent extra **`cmdline`**（必須全小寫）。效果等同下面這行 adb 指令：

   ```bat
   adb shell "am start -S -n com.YourCompany.example/com.epicgames.unreal.GameActivity --es cmdline '-wsserverip=192.168.9.102 -wsport=5000 -mapName=tt -roomId=room1 -serverip=192.168.9.102:7777'"
   ```

   | 參數 | 來源 | 用途 |
   |------|------|------|
   | `-serverip` | `connectData.ip` + `port` | content server 位址，Content 用 `open` 連進去 |
   | `-wsserverip` / `-wsport` | Lobby 連的後台位址 | `StartSystem` 的 IP / Port |
   | `-roomId` | `connectData.roomId` | `StartSystem` 的 RoomID |
   | `-mapName` | `connectData.mapName` | `StartSystem` 的 MapName（Aruco 地圖代號） |

   > **不經過 Lobby 測試**：content server 跑起來後，直接用上面的 adb 指令（換成自己的 package、IP、房間、地圖）就能模擬 Lobby 啟動遊戲。`-S` 會先關掉已在執行的 App 再重開。

2. **Content 啟動後由專案自行讀取這些參數**（插件不會自動解析）：Blueprint 用 `Get Command Line` → `Parse Command Line` 拆成 Key/Value，再用 Key 找值（C++ 可用 `FParse::Value`）。沒帶參數時用預設值，方便單機測試。
3. **Pawn 的 BeginPlay**（每次進關卡都會跑）：先綁定 MultiVR 事件（`OnSystemReady`、`WSMessage_CloseGame` 等），再依情況二選一：
   - **有帶 `-serverip`，而且目前是單機（`Is Standalone`）** → 執行 Console Command `open <serverip>`，連進 content server。
   - **否則，如果插件還沒連線（`IsConnected()` = false）** → 呼叫 `StartSystem`：
     - `IP` / `Port` = `-wsserverip` / `-wsport`
     - `RoomID` = `-roomId`
     - `MapName` = `-mapName`
     - **`ClientId` 留空**：插件會用裝置 IPv4，與 Lobby 使用的身分相同，後台才對得上同一台頭盔。
4. `open` 會換關卡，**Pawn 重生、BeginPlay 再跑一次**。這時已經是 client（不是單機），所以改走 `StartSystem`。
   - 事件每次都要重新綁定；`StartSystem` 只會在第一次（`IsConnected()` = false）時呼叫。
   - 沒帶 `-serverip`（單機測試）時第一次就直接 `StartSystem`，留在目前關卡。
5. 插件自動完成標定、地圖、首次定位 → **`OnSystemReady`**，遊戲可以開始。
6. 插件會自動回報 content server 連線狀態（`headset_status.contentServer`）：
   - 進入 content server 的關卡 → `connected`
   - 連不上 → `failed`（附錯誤原因）
   - 連上後斷線，或回到單機關卡 → `disconnected`
7. 後台結束遊戲時，Content 收到 `headset_game_backend_command`（`command=disconnect`），插件自動回覆 `game_close` 並廣播 **`WSMessage_CloseGame`**。**請在這裡結束 App（`Quit Game`）**，Lobby 會回到前景並重新連線後台。

> 完整的 Blueprint 範例見 example 專案的 `BP_XRPawn`（頭盔端）、`BP_XRGameMode`（Content Server 端）與 `BFL_GetArg`（讀命令列參數）。
>
> Lobby 與 Content 在同一台頭盔共用同一個 IPv4 作為 `clientId`，同一時間只能有一方連著後台；Lobby 會在啟動 Content 前先斷線，Content 結束後再重連。Lobby 的細節（安裝、watchdog、kiosk 設定）見 Lobby 專案的 README。
>
> `WSMessage_StartGame` 是舊流程（Lobby 也用本插件時）使用的事件，目前的 Lobby App 架構下 Content 不需要綁定。

**Content Server 端**：

- 啟動範例：`exampleServer.exe /Game/<地圖> -port=7777 -ip=<本機區網 IP> -wsserverip=<後台 IP> -wsport=5000 -roomId=<房間> -headsetIds=<逗號分隔>`。地圖要放第一個參數，否則會載入空的預設地圖。
- **`-ip` 不能空**：後台收到空 ip 會忽略這台 content server，房間綁不上，開始 / 暫停都沒反應。
- GameMode 的 BeginPlay（只在 Dedicated Server 上執行）：綁定事件後呼叫 `StartConnectWebSocket`（IP / Port / RoomID / HeadsetIds 由命令列帶入，`ClientID` 留空）。插件以 `game_backend` 身分連線，並把 ip / port 回報給後台。
- **房間要先建立，content server 才能啟動**。
- 用 `WSSend_Progress` 回報載入進度、`WSSend_GameStatus` 回報遊戲狀態。
- 綁定 `OnWebAction_StartGame` / `OnWebAction_PauseGame` 接收後台的開始 / 暫停。
- 後台送 `close_content_server` 時插件會直接結束程式。

### 5.（選用）效能設定

需要省 CPU 或調整 log 時，呼叫 `ApplyPerfSettings`，見[效能設定](#效能設定)。不呼叫則維持預設行為。

---

## API 參考

### UMultiVRGIS（GameInstanceSubsystem）

#### 連線

| 函式 | 說明 |
|------|------|
| `StartSystem(IP, Port, ClientId, MapName, RoomID, IsHTCLBS, SlamType, HMDType)` | 一鍵啟動：設定 SLAM 類型 + 連線 WebSocket |
| `StartConnectWebSocket(IP, Port, ClientID, RoomID, HeadsetIds)` | 只建立主連線 |
| `StartConnectMultiWebSocket(IP, ServerPort, ClientID, RoomID, IsGameServer)` | 建立多條連線（後台 / 多頭盔測試用），存於 `WebSockets` 陣列 |
| `WebSocketSend(Text)` / `CloseWebSocket()` / `CloseAllWebSockets()` / `IsConnected()` | 基本收送與關閉 |
| `GetRoomID()` / `GetClientID()` / `GetIsHTCLBS()` / `SetIsHTCLBS(bool)` | 狀態查詢與設定 |
| `GetHeadsetIDBySN(SN)` | 以頭盔序號查詢 HeadsetID |

斷線時會自動重試，重連成功走 `WSOnReConnected`。

#### 資料傳送

| 函式 | 送出 type | 用途 |
|------|-----------|------|
| `WSSend_HeadSetData(Position, Rotation)` | `transform_update` | 頭盔位置/朝向（**經 UDP 7000 送出**，單位由 cm 轉為 m） |
| `WSSend_HeadSetDataToIndex(SocketIndex, HeadsetID, Pos, Rot)` | `transform_update` | 指定第 N 條 WebSocket 送出（多頭盔模擬） |
| `WSSend_Progress(Percent)` | `game_progress` | 遊戲載入進度（GameServer） |
| `WSSend_GameStatus(EGameStatus)` | `game_status` | `GAME_IDLE / GAME_PLAYING / GAME_END / GAME_PAUSE` |
| `SendEmergencyAlertAck(AlertId)` | `emergency_alert_ack` | 收到警報時已自動回覆，必要時可手動再送 |

#### SLAM / 相機

| 函式 | 說明 |
|------|------|
| `ActiveCamera()` | 開啟頭盔透視相機 |
| `InitSlam_Aruco(ArucoSize, MaxDetectLength)` | 初始化 Aruco SLAM（系統自動呼叫，預設 0.40 m / 4 m；marker 尺寸實際以地圖檔為準） |
| `SetOutlierFilterParam(RejectDistCm, ConfirmCount, HistoryLen)` | 水平離群過濾參數（預設 100 cm / 3 次） |
| `SetOutlierFilterParamZ(RejectDistZCm)` | 高度離群過濾門檻（預設 30 cm） |
| `GetCameraTexture()` / `GetCameraData(W,H)` / `UpdateCameraTexture()` | 取得相機畫面 Texture2D（640×480 BGRA），可做除錯顯示 |
| `SetBaseOffsetInMeters()` / `SetBaseRotationInMeters()` | 調整 XR 基準位移與旋轉 |
| `GetZoneState(Pos)` | 回傳位置相對危險區（牆、柱）的狀態：`Safe` / `Warning` / `Danger` |

#### 效能

| 函式 | 說明 |
|------|------|
| `ApplyPerfSettings(bSlamPaused, SlamInterval, PluginLogLevel, UploadHz, RemoteHz)` | 一次設定全部效能參數，見[效能設定](#效能設定) |
| `GetPerfSettings(...)` | 讀回目前設定與網路狀態（往返延遲、伺服器停留時間、每秒收送筆數） |

#### WebRTC 串流

`ConnectWebRTC()`（送 `stream_request`）、`CloseWebRTC()` / `EndWebRTC()`（送 `stream_end`）、`CallOffer()`、`WebRTC_SendOffer/Answer/IceCandidate(...)`。

#### JSON 工具（皆為 BlueprintCallable）

`GetStringValue`、`GetNumberValue`、`GetBoolValue`、`GetJsonStringValue`、`GetJsonStringArrayValue`、`GetStringArrayValue`、`GetNumberArrayValue`、`GetFloatArrayValue`。
可直接在 Blueprint 解析 `WSOnMessage` 收到的原始字串。

### UMultiVRComponent（SceneComponent）

| 成員 | 說明 |
|------|------|
| `SetDetectInfo(DetectCD, InterpSpeed)` | 定位冷卻秒數（預設 0.5 秒）與位置插值速度（Android 預設 25，並依修正距離自動調整） |
| `SetPillarFlashParam(MinDistCm, CooldownSec)` | 柱子重定位閃爍條件：修正距離門檻（預設 100 cm）與冷卻（預設 3 秒） |
| `OnRelocalizeFlash`（BlueprintImplementableEvent） | 場景「直接拉過去」時觸發，接閃爍過場 Blueprint |
| `SetAutoZonePassthrough(bEnable)` | 開關「靠近危險區自動開透視」（預設開）。關閉時透明度回到 `SetMetaPassthroughOpacity` 設定的值 |
| `SetMetaPassthroughOpacity(Opacity)` | 手動設定 Passthrough 基礎透明度（0 = 關閉透視、1 = 全透視） |

內部狀態機 `ESlamState`：`Init`（首次定位，直接 Snap）→ `Waiting`（跨一幀）→ `Lerping`（持續平滑修正）。

---

## 事件（Event Dispatcher）

### 通用

| 事件 | 參數 | 觸發時機 |
|------|------|----------|
| `WSOnConnected` | — | WebSocket 連線成功 |
| `WSOnReConnected` | — | 斷線後重連成功 |
| `WSOnConnectClose` | — | 連線關閉 |
| `WSOnConnectError` | `Message` | 連線錯誤（會自動重試） |
| `WSOnMessage` | `Message` | 收到任何原始訊息（除錯 / 自訂協定用） |
| `WSMessage_CloseGame` | — | 後台要求關閉遊戲 |

### Lobby（頭盔端）

| 事件 | 參數 |
|------|------|
| `WSMessage_StartGame` | `roomid, execPath, ip, port`：舊流程用（Lobby 也使用本插件時）。目前 Lobby 是獨立 App，Content 改由命令列取得參數，見 [Lobby → Content 流程](#4-lobby--content-流程多人遊戲) |

### Content（頭盔端）

| 事件 | 參數 | 說明 |
|------|------|------|
| `OnSystemReady` | — | 首次定位完成、UDP 啟動，可開始遊戲 |
| `OnDetectNearbyUsers` | `TArray<FUser>` | 鄰近玩家（`Clientid` / `Position` / `Rotation` / `IsSameRoom`），已換算到 Aruco 中心座標系 |
| `OnCameraBlock` | — | 相機被遮蔽（SLAM 回傳 `block`） |
| `OnHMDLanguageSet` | `Language` | 後台下發語系 |
| `OnTemperatureGet` | `Temperature` | 場館氣溫 |
| `OnEmergencyAlert` | `FEmergencyAlertData` | 緊急警報（`AlertId` / `Level` / `Cause` / `Instruction` / `Timestamp`），已自動回 ack |
| `OnEmergencyClear` | `FEmergencyClearData` | 解除警報（`AlertId` / `ResumedRooms` / `Timestamp`），UI 應清除所有警報 |
| `OnUpdateEyeTracking` | `GazeData, StereoGazeData, IsSuccess` | Meta 眼動 |
| `OnUpdateFaceTracking` | `FaceState, IsSuccess` | Meta 表情（70 組 blendshape 權重） |

`EEmergencyLevel`：`NOTICE / WARNING / CRITICAL`
`EEmergencyCause`：`CUSTOM / FIRE / EARTHQUAKE / EVACUATION`

#### 緊急警報 UI 建議

| Level | 建議 UI |
|-------|---------|
| `CRITICAL` | 全螢幕強制遮罩 + 音效，玩家無法關閉 |
| `WARNING` | 警告 UI 覆蓋，遊戲暫停 |
| `NOTICE` | 角落通知，不中斷遊戲 |

- 顯示文字一律用 `Instruction`（後台已組好、含多語系），`Cause` 只用來決定圖示或音效。
- 收到 `OnEmergencyClear` 時**一律清除所有警報 UI**，`AlertId` 可能是空字串（強制清除），不需判斷。
- ack 已內建（收到 `emergency_alert` 立即回 `emergency_alert_ack`），content 不用處理。

```
[BeginPlay]
   Get MultiVRGIS
     ├─ Bind Event to On Emergency Alert → Switch on Level → 顯示對應 UI，Set Text ← Instruction
     └─ Bind Event to On Emergency Clear → 移除所有警報 UI
```

### GameServer（Dedicated Server）

| 事件 | 說明 |
|------|------|
| `OnWebAction_StartGame` | 後台按下「開始遊戲」 |
| `OnWebAction_PauseGame` | 後台按下「暫停遊戲」 |

---

## 通訊協定

### 送出（Client → Server）

| type | 觸發 |
|------|------|
| `transform_update` | `WSSend_HeadSetData`（UDP 7000）／`WSSend_HeadSetDataToIndex` |
| `game_progress` / `game_status` | `WSSend_Progress` / `WSSend_GameStatus` |
| `emergency_alert_ack` | 收到 `emergency_alert` 時自動送出 |
| `get_calibration` | 收到 `init_player_info` 後自動索取相機標定（帶 `deviceType` / `mapType`） |
| `get_map` | 標定取得後自動索取 Aruco 地圖（帶 `codeName`） |
| `game_close` | 收到 `headset_game_backend_command: disconnect` 時自動回覆 |
| `headset_tracking_update` | 首次定位完成、或 `resync_pose` 重新定位完成時回報 tracking 狀態 |
| `headset_status`（`arucoVisible` / `arucoAvgVisible`） | 每秒回報目前看到幾顆 marker 與過去 1 秒平均，供後台監看定位狀況 |
| `headset_status`（`contentServer` / `contentServerError`） | 頭盔與 content server 的連線狀態變化時回報：`connecting` / `connected` / `disconnected`（連上後斷線）/ `failed`（一開始就連不上）；後台連線或重連後也會補送一次 |
| `stream_request` / `stream_end` | WebRTC 串流開始 / 結束 |
| `signal_offer` / `signal_ice_candidate` / `headset_msg` | WebRTC 信令 |

### 接收（Server → Client）

| type | 處理 |
|------|------|
| `proximity_update` | 解析鄰近玩家 → `OnDetectNearbyUsers`（WebSocket 與 UDP 兩條路徑皆支援） |
| `headset_game_backend_command` | `command=connect` → `WSMessage_StartGame`；`command=disconnect` → 回 `game_close` + `WSMessage_CloseGame` |
| `start_game` / `pause_game` | → `OnWebAction_StartGame` / `OnWebAction_PauseGame` |
| `close_content_server` | 直接 `RequestExit` 並廣播 `WSMessage_CloseGame` |
| `init_player_info` | 取語系、氣溫；HTC LBS 直接就緒，否則進入標定流程 |
| `get_calibration_ack` | 存檔左眼 `metamap-cam.yml`、右眼 `metamap-cam-right.yml`（後台沒給右眼時，右眼沿用左眼標定） |
| `get_map_ack` | 存檔 `metamap.yml`、解析地圖參數並初始化 SLAM |
| `resync_pose` | 後台要求重新定位：重做一次首次定位以校正朝向，完成後回報 `headset_tracking_update` |
| `language_change` | → `OnHMDLanguageSet` |
| `emergency_alert` / `emergency_clear` | → 對應事件（alert 自動 ack） |
| `stream_accepted` / `signal_answer` / `signal_ice_candidate` / `stream_ended` / `stream_end_ack` | WebRTC 信令流程 |
| `headset_msg_res`（內含 `webrtc_offer` / `webrtc_answer` / `webrtc_icecandidate`） | 頭盔對頭盔的 WebRTC 信令轉送 |

---

## SLAM 定位流程

```
Java 相機橋接取左右眼影像（成對）
      ↓
SLAM：左右眼各自偵測 Aruco → 雙眼融合求相機位置
      ↓ JSON { pos, orient, pillar, yawPair … }
首次定位閘門 / 離群過濾
      ↓
UMultiVRComponent 修正 Pawn（首次 Snap，之後平滑 Lerp）
```

### 定位演算法重點

- **雙眼定位**：左右眼各自使用後台提供的相機內參（`get_calibration` 的左右眼標定），並套用相機到頭盔的外參，算出的是頭盔（眼睛）位置而不是相機位置。
- **單眼遮擋互補**：一眼被遮住或看不到 marker 時，自動改用另一眼的結果。
- **多 marker 聯合求解**：同時看到多顆 marker 時一起求解；只看到一顆時另外處理單顆 marker 的角度歧義，減少朝向翻轉。
- **延遲補償**：SLAM 計算需要時間，校正時會回看「拍照當下」的頭部位置再套用修正，走動中也不會被拉回舊位置。
- **原地死區**：玩家站著不動時忽略微小修正，避免畫面細微抖動；頭部移動或修正量變大時才套用（高度差也會解除死區）。

### 地圖決定的參數

下列參數由後台地圖檔（`get_map`）提供，不需要在專案中設定：

- **marker 尺寸**：由地圖中 marker 的實際邊長計算。
- **場地座標對齊**：`aruco_x_offset` / `aruco_y_offset` / `aruco_foward`，把地圖座標轉成 UE 場景座標。
- **柱子 marker**：地圖 `extra_info.pillar_ids`。
- **偵測靈敏度**：地圖 `extra_info.detect_thres / detect_window`（投影或低對比場地用；沒寫則用預設值）。
- **危險區（牆、柱）**：地圖 `danger_rects`，用於 Passthrough 自動透視與 `GetZoneState`。

### 首次定位（開場鎖定）

場景朝向只在首次定位時鎖定一次，鎖歪會整場歪，所以首次定位比之後的定位嚴格：

1. **只用牆面 marker**：地板 marker 算出的朝向不穩定，首次定位不採用。
2. **左右眼一致**：左右眼各自算出的朝向必須一致。
3. **頭部姿態**：頭要擺正（roll ≤ 15°、pitch ≤ 30°）。
4. **站定**：0.5 秒內朝向擺動小於 2° 才鎖定。
5. 首次定位期間**不受 `SlamInterval` 限速**，鎖定後才套用。

玩家開場時請**正對牆上的 marker 站穩 1～2 秒**。正常情況 1～3 秒內完成鎖定。

### 之後的定位

1. **取樣一致性**：同批取樣中若兩幀距離 ≥ 60 cm，整批作廢。
2. **離群過濾**：新定位與最近幾筆歷史的中位數比較：
   - 水平距離 ≤ 100 cm（`SetOutlierFilterParam`）且高度差 ≤ 30 cm（`SetOutlierFilterParamZ`）→ 接受；
   - 超過門檻 → 必須連續 3 次且彼此一致才承認為「真移動」，否則丟棄。
3. **柱子優先重定位**：該幀由柱子 marker 驅動時，若修正距離 > 100 cm 且已過冷卻，場景直接 Snap 並觸發 `OnRelocalizeFlash`；小幅修正仍走平滑 Lerp。
4. **Recenter / 重新定位**：玩家長按 Home 鍵 recenter 時，先把場景補償回 recenter 前的位置，再重新做一次首次定位校正朝向。後台送 `resync_pose` 也會觸發同樣的重新定位。
5. **定位冷卻**：每次定位成功後冷卻 `DetectCD`（預設 0.5 秒）。

---

## 效能設定

藍圖節點 **Apply Perf Settings**（Target 接 `Multi VRGIS`）一次設定以下 5 個參數。全部使用預設值 = 原本的行為，**不呼叫也不影響任何功能**。

| 參數 | 預設 | 說明 |
|------|------|------|
| `Slam Paused` | 不勾 | 勾選 = 暫停定位，只靠頭盔自身追蹤（會逐漸漂移） |
| `Slam Interval` | 0 | 兩次定位之間最少間隔幾秒。0 = 不限（約每秒 13 次）。**建議 0.3～0.5**；實測 0.5 秒時 CPU 約從 89% 降到 62% |
| `Plugin Log Level` | 2 | 0 = 全關、1 = 重要（建議正式上線）、2 = 全部（含每秒效能診斷） |
| `Upload Hz` | 0 | 頭盔位置每秒最多上傳幾次。0 = 不限（原本每秒 20 次） |
| `Remote Hz` | 0 | 其他玩家位置每秒最多處理幾次。0 = 全部處理 |

使用注意：

- **沒有呼叫順序的要求**：Start System 之前或之後、BeginPlay 或任何事件之後都可以，隨時呼叫立即生效。
- **每次呼叫都會設定全部 5 個值**，沒填的欄位會回到預設值。只想改其中一項時，其他欄位也要填回目前的值（可先用 **Get Perf Settings** 讀出）。
- 確認生效：logcat 會印 `[PERFCTRL] SlamPaused=… SlamInterval=…`。

---

## Meta XR 功能

首次定位完成時 `UMultiVRComponent` 會自動掛載三個元件：

| 元件 | 功能 |
|------|------|
| `UMultiVRMetaPassthroughComponent` | 依玩家與危險區（牆、柱）的距離動態調整 Passthrough 透明度：距邊界 40 cm 開始漸變，10 cm 內完全透視。可用 `SetAutoZonePassthrough(false)` 關閉 |
| `UMultiVRMetaEyeTrackingComponent` | 每幀取 Gaze / StereoGaze → `OnUpdateEyeTracking` |
| `UMultiVRMetaFaceTrackingComponent` | 每幀取 70 組表情權重 → `OnUpdateFaceTracking` |

`UMultiVRMetaXRMovementFuncLib::IsSupportEyeTracking() / IsSupportFaceTracking()` 可先行檢查裝置支援度。

---

## Android 打包注意事項

`MultiVRManifest_APL.xml` 會自動注入：

- 權限：`CAMERA`、`horizonos.permission.HEADSET_CAMERA`、`RECORD_AUDIO`、`MANAGE_EXTERNAL_STORAGE`
- 功能宣告：`android.hardware.camera2.full`
- 相依：`google-webrtc-1.0.32006.aar`（複製到 build/libs 並加入 gradle `implementation`）
- Java 端相機橋接（`com.multivr.CameraBridge` 等）

`ThirdParty_APL.xml` 於啟動時載入 `libomp.so`（OpenMP，未載入會 `UnsatisfiedLinkError` 閃退）。

其他：

- 只支援 **arm64-v8a**，Texture 格式建議 ASTC。
- NDK 版本依引擎：UE 5.3.2 用 r25b、UE 5.8.4 用 r27c。
- 執行期會先請求 Android 權限，權限未過前連線會先擱置，取得後自動續連。
- 標定檔與地圖檔存放於 `FPaths::ProjectPersistentDownloadDir()`（`metamap-cam.yml` / `metamap-cam-right.yml` / `metamap.yml`）。
- 若用 `-CookAll` 打包，可在 `DefaultGame.ini` 的 `[/Script/UnrealEd.ProjectPackagingSettings]` 加入 `+DirectoriesToNeverCook=(Path="/MultiVR/EUW")`，排除編輯器工具 Widget。

---

## Log 說明

以 `adb logcat | findstr <關鍵字>` 查看。`Plugin Log Level` 設為 0 時以下 log 都不會印（`[PERFCTRL]` 除外）。

| 關鍵字 | 內容 |
|--------|------|
| `[PERFCTRL]` | 每次呼叫 `ApplyPerfSettings` 時印出完整設定 |
| `[first-init]` | 首次定位前每秒一行：收到幾幀、各被哪一關擋下（無 marker / 地板 / 雙眼不一致 / 未站定）；鎖定時印「鎖定完成，耗時 X 秒」 |
| `[SURVEY-MK]` / `[SURVEY-FIX]` / `[SURVEY-POS]` | 定位品質巡檢用：每顆 marker 的偵測結果、每次定位的校正量、每秒頭部位置 |
| `[NETLAT]` | 每 10 秒印網路往返與伺服器停留時間（平均 / P95） |
| `[STEREOPERF]` / `[SLAMPERF]` | 每秒定位處理次數與耗時（Log Level 2 才印） |

---

## 疑難排解

| 症狀 | 可能原因 / 處理 |
|------|-----------------|
| 開專案時說 MultiVR 模組遺失、要求重新編譯 | 引擎 BuildId 不是交付包指定的值。檢查安裝步驟 1，再做一次步驟 5（兩行都要跑） |
| 打包時 `Missing precompiled manifest` | 安裝步驟 3 沒做，或 `Plugins\MultiVR\Intermediate` 被刪除 |
| Server 連結時 `LNK2011` / `LNK1120` | 使用了舊版交付包，請改用最新版 |
| 引擎版本與交付包不符 | 每個交付包只能用於對應的引擎版本（5.3.2 / 5.8.4），請索取對應版本 |
| 裝置一啟動就閃退，log 有 `UnsatisfiedLinkError: libomp.so` | `libomp.so` 未打包，確認 `Plugins\MultiVR\Library` 完整並重新打包 |
| `calibration not exist` / `aruco map not exist` | 後台沒有該 `deviceType` 的標定檔或該 `MapName` 的地圖，向後台確認代號 |
| 開場定位很久才完成 | 看 `[first-init]` log：「無 marker」或「地板」多 → 請正對牆面 marker；「未站定」多 → 請站穩；「雙眼不一致」多 → 靠近 marker 一點再試 |
| 走動時位置校正變慢 | `Slam Interval` 設太大，調小或設回 0 |
| 收不到 `proximity_update` | 確認 UDP 7000 未被防火牆阻擋，且首次定位已完成 |
| HTC 裝置定位無效 | HTC 走 LBS 模式，`StartSystem` 的 `IsHTCLBS` 必須為 `true`，此模式不執行 Aruco SLAM |
