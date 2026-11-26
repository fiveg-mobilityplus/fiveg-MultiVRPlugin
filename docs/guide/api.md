# API 參考

[← 返回文件首頁](../../README.md#目錄)

### Multi VRGIS

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
| `ApplyPerfSettings(bSlamPaused, SlamInterval, PluginLogLevel, UploadHz, RemoteHz)` | 一次設定全部效能參數，見[效能設定](xr-performance.md#效能設定) |
| `GetPerfSettings(...)` | 讀回目前設定與網路狀態（往返延遲、伺服器停留時間、每秒收送筆數） |

#### WebRTC 串流

`ConnectWebRTC()`（送 `stream_request`）、`CloseWebRTC()` / `EndWebRTC()`（送 `stream_end`）、`CallOffer()`、`WebRTC_SendOffer/Answer/IceCandidate(...)`。

#### JSON 工具（皆為 BlueprintCallable）

`GetStringValue`、`GetNumberValue`、`GetBoolValue`、`GetJsonStringValue`、`GetJsonStringArrayValue`、`GetStringArrayValue`、`GetNumberArrayValue`、`GetFloatArrayValue`。
可直接在 Blueprint 解析 `WSOnMessage` 收到的原始字串。

### MultiVR Component

| 成員 | 說明 |
|------|------|
| `SetDetectInfo(DetectCD, InterpSpeed)` | 定位冷卻秒數（預設 0.5 秒）與位置插值速度（Android 預設 25，並依修正距離自動調整） |
| `SetPillarFlashParam(MinDistCm, CooldownSec)` | 柱子重定位閃爍條件：修正距離門檻（預設 100 cm）與冷卻（預設 3 秒） |
| `OnRelocalizeFlash`（BlueprintImplementableEvent） | 場景「直接拉過去」時觸發，接閃爍過場 Blueprint |
| `SetAutoZonePassthrough(bEnable)` | 開關「靠近危險區自動開透視」（預設開）。關閉時透明度回到 `SetMetaPassthroughOpacity` 設定的值 |
| `SetMetaPassthroughOpacity(Opacity)` | 手動設定 Passthrough 基礎透明度（0 = 關閉透視、1 = 全透視） |

[← 返回文件首頁](../../README.md#目錄)
