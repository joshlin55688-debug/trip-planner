---
description: 把 Firebase Hosting 換成好看的新網址（xxx.web.app），舊網址自動轉址且使用者資料自動搬家。用法：/rename-site 新名字
argument-hint: 新站名（小寫英文/數字/連字號）
---

把這個專案的正式網址換成 `$ARGUMENTS.web.app`（名字轉小寫；未提供就先問使用者要什麼名字）。照下列步驟做，每步驗證後才進下一步。

## 1. 建立新站台（不需要進 Firebase 控制台）

加一個一次性 workflow `.github/workflows/setup-site.yml`（跑完就刪）：

```yaml
name: Setup hosting site (one-off)
on: workflow_dispatch
jobs:
  create:
    runs-on: ubuntu-latest
    steps:
      - name: Write service account key
        env:
          SA_JSON: ${{ secrets.<部署用的服務帳號 secret 名稱> }}
        run: printf '%s' "$SA_JSON" > "$RUNNER_TEMP/sa.json"
      - name: Create site
        env:
          GOOGLE_APPLICATION_CREDENTIALS: ${{ runner.temp }}/sa.json
        run: npx -y firebase-tools@13 hosting:sites:create <新站名> --project <專案id>
```

push → 用 GitHub API/MCP 觸發 workflow_dispatch → 看 job log 確認「Site URL: https://<新站名>.web.app」。名字被占用就回報使用者換一個。

## 2. 多站台部署設定

- `.firebaserc`：加 `targets`，`app` → 新站名、`legacy` → 舊站名
- `firebase.json`：hosting 改陣列，`app` 目標指向原本的 public 目錄（ignore 加上 `legacy/**`）、`legacy` 目標指向 `legacy/` 資料夾
- PR 預覽 workflow 加 `target: app`；merge workflow 不用改（`firebase deploy --only hosting` 會兩個站台一起部署）

## 3. 舊站轉址頁＋資料搬家（最重要，漏了使用者資料會「看起來全消失」）

**瀏覽器 localStorage 是每個網址各自獨立的**，所以 `legacy/index.html` 要：

1. 讀舊網址 localStorage 裡的應用資料（本專案是 tripPlanner.v1 / .fb / .me）
2. 打包成 URL-safe base64，掛在 `#m=` 轉到新網址（同時把 `location.search` 的 `?room=` 等參數原樣帶過去，舊邀請連結才不會壞）
3. 新站 index.html 啟動時偵測 `#m=`：**只補不存在的資料（比對 id/roomId）、不覆蓋**、匯入後清掉 hash——重複點舊連結不能重複匯入

另放 `legacy/sw.js` 自殺型 Service Worker（skipWaiting → 清全部 caches → unregister → 重新導航所有視窗），否則已安裝舊 App 的裝置永遠卡在舊版快取。

## 4. 收尾

- README / CLAUDE.md / 使用說明內的網址全部更新，註明「舊網址自動轉址」
- 刪掉一次性 setup workflow
- Playwright 端對端驗證：route 攔截新網址域名、由本機測試站回應 → 實測「舊站有資料 → 轉址 → 新站自動匯入、hash 清除、重複轉址不重複匯入、?room= 參數保留」＋原功能回歸測試
- commit → push → 確認 Actions 綠燈且 log 顯示**兩個站台**都部署成功

## 5. 回報使用者

新網址、舊連結不會壞、以及他要做的兩件事：**用常用瀏覽器開一次舊網址讓資料搬家**、**刪掉舊的主畫面 App 到新網址重裝**。
