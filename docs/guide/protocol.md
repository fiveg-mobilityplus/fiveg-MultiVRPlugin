# 通訊協定

[← 返回文件首頁](../../README.md#目錄)

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

[← 返回文件首頁](../../README.md#目錄)
