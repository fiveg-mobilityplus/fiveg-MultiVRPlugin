# Log 與疑難排解

[← 返回文件首頁](../../README.md#目錄)

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
| 開專案時說 MultiVR 模組遺失、要求重新編譯 | 引擎 BuildId 不是交付包指定的值。檢查安裝步驟 1，再做一次步驟 6（兩行都要跑） |
| 打包時 `Missing precompiled manifest` | 安裝步驟 4 沒做，或 `Plugins\MultiVR\Intermediate` 被刪除 |
| Server 連結時 `LNK2011` / `LNK1120` | 使用了舊版交付包，請改用最新版 |
| 引擎版本與交付包不符 | 每個交付包只能用於對應的引擎版本（5.3.2 / 5.8.4），請索取對應版本 |
| 裝置一啟動就閃退，log 有 `UnsatisfiedLinkError: libomp.so` | `libomp.so` 未打包，確認 `Plugins\MultiVR\Library` 完整並重新打包 |
| `calibration not exist` / `aruco map not exist` | 後台沒有該 `deviceType` 的標定檔或該 `MapName` 的地圖，向後台確認代號 |
| 開場定位很久才完成 | 看 `[first-init]` log：「無 marker」或「地板」多 → 請正對牆面 marker；「未站定」多 → 請站穩；「雙眼不一致」多 → 靠近 marker 一點再試 |
| 走動時位置校正變慢 | `Slam Interval` 設太大，調小或設回 0 |
| 編輯器裡按 Play 一直等不到 `OnSystemReady` | 正常：相機只在頭盔上開，定位要打包上頭盔測（見[頭盔驗證](headset.md#在頭盔上啟動與驗證)） |
| 建置時找不到 `OculusXR` / `MultiVRMetaModule` 相關符號 | 專案沒放入或沒啟用 Meta XR 插件（安裝步驟 5） |
| 頭盔 `open` content server 後連不上（`contentServer = failed`） | 執行 server 的電腦防火牆要開放 UDP 7777；確認 `-serverip` 是電腦的區網 IP |
| 收不到 `proximity_update` | 確認 UDP 7000 未被防火牆阻擋，且首次定位已完成 |
| HTC 裝置定位無效 | HTC 走 LBS 模式，`StartSystem` 的 `IsHTCLBS` 必須為 `true`，此模式不執行 Aruco SLAM |

[← 返回文件首頁](../../README.md#目錄)
