# 行程規劃工具

單一 HTML 檔的旅遊行程規劃工具：多人即時共編、地圖與最短路徑、航班通勤估算、分帳。

**正式網址：https://trip-7ab77.web.app**（push 到 `main` 會自動部署到這裡）

給接手開發的 Claude（尤其是手機端全新對話）：**先讀 [`CLAUDE.md`](./CLAUDE.md)**，裡面有完整的架構、已知雷區、驗證流程與部署方式。

## 檔案結構

- `index.html` — 唯一的原始碼檔案（HTML + CSS + JS 全部內嵌，無建置流程）
- `firebase.json` / `.firebaserc` — Firebase Hosting 部署設定
- `database.rules.json` — Realtime Database 安全規則
- `manifest.webmanifest` / `sw.js` / `icon.svg` — PWA 設定
- `.github/workflows/` — 自動部署設定
