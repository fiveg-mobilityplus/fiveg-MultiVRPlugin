# MultiVR Plugin

Unreal Engine 5 插件，提供 **大空間多人 VR（Location-Based VR）** 所需的定位、連線與 XR 週邊整合。
針對 Meta Quest 3 / 3S / Pro（Android arm64）與 Win64 編輯器開發環境設計。

主要能力：

- **Aruco / Markless SLAM 重定位**：直接讀取 HMD 透視相機影像，在 Android 端以 OpenCV + Aruco + RTAB-Map 計算世界座標，把 VR 場景對齊到真實場地。
- **WebSocket / UDP 連線**：與 MultiVR 後端（大廳、遊戲後端、營運後台）交換房間、玩家位置、遊戲狀態、緊急事件等訊息。
- **鄰近玩家同步**：以 `proximity_update` 廣播他人位置，供 avatar 或防碰撞提示使用。
- **Meta XR 整合**：Passthrough 動態透明度（走出場地邊界自動開透視）、Eye Tracking、Face Tracking。
- **WebRTC 串流信令**：頭盔畫面串流給營運端的 offer / answer / ICE 交換。
- **編輯器工具**：Galaxy 素材庫瀏覽 / 下載 / 匯入（EUW + `MultiVREditor` 模組）。

---

## 目錄

- [額外下載檔案（必做）](#額外下載檔案必做)
- [環境需求](#環境需求)
- [安裝步驟](#安裝步驟)
- [目錄結構](#目錄結構)
- [模組架構](#模組架構)
- [快速開始](#快速開始)
- [API 參考](#api-參考)
- [事件（Event Dispatcher）](#事件event-dispatcher)
- [通訊協定](#通訊協定)
- [SLAM 定位流程與調校](#slam-定位流程與調校)
- [Meta XR 功能](#meta-xr-功能)
- [Android 打包注意事項](#android-打包注意事項)
- [疑難排解](#疑難排解)

---

## 額外下載檔案（必做）

Include / Library 因體積龐大**未納入 Git**，Clone 之後必須另外下載並解壓縮，否則 Android 編譯會找不到標頭檔與靜態庫。

### === 更新 Include ===

1. 下載 https://drive.google.com/file/d/1QTqyZEEMALUr-rLC0KLCdpFL1AuSgw1J/view?usp=sharing
2. 索引到 `專案\Plugins\MultiVR\Include\Android\include` 解壓縮

### === 更新 library ===

1. 下載 https://drive.google.com/file/d/1sBz7Is1bqi5pGB7Mha1HX-KHheI1jMv8/view?usp=sharing
2. 索引到 `專案\Plugins\MultiVR\Library\Android\libs` 解壓縮並覆蓋

解壓縮後應具備下列結構（`ThirdParty.Build.cs` 依此路徑連結）：

```
Plugins/MultiVR/
├─ Include/
│  ├─ Android/include/       ← 第 1 包解壓於此
│  │   ├─ opencv2/  aruco/  rtabmap/  3rdparty/eigen3/  pcl/  boost/
│  └─ Win64/include/
└─ Library/
   ├─ Android/
   │   ├─ libs/arm64-v8a/    ← 第 2 包解壓於此（librtabmap_core.a、libopencv_*.a、libomp.so …）
   │   ├─ 3rdparty/libs/arm64-v8a/
   │   └─ libMylibs.a
   └─ Win64/                 （opencv_world4110.dll / .lib）
```

> 更新完 Include / Library 後，請刪除 `Intermediate`、`Binaries` 後重新產生專案檔並重編。

---

## 環境需求

| 項目 | 需求 |
|------|------|
| Unreal Engine | 5.3 以上（`MultiVR.Build.cs` 以 UE 5.3 的 `ProjectDescriptor` API 偵測 MetaXR） |
| 目標平台 | Android（arm64-v8a）為主，Win64 可在編輯器內開發 |
| HMD | Meta Quest 3 / 3S / Quest Pro；HTC 裝置走 LBS 模式（`IsHTCLBS`） |
| 相依插件 | `MetaXR (OculusXR)`（選用，啟用後自動開啟 Passthrough / Eye / Face Tracking）、`AndroidPermission`、`WebSockets`、`EyeTracker` |
| 其他 | NDK 需支援 arm64-v8a；專案需開啟 Exceptions / RTTI（ThirdParty 模組已自行設定） |

---

## 安裝步驟

1. 將 `MultiVR`、`MetaXR`、`MoonshineEngine`複製到專案的 `Plugins/` 底下。
2. 依上方[額外下載檔案](#額外下載檔案必做)補齊 `Include` 與 `Library`。
3. 在 `.uproject` 啟用插件；若要使用 Passthrough / Eye / Face Tracking，需一併啟用 `OculusXR`。
   - 編譯時 log 會顯示 `--- [MultiVR] MetaXR Plugin detected! WITH_META_XR=1 ---`，代表 Meta 功能已編入。
4. 產生 Visual Studio 專案檔 → 編譯 → 開啟編輯器。

---

## 目錄結構

```
Plugins/MultiVR/
├─ MultiVR.uplugin
├─ Content/                       Curve、Materials、EUW（編輯器工具 Widget、Galaxy 素材庫）
├─ Include/ , Library/            第三方標頭檔與靜態庫（需另外下載）
└─ Source/
   ├─ MultiVR/                    Runtime 主模組（Android + Win64）
   │  ├─ Public/                  MultiVRGIS.h / MultiVRComponent.h / MultiVRDatas.h / MultiVRPluginSettings.h
   │  ├─ Private/
   │  ├─ Android/MultiVRManifest_APL.xml   權限、WebRTC aar、Java 注入
   │  └─ Java/com/multivr/        CameraBridge / CameraViewModel(Single/Double) 相機橋接
   ├─ ThirdParty/                 SLAM 實作（slam.cpp / mapper_S.cpp）+ OpenCV / Aruco / RTAB-Map 連結設定
   ├─ PlatformUtility/Meta/        MetaXR 專用元件（Passthrough / EyeTracking / FaceTracking）
   └─ MultiVREditor/              編輯器模組（Http 物件、素材下載匯入、miniz 解壓）
```

同 repo 內的其他插件：

- **MetaXR**：Meta 官方 OculusXR 插件（Passthrough、Anchors、Scene、Movement…）。
- **MoonshineEngine**：全景 / 投影相關的編輯器工具與範例關卡（`EUW_MoonshineEngine`、`SL_*.umap`）。

---

## 模組架構

| 模組 | 型別 | 平台 | 說明 |
|------|------|------|------|
| `MultiVR` | Runtime | Android / Win64 | 核心：`UMultiVRGIS`（GameInstanceSubsystem）、`UMultiVRComponent`（SceneComponent） |
| `ThirdParty` | Runtime | Android / Win64 | SLAM 演算法與 OpenCV / Aruco / RTAB-Map / PCL / Boost 靜態庫連結；Android 額外載入 `libomp.so` |
| `MultiVRMetaModule` | Runtime | 需 OculusXR | Passthrough 透明度、Eye / Face Tracking 封裝；由 `WITH_META_XR` 控制是否編入 |
| `MultiVREditor` | Editor | — | Galaxy 素材庫（登入、分類、下載、FBX 匯入）、編輯器通知、Http 封裝 |

執行期核心關係：

```
UMultiVRGIS (GameInstanceSubsystem)          UMultiVRComponent (掛在 VR Pawn 上)
 ├─ WebSocket / UDP 連線與收發               ├─ 訂閱 OnDetectAruco → 修正 Pawn 位置 / 旋轉
 ├─ 相機影像 → SLAM → OnDetectAruco 廣播      ├─ Lerping 狀態每 0.05s 送出頭盔 transform
 ├─ Aruco 地圖參數（範圍、offset、baseheight） ├─ Recenter 偵測、柱子重定位閃爍
 └─ Meta Passthrough 透明度 / Eye / Face      └─ 自動掛載 Meta 三個元件（WITH_META_XR）
```

---

## 快速開始

### 1. 在 VR Pawn 上加入元件

於 VR Pawn（含 `UCameraComponent` 的 Pawn）加入 **MultiVR Component**。
元件會在第一次定位成功時自動抓取 Camera 並把 Pawn 設為 Aruco 中心 Actor。

### 2. 啟動系統

Blueprint 取得 `MultiVRGIS`（Get Game Instance Subsystem → MultiVRGIS）後呼叫：

```
StartSystem(
    IP        = "192.168.9.223",   // 留空則向 https://multivrapi.moonsihemultivr.workers.dev/ip 自動查詢
    Port      = "5000",
    ClientId  = "",                // 留空自動用本機 IP 當 ID
    MapName   = "MapCodeName",     // Aruco 地圖代號，向後端索取
    RoomID    = "1234",
    IsHTCLBS  = false,             // true = 使用 HTC LBS，不跑 Aruco SLAM
    SlamType  = ARUCO,             // ARUCO / MARKLESS / MIX
    HMDType   = "quest3s"
)
```

流程會自動完成：連線 WebSocket → 取得 `init_player_info` → 索取相機標定 `get_calibration` → 索取地圖 `get_map` → 初始化 SLAM → 開啟相機 → 定位成功後 `OnSystemReady`。

### 3. 綁定事件

至少建議綁定：`OnSystemReady`、`OnDetectNearbyUsers`、`WSMessage_StartGame`、`WSMessage_CloseGame`、`OnEmergencyAlert`、`OnEmergencyClear`、`OnCameraBlock`。



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
| `GetRoomID()` / `GetClientID()` / `GetIsHTCLBS()` | 狀態查詢 |



斷線時會自動重試（`RetryConnectWebsocket`），重連成功走 `WSOnReConnected`。

#### 資料傳送

| 函式 | 送出 type | 用途 |
|------|-----------|------|
| `WSSend_HeadSetData(Position, Rotation)` | `transform_update` | 頭盔位置/朝向（**經 UDP 7000 送出**，單位由 cm 轉為 m） |
| `WSSend_HeadSetDataToIndex(SocketIndex, HeadsetID, Pos, Rot)` | `transform_update` | 指定第 N 條 WebSocket 送出（多頭盔模擬） |
| `WSSend_Progress(Percent)` | `game_progress` | 遊戲載入進度（GameServer） |
| `WSSend_GameStatus(EGameStatus)` | `game_status` | `GAME_IDLE / GAME_PLAYING / GAME_END / GAME_PAUSE` |
| `SendEmergencyAlertAck(AlertId)` | `emergency_alert_ack` | 收到警報時 C++ 已自動回覆，必要時可手動再送 |

#### SLAM / 相機

| 函式 | 說明 |
|------|------|
| `ActiveCamera()` | 透過 JNI 呼叫 Java `SetupCamera` / `CameraBridge.openCameraFromNative` 開相機 |
| `InitSlam_Aruco(ArucoSize, MaxDetectLength)` | 初始化 Aruco SLAM（預設 0.40 m / 5 m，由 `OnReadyInitSlam` 呼叫） |
| `SetOutlierFilterParam(RejectDistCm, ConfirmCount, HistoryLen)` | 定位離群過濾參數（預設 100 cm / 3 次 / 5 筆） |
| `GetCameraTexture()` / `GetCameraData(W,H)` / `UpdateCameraTexture()` | 取得相機畫面 Texture2D（640×480 BGRA），可做 AI 影像或除錯顯示 |
| `SetBaseOffsetInMeters()` / `SetBaseRotationInMeters()` | 調整 XR 基準位移與旋轉 |

#### WebRTC 串流

`ConnectWebRTC()`（送 `stream_request`）、`CloseWebRTC()` / `EndWebRTC()`（送 `stream_end`）、`CallOffer()`、`WebRTC_SendOffer/Answer/IceCandidate(...)`。

#### JSON 工具（皆為 BlueprintCallable）

`GetStringValue`、`GetNumberValue`、`GetBoolValue`、`GetJsonStringValue`、`GetJsonStringArrayValue`、`GetStringArrayValue`、`GetNumberArrayValue`、`GetFloatArrayValue`。
可直接在 Blueprint 解析 `WSOnMessage` 收到的原始字串。

### UMultiVRComponent（SceneComponent）

| 成員 | 說明 |
|------|------|
| `SetDetectInfo(DetectCD, InterpSpeed)` | 定位冷卻秒數與位置插值速度（Android 預設 InterpSpeed 25，並依距離自動調整 2 / 3.75 / 10 / 15） |
| `SetPillarFlashParam(MinDistCm, CooldownSec)` | 柱子重定位閃爍條件：修正距離門檻（預設 100 cm）與冷卻（預設 3 秒） |
| `OnRelocalizeFlash`（BlueprintImplementableEvent） | 場景「直接拉過去」時觸發，接閃爍過場 Blueprint |
| `TickInterval = 0.05` | 每 0.05 秒送一次頭盔 transform |

內部狀態機 `ESlamState`：`Init`（首次定位，直接 Snap 並初始化 Meta 元件）→ `Waiting`（跨一幀）→ `Lerping`（持續平滑修正）。

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
| `WSMessage_CloseGame` | — | 後端要求關閉遊戲 |

### Lobby（頭盔端）

| 事件 | 參數 |
|------|------|
| `WSMessage_StartGame` | `roomid, execPath, ip, port` |

### Content（頭盔端）

| 事件 | 參數 | 說明 |
|------|------|------|
| `OnSystemReady` | — | 首次定位完成、UDP 啟動，可開始遊戲 |
| `OnDetectNearbyUsers` | `TArray<FUser>` | 鄰近玩家（`Clientid` / `Position` / `Rotation` / `IsSameRoom`），已換算到 Aruco 中心座標系 |
| `OnCameraBlock` | — | 相機被遮蔽（SLAM 回傳 `block`） |
| `OnHMDLanguageSet` | `Language` | 後端下發語系 |
| `OnTemperatureGet` | `Temperature` | 場館氣溫 |
| `OnEmergencyAlert` | `FEmergencyAlertData` | 緊急警報（`AlertId` / `Level` / `Cause` / `Instruction` / `Timestamp`），C++ 已自動回 ack |
| `OnEmergencyClear` | `FEmergencyClearData` | 解除警報（`AlertId` / `ResumedRooms` / `Timestamp`），UI 應清除所有警報 |
| `OnUpdateEyeTracking` | `GazeData, StereoGazeData, IsSuccess` | Meta 眼動 |
| `OnUpdateFaceTracking` | `FaceState, IsSuccess` | Meta 表情（70 組 blendshape 權重） |

`EEmergencyLevel`：`NOTICE / WARNING / CRITICAL`
`EEmergencyCause`：`CUSTOM / FIRE / EARTHQUAKE / EVACUATION`

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
| `headset_tracking_update` | 系統就緒時回報 tracking 狀態 |
| `stream_request` / `stream_end` | WebRTC 串流開始 / 結束 |
| `signal_offer` / `signal_ice_candidate` / `headset_msg` | WebRTC 信令 |

### 接收（Server → Client）

| type | 處理 |
|------|------|
| `proximity_update` | 解析鄰近玩家 → `OnDetectNearbyUsers`（WebSocket 與 UDP 兩條路徑皆支援） |
| `headset_game_backend_command` | `command=connect` → `WSMessage_StartGame`；`command=disconnect` → 回 `game_close` + `WSMessage_CloseGame` |
| `start_game` / `pause_game` | → `OnWebAction_StartGame` / `OnWebAction_PauseGame` |
| `close_content_server` | 直接 `RequestExit` 並廣播 `WSMessage_CloseGame` |
| `init_player_info` | 取語系、氣溫、身高（自動 −10 cm 補正相機到頭頂距離）；HTC LBS 直接就緒，否則進入標定流程 |
| `get_calibration_ack` | 存檔 `metamap-cam.yml` 至 `ProjectPersistentDownloadDir` |
| `get_map_ack` | 存檔 `metamap.yml`、解析地圖參數並初始化 SLAM |
| `language_change` | → `OnHMDLanguageSet` |
| `emergency_alert` / `emergency_clear` | → 對應事件（alert 自動 ack） |
| `stream_accepted` / `signal_answer` / `signal_ice_candidate` / `stream_ended` / `stream_end_ack` | WebRTC 信令流程 |



## SLAM 定位流程與調校

```
Java CameraBridge 取影像
      ↓
slam.cpp processImage()（OpenCV + Aruco，Kalman 濾波）
      ↓ JSON { pos, orient, pillar }
取樣平均（MaxSlamSample = 2 幀）
      ↓
離群過濾 AcceptSlamPos()
      ↓
OnDetectAruco.Broadcast(pos, orient, bPillar)
      ↓
UMultiVRComponent 修正 Pawn（Snap 或 Lerp）
```

### 品質控管機制

1. **首次定位角度限制**：第一次 Aruco 定位的 yaw 必須落在 0/90/180/270 附近（誤差 < 10°），否則丟棄重測。
2. **取樣一致性**：同批取樣中若兩幀距離 ≥ 60 cm，整批作廢。
3. **離群過濾**（`AcceptSlamPos`）：新定位與最近 N 筆歷史的 **2D 中位數** 比較；
   - 距離 ≤ `OutlierRejectDist`（預設 100 cm）→ 接受；
   - 超過門檻 → 視為離群候選，必須連續 `OutlierConfirmCount`（預設 3）次且彼此一致才承認為「真移動」，否則丟棄不廣播。
   - 以 `SetOutlierFilterParam()` 調整。
4. **柱子優先重定位**：SLAM 判定該幀由柱子 marker 驅動時（JSON 帶 `"pillar": true`），若修正距離 > `PillarFlashMinDist` 且已過 `PillarFlashCooldown`，場景直接 Snap 並觸發 `OnRelocalizeFlash`；小幅修正仍走平滑 Lerp，避免一直閃。
5. **Recenter 偵測**：比對 HMD pose 的位置/旋轉跳變（一幀內大跳且前一幀靜止），標記為使用者按下 Recenter。
6. **定位冷卻**：定位成功後進入 `DetectCD`（預設 1.5 秒）冷卻，期間不再接受新定位。

---

## Meta XR 功能

啟用 `OculusXR` 插件後（`WITH_META_XR=1`），首次定位完成時 `UMultiVRComponent` 會自動掛載三個元件：

| 元件 | 功能 |
|------|------|
| `UMultiVRMetaPassthroughComponent` | 依玩家相對場地邊界的距離動態調整 Passthrough 透明度：在場地內 = 0（關閉透視），越界或距邊界 10 cm 內漸變到 1（開啟透視）。計算於 `HandleMetaPassthroughOpacity` |
| `UMultiVRMetaEyeTrackingComponent` | 每幀取 Gaze / StereoGaze → `OnUpdateEyeTracking` |
| `UMultiVRMetaFaceTrackingComponent` | 每幀取 70 組表情權重 → `OnUpdateFaceTracking` |

`UMultiVRMetaXRMovementFuncLib::IsSupportEyeTracking() / IsSupportFaceTracking()` 可先行檢查裝置支援度。

未啟用 OculusXR 時 `WITH_META_XR=0`，上述程式碼不編入，插件其餘功能不受影響。

---

## Android 打包注意事項

`MultiVRManifest_APL.xml` 會自動注入：

- 權限：`CAMERA`、`horizonos.permission.HEADSET_CAMERA`、`RECORD_AUDIO`、`MANAGE_EXTERNAL_STORAGE`
- 功能宣告：`android.hardware.camera2.full`
- 相依：`google-webrtc-1.0.32006.aar`（複製到 build/libs 並加入 gradle `implementation`）
- Java 端相機橋接（`com.multivr.CameraBridge` 等）

`ThirdParty_APL.xml` 於啟動時載入 `libomp.so`（OpenMP，未載入會 `UnsatisfiedLinkError` 閃退）。

其他：

- 只支援 **arm64-v8a**。
- 執行期會先請求 Android 權限，權限未過前 `StartConnectWebSocket` 會先擱置，取得後自動續連。
- 標定檔與地圖檔存放於 `FPaths::ProjectPersistentDownloadDir()`（`metamap-cam.yml` / `metamap.yml`）。

---

## 疑難排解

| 症狀 | 可能原因 / 處理 |
|------|-----------------|
| 編譯找不到 `opencv2/…`、`rtabmap/…`、`aruco/…` | Include 包未解壓或路徑錯誤，見[額外下載檔案](#額外下載檔案必做) |
| Link 失敗：`librtabmap_core.a` / `libopencv_*.a` 找不到 | Library 包未解壓到 `Library/Android/libs/arm64-v8a` |
| 裝置一啟動就閃退，log 有 `UnsatisfiedLinkError: libomp.so` | `libomp.so` 未打包，確認 Library 包完整並重新打包 |
| Passthrough / 眼動 / 表情沒作用 | 編譯 log 出現 `WITH_META_XR=0`，請在 `.uproject` 啟用 `OculusXR` 後重編 |
| `calibration not exist` / `aruco map not exist` | 後端沒有該 `deviceType` 的標定檔或該 `MapName` 的地圖，向後端確認代號 |
| 一直 `Cant Slam!!!!!!!` | 頭部移動過快（`IsCanSlam` 需位移 < 0.5 cm、旋轉 < 0.8°），請站定後再定位 |
| 收不到 `proximity_update` | 確認 UDP 7000 未被防火牆阻擋，且 `ArucoCenterActor` 已設定（首次定位完成後才會設定） |
| HTC 裝置定位無效 | HTC 走 LBS 模式，`StartSystem` 的 `IsHTCLBS` 必須為 `true`，此模式不執行 Aruco SLAM |
