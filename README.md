# 行程規劃工具

單一 HTML 檔的旅遊行程規劃工具：多人即時共編、地圖與最短路徑、航班通勤估算、分帳。

**正式網址：https://planngo.web.app**（push 到 `main` 會自動部署到這裡）

給接手開發的 Claude（尤其是手機端全新對話）：**先讀 [`CLAUDE.md`](./CLAUDE.md)**，裡面有完整的架構、已知雷區、驗證流程與部署方式。

## 檔案結構

- `index.html` — 唯一的原始碼檔案（HTML + CSS + JS 全部內嵌，無建置流程）
- `firebase.json` / `.firebaserc` — Firebase Hosting 多站台部署設定（app → planngo、legacy → 舊網址轉址頁）
- `legacy/` — 舊網址 trip-7ab77.web.app 的轉址頁（含 localStorage 資料搬家與退役 sw）
- `database.rules.json` — Realtime Database 安全規則
- `manifest.webmanifest` / `sw.js` / `icon.svg` + 各 PNG 圖示 — PWA 設定
- `.github/workflows/` — 自動部署設定
- `.claude/commands/` — 可重複使用的指令：`/ship-app`（App 化）、`/rename-site`（換網址）

## 工作流程文件

- 📖 [`工作流程手冊.md`](./工作流程手冊.md) — **總目錄**：電腦手機互聯系統怎麼運作、四大流程（日常改版／新專案上架／App 化／換網址）、全系統雷區、指令小抄
- 🔧 [`電腦手機共用編輯workflow.md`](./電腦手機共用編輯workflow.md) — 新專案上架的完整技術步驟（GitHub repo 化、服務帳戶、自動部署）
