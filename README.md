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

## 想在其他專案複製這套「手機可編輯 + 自動部署」流程？

看 [`電腦手機共用編輯workflow.md`](./電腦手機共用編輯workflow.md) — 這份是把這個 repo 從「只在電腦開發」升級成「push 到 main 自動上線、手機 Claude 直接接手改」的完整步驟記錄，包含 GitHub Actions 服務帳戶設定的每一步指令，照著做即可套用到任何 Firebase Hosting 專案。
