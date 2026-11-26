# MultiVR Plugin

**FreeRoam Studio** 是 Moonshine FreeRoam 多人自由走動式 VR 平台中，負責內容與開發整合的模組。**MultiVR Plugin 是 FreeRoam Studio 在 Unreal Engine 5 上的核心插件**，讓開發者能將 UE 遊戲接入場館後台，把虛擬場景對齊實體場地，取得同場玩家的位置，並整合頭盔的透視與追蹤功能。[Moonshine FreeRoam 平台介紹](https://freeroam.moonshine.tw/zh-tw/)

![FreeRoam Studio 場域示意，多位參與者戴著 VR 頭盔在同一個實體空間活動](docs/images/multivr/freeroam-studio.webp)

## 主要能力

| 能力 | 用途 |
|---|---|
| 場地定位 | 利用頭盔相機辨識 ArUco 標記，或透過無標記定位（Markerless SLAM）估算位置與朝向，讓虛擬場景對齊實體場地。 |
| 場館連線 | 透過 WebSocket 與 UDP 交換房間資訊、玩家位置、遊戲狀態及緊急警報。 |
| 鄰近玩家同步 | 取得附近玩家的位置與朝向，供開發者顯示虛擬角色或接近提示。 |
| Meta XR 整合 | 玩家靠近場地牆面或柱子時自動調整透視透明度，並可在支援的裝置上取得眼動與臉部追蹤資料。 |
| 效能控制 | 開發者可透過 Blueprint 調整定位頻率、網路收送頻率與日誌等級。 |
| WebRTC 串流信令 | 在頭盔與營運端之間交換建立畫面串流所需的 offer、answer 與 ICE candidate。 |
| Galaxy 素材庫 | 開發者可從 UE 編輯器存取 FreeRoam Studio 的 Galaxy 素材庫，瀏覽、下載模型並匯入遊戲專案。 |

開發者負責交付頭盔上的 Android 遊戲 APK（Content）與場地電腦上的 Win64 遊戲伺服器（Content Server）。操作頁面、場館後台（Local Server）與頭盔待機程式（Lobby）由系統方提供。

## 目錄

| 想找的內容 | 章節 |
|---|---|
| 系統怎麼運作 | [整合路線](docs/guide/system.md#第一次整合從這裡開始) · [系統架構](docs/guide/system.md#先認識這個系統) · [一場遊戲的時序](docs/guide/system.md#一場遊戲怎麼開始與結束) · [執行時程式與功能](docs/guide/system.md#執行時程式與功能) |
| 安裝交付包 | [環境需求](docs/guide/installation.md#環境需求) · [安裝步驟](docs/guide/installation.md#安裝步驟) · [插件目錄結構](docs/guide/installation.md#目錄結構) |
| 在 UE 專案接線 | [範例專案對照](docs/guide/headset.md#範例專案對照) · [單台頭盔快速開始](docs/guide/headset.md#快速開始) · [Lobby 與多人遊戲](docs/guide/multiplayer.md#multiplayer) · [Content Server 的 GameMode](docs/guide/multiplayer.md#content-server-的-gamemode) |
| 打包與場測 | [Android APK 與 Win64 Server 打包](docs/guide/delivery.md#android-打包注意事項) · [場地驗收](docs/guide/delivery.md#場地驗收) · [Log](docs/guide/troubleshooting.md#log-說明) · [疑難排解](docs/guide/troubleshooting.md#疑難排解) |
| 查功能與參數 | [API](docs/guide/api.md#api-參考) · [事件](docs/guide/events.md#events) · [通訊協定](docs/guide/protocol.md#通訊協定) · [SLAM 定位](docs/guide/slam.md#slam-定位流程) · [效能設定](docs/guide/xr-performance.md#效能設定) · [Meta XR](docs/guide/xr-performance.md#meta-xr-功能) · [Galaxy 素材庫](docs/guide/galaxy.md#galaxy-素材庫) |
