---
description: 把目前的網頁作品變成可安裝的 App（PWA）：圖示、manifest、sw、安裝按鈕、內建瀏覽器引導，驗證後部署
---

把目前 repo 的網頁作品「App 化」。逐步執行，每一步都要實際驗證，不要只看程式碼就假設沒問題。

## 1. 偵測現況

- 找出進入點（單檔 index.html 或 build 產物）、App 名稱、主色（theme color）
- 盤點已有的 PWA 資產：manifest、sw.js、各尺寸圖示、安裝按鈕
- **已有部署管道（如 Firebase Hosting 自動部署）就沿用，絕對不要另架第二個部署**（例如再開 GitHub Pages）——會造成兩個網址、分享連結分裂

## 2. 產齊 PWA 資產（缺什麼補什麼）

- **圖示**：以 icon.svg（沒有就先設計一個）為原始檔，用 Playwright 開 Chromium 截圖產生 PNG：
  - `icon-192.png`、`icon-512.png`（圓角、透明背景，purpose "any"）
  - `icon-512-maskable.png`（**滿版不透明**，把 SVG 的 rx 圓角改 0，purpose "maskable"，Android 自適應圖示用）
  - `apple-touch-icon.png`（180×180、**滿版不透明**）——**iOS 不支援 SVG 圖示，這張一定要 PNG**，否則 iPhone 加到主畫面是空白圖
  - 產圖腳本範本在 repo 歷史（genicons 流程）：newPage 指定 viewport=尺寸 → setContent 塞 SVG → screenshot
- **manifest.webmanifest**：name/short_name/lang/display=standalone/theme_color/background_color/start_url=./、icons 列出上面全部
- **head**：`<link rel="apple-touch-icon" href="apple-touch-icon.png">`、theme-color、manifest 連結
- **sw.js**：離線快取（靜態資產 cache-first＋背景更新、導覽 network-first）。**動態服務（資料庫、地理編碼含 photon、路線、圖磚）必須排除在攔截之外**。每次改 sw 或資產，`CACHE` 版本號 +1
- **安裝按鈕**（頂欄「📲 安裝」，standalone 模式下隱藏）：
  - Android/桌面 Chrome：攔 `beforeinstallprompt`（preventDefault 存起來）→ 按鈕點擊時 `prompt()`
  - iOS（無此事件）：顯示「Safari 分享 → 加入主畫面」教學彈窗
  - **LINE/FB/IG/WeChat 內建瀏覽器（UA 含 Line\/|FBAN|FBAV|Instagram|MicroMessenger）裝不了 PWA**：顯示「用預設瀏覽器開啟」引導

## 3. 檢查 repo 是 public（免費 Hosting/Pages 需要），private 就提醒使用者

## 4. 驗證（用真實瀏覽器，不是用看的）

- 沙盒連不到外部 CDN 時：`npm pack` 抓 leaflet/firebase 等到本機、sed 改測試副本的 CDN 路徑、`python3 -m http.server` 起站
- Playwright 至少測：預設不顯示安裝按鈕 → 派發假 beforeinstallprompt 後出現 → 點擊有呼叫 prompt；iOS UA 顯示教學；LINE UA 顯示外開引導；四張 PNG 都抓得到；manifest 合法；原有功能回歸測試無 FAIL

## 5. commit → push → 確認 Actions 部署綠燈 → 回報

回報內容：App 網址、三種平台各自的安裝方式、這次補了什麼。commit 訊息說清楚做了什麼與驗證結果。
