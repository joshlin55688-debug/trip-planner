# 行程規劃工具（trip-planner）— 給接手的 Claude 看

這是單一 HTML 檔的旅遊行程規劃工具，正式檔是 **`index.html`**（唯一要改的檔案；`trip_planner.html` 只是轉址到 index.html 的相容 stub，不要改它）。無建置流程、無 npm 套件、無框架 —— 純 HTML + CSS + 內嵌 `<script>`，直接用文字編輯器改。

**正式網址**：https://trip-7ab77.web.app
**Firebase 專案**：trip-7ab77（Realtime Database + Hosting）
**Repo**：這個 repo 獨立於使用者其他專案，push 到 `main` 會自動部署上線（見下方「部署」）。

---

## 你在幫誰做這件事

使用者（Josh）用這個工具規劃自己的旅行，有時跟朋友一起共編行程（例如「香港」那趟）。他不是工程師，說話很直接、簡短（例如「拖曳排序後所有的東西都會消失」「搜不到是你的問題」），**回報 bug 時只會描述現象，不會給你技術細節或 log**。你需要自己在瀏覽器裡重現問題、找根因、修好、驗證、部署，然後用白話跟他報告「為什麼會這樣」和「我改了什麼」。

不要要求他貼 console log 或去開發者工具查——他不會做這件事，直接用工具（瀏覽器 preview / 讀程式碼）自己查。

## 開發與驗證方式（很重要，照這個流程做）

1. **改 `index.html`**（唯一原始碼檔案）。
2. **用瀏覽器 preview 驗證，不要只憑閱讀程式碼就假設沒問題**：
   - 啟動一個靜態伺服器指向這個資料夾（例如 `npx serve trip-planner -p 4325`，或任何你環境裡可用的靜態伺服器/預覽工具）。
   - 用瀏覽器工具的 `eval`／`javascript_tool` 直接呼叫頁面裡的 JS 函式來重現/驗證 bug（這個專案大量函式都掛在 `window` 上，例如 `optimizeDay`、`applyRemote`、`geocode`、`computeBalances`），比截圖快很多且不會被 UI 動畫卡住。
   - 每次修完 bug，**用真實資料流程重現一次原本的問題，確認修好**，不要只看程式碼邏輯覺得「應該對了」。
   - 驗證完記得清掉測試用的假行程資料（`state.trips = []; save();` 之類），不要把測試垃圾留給使用者。
3. **部署**：這個 repo 接了 GitHub Actions，**push 到 `main` 分支就會自動 `firebase deploy` 上線**，不需要手動跑 `firebase deploy`。改完、測完、commit、push 即可，數分鐘內 https://trip-7ab77.web.app 就會更新。
   - 如果你的環境沒有 push 權限或沒有 GitHub Actions 可跑（例如某些沙盒環境），才需要手動用 `firebase deploy --only hosting`（前提是有登入 `firebase login`，這台裝置本機平常已登入，但雲端沙盒環境不會有這個登入狀態，所以優先用 push 觸發部署）。
   - 部署設定檔：`firebase.json`（`public: "."`，把整個資料夾當靜態網站根目錄）、`.firebaserc`（綁定專案 trip-7ab77）。

## 這個工具長什麼樣子（功能總覽）

單頁應用，`view.page` 控制目前畫面：`list`（行程列表）／`trip`（行程詳細，時間軸＋候選地點側欄＋可選地圖）／`overview`（漂亮的可列印行程表，含每日小地圖）／`split`（分帳）。

- **多行程管理**：`state.trips[]`，存在 `localStorage` key `tripPlanner.v1`。
- **每日時間軸**：活動有分類（景點/美食/交通/住宿/購物/其他）、時間、結束時間（算時長、偵測重疊）、費用、地點、訂位連結、備註。排序用每筆活動的 **▲▼ 按鈕**（`moveAct`）——**不要重新做成拖曳排序，手機上 HTML5 拖曳基本不能用，這是已經驗證過的教訓**。
- **預算＋幣別**：可設定非台幣幣別＋匯率，自動換算「約合台幣」。
- **航班**：填去/回程航班時間、通勤時間、報到安檢預留，自動算「建議幾點出門」，並自動把航班變成當天時間軸上的一筆活動（`syncFlightActs`，掛 `flightKey` 標記、不可被拖曳/誤編輯覆蓋）。
- **路線估時**（`estimateRoute`）與**最短路徑**（`optimizeDay` + `solveTSP`，最近鄰+2-opt，可選起點與是否來回）：靠 OSRM 免費服務算車程。
- **地圖**（Leaflet + OpenStreetMap 圖磚）：`initDayMap` 畫當天路線圖，`initOverviewMaps` 在行程表每天畫一張小圖。圖釘可拖曳或用「📍 點地圖設定位置」（`armPlace`，手機友善，比拖曳可靠）校正位置；查不到座標的地點會顯示成橘色閃爍圖釘，放在地圖中央等待你設定。
- **候選地點側欄**（`trip.pool`，**只存本機、不同步共編**）：可貼地名/Google 地圖連結/Plus Code，或匯入 Google Takeout（GeoJSON）/我的地圖（KML）/CSV，拖曳或按鈕加進某天變成正式活動。
- **地理定位**（`geocode`）：多層 fallback，見下方「地理定位」章節，**這是這個專案最常出問題、也是修最多次的部分**。
- **分帳**（`view.page='split'`）：`trip.people` + `trip.expenses`，`computeBalances` 算每人淨額，`settleUp` 用貪婪演算法算出最少筆數的還款方式。
- **多人即時共編**：見下方獨立章節，**這是整個專案最脆弱、最容易在改動時引入 regression 的部分，改航班/活動/共編邏輯之前務必先讀完這節**。
- **PWA**：`manifest.webmanifest` + `sw.js`，可安裝到手機主畫面、離線可看已存的行程。改 `sw.js` 記得升版本號（`CACHE` 常數），不然使用者裝置會卡在舊快取。
- **匯出**：📋複製純文字行程、📤匯出/📥匯入 JSON 備份。

## 多人即時共編（最容易出問題的部分，改之前先讀）

用 **Firebase Realtime Database**，免登入帳號，安全模型是「房號是隨機亂碼、猜不到 + 資料庫規則擋掉列舉」（見 `database.rules.json`：只開放 `rooms/$roomId` 讀寫、root 禁止列舉、房號需 6–64 字元）。

- 使用者第一次按「🤝 共編」要貼自己的 Firebase 專案設定（存在 `localStorage.tripPlanner.fb`）。已經設定過了（這台環境部署的就是 trip-7ab77 專案），一般不需要再重新設定。
- 邀請連結格式 `?room=xxx&cfg=base64(firebaseConfig)`，用 **URL-safe base64**（`b64encode`/`b64decode`，把 `+/=` 轉成 `-_`，因為標準 base64 的這些字元在網址裡會被破壞）。
- 資料結構：`rooms/{roomId}/{info, acts/{actId}, expenses/{expId}}`。`info` 存行程基本資料（名稱/日期/預算/幣別/**region 主要地區**/航班/同行人員）。`acts` 是扁平物件、每筆有穩定 `id` 和 `di`（第幾天）欄位。
- **讀取用 `onValue` 整房重建**（`applyRemote`），**寫入用單點 `set`/`remove`**（`rAct`/`rDelAct`/`rInfo`/`rExpense`/`rDelExpense`）。
- **`applyRemote` 重建活動時一定要帶 `di`**——這是修過的嚴重 bug：漏了 `di` 會讓同步後的活動 `di` 變 `undefined`，之後任何重新上傳（拖曳排序、最短路徑）會被 `actToRemote` 的 `a.di | 0` 預設成 `0`，導致活動集體「跳到第一天消失」。
- **`subscribeRoom` 的 Firebase 回呼要合併延遲套用**（`pendingSnap`/`applyTimer`，`setTimeout(0)` 合併），不能收到快照就立刻 `applyRemote`——因為我們自己連續寫多筆資料時（例如先寫 acts 再寫 flights），中途的同步快照是「半套」狀態，若立即套用會把使用者剛編輯的內容打回舊值再寫回雲端，造成「怎麼改都沒用」的 bug（已修過，別重蹈覆轍）。
- **刪除/離開共編行程要呼叫 `unsubscribeRoom()`**，否則監聽器還活著，行程刪掉後同伴一有動靜又會把它「復活」。
- 航班活動有 `flightKey` 標記，不能被一般活動編輯流程覆蓋掉這個標記（否則 `syncFlightActs` 之後認不出它，會產生孤兒重複活動）；`syncFlightActs` 有自我修復機制，會清掉標題為「✈️ 搭機」但沒有 `flightKey` 的孤兒。
- 共編列有**連線狀態指示**：`initFirebase` 掛了 `.info/connected` 監聽（只掛一次，`initFirebase._connWatch` 防重複），斷線時 `fbConnected=false`、共編列變橘色提醒「改動先存本機」，重連自動變回綠色。Firebase 重連後會自動補送離線期間的寫入，這裡只負責提示。

## 地理定位（`geocode` 函式，多層 fallback）

免費、免金鑰，但準確度有限，這是使用者最常抱怨的部分。順序：

1. 使用者直接貼**座標**（`22.30, 114.17`）→ 直接用，最準。
2. 使用者貼 **Google Plus Code**（`853C+F2 佐敦 香港`）→ 用 `olcDecode`/`olcRecover` 純算法解碼，完全不靠任何網路服務，最準。地名可以在碼的前面或後面。
3. **Photon**（`photon.komoot.io`，OSM 資料但模糊搜尋/POI 比 Nominatim 好）→ 主要引擎，會帶入行程的 `region`（主要地區，使用者可在編輯行程時填，例如「香港」）與 `geoBias()`（用行程裡已知座標算出的中心點）來提高準確度。
4. **Nominatim**（`nominatim.openstreetmap.org`）→ Photon 失敗或逾時（6 秒 `fetchTimeout`）才退回這個。
5. 都查不到 → 該地點在地圖上顯示成**橘色閃爍圖釘**放在地圖中央，等使用者手動拖曳或點地圖設定。

**這套 fallback 是免費方案能做到的極限**——OpenStreetMap 資料庫沒有 Google Maps 那麼完整，有些小型店家/飯店（例如使用者提過的「登瑞酒店 Stage Well Hotel」）在 OSM 根本查不到，這不是程式 bug，是資料源限制。如果使用者持續抱怨定位不準，正確的解法是接 **Google Geocoding API**（他有自己的 Google Cloud 專案 trip-7ab77，可以開這個 API，有免費額度），不是繼續在免費 OSM 方案裡打轉。

批次定位（一次查很多點，例如開地圖或算最短路徑）用 `geocodeSlow`，會依實際用到的服務自動調整等待時間（快取/座標/Plus Code 不等、Photon 等 0.6 秒、Nominatim 等 1.1 秒），比固定等 1.1 秒快很多——**不要把這個改回固定 `sleep(1100)`**。

`geoCache` 的快取規則（修過的 bug，不要改回去）：**「查不到」（null）只在當次頁面有效，不寫進 localStorage**——以前失敗會被永久存起來，一次網路不穩就讓那個地點永遠定位不到；`saveGeo` 存檔時會過濾 null，載入時也會清掉舊版殘留的 null。同理 **`sw.js` 一定要把 `photon.komoot.io` 排除在快取攔截之外**（photon 查無結果也回 200，被 cache-first 存住就永遠重查不到；nominatim/OSRM 本來就有排除）。

開「🗺️ 看地圖」時只等**當天活動**定位完就開圖；候選地點（pool）是開圖後在背景一顆一顆補灰點（`pendingPool`），**不要改回「等全部候選定位完才開圖」**——候選一多（匯入 Takeout）會讓地圖卡一分鐘開不出來。

## 診斷雲端資料

```bash
export PATH="$PATH:/c/Users/z2398/AppData/Roaming/npm"   # 或你環境裡 firebase CLI 的路徑
export MSYS_NO_PATHCONV=1   # Git Bash 在 Windows 上會把 /rooms 這種路徑誤轉成 Windows 路徑，一定要設這個
firebase database:get /rooms --shallow --instance trip-7ab77-default-rtdb
firebase database:get /rooms/<roomId>/acts --instance trip-7ab77-default-rtdb
```

使用者目前主要在用的共編房間是 `rmqszgdlzk8tto`（他的「香港」行程）。裡面第一天／最後一天各有一筆「登瑞酒店」是**故意的**（早上從飯店出發、晚上回飯店），不是重複資料，不要「清理」掉。

## 常見雷區總結（都是真實修過的 bug，不要重蹈覆轍）

- 拖曳排序在手機上不能用 → 用 ▲▼ 按鈕，不要做回拖曳。
- 地圖自動縮放不能把候選地點（`trip.pool`）也算進去，否則遠方的候選會把地圖拉到看全世界；只框當天活動的座標。
- 任何「重新上傳一批活動到雲端」的操作（拖曳排序、最短路徑、估路線）都要確保每筆活動的 `di` 正確，並且真的呼叫 `rAct` 同步上去（估路線曾經漏掉同步，車程算完一重新整理就消失）。
- 編輯活動時如果地點文字沒變，要保留使用者之前手動校正過的座標（`a.lat`/`a.lon`）；地點文字改了才清掉重新定位。
- 航班「建議出發時間」在通勤+報到預留都是 0 的時候，算出來會等於起飛時間本身，這是誤導的假資訊，要判斷 `lead > 0` 才顯示，否則顯示「請填通勤與報到時間」的提示（文字匯出、卡片顯示、自動產生的航班活動備註，三個地方都要一致處理，之前修過但漏了文字匯出那一處）。
- Realtime Database 的安全規則没有使用者驗證，純靠房號不可猜測 + 不可列舉，改規則時要保持這個設計，不要為了方便加上寬鬆的萬用讀寫權限。

## 開發環境備忘

- Windows + Git Bash。Firebase CLI 裝在 `C:\Users\z2398\AppData\Roaming\npm`，Bash 裡要先 `export PATH="$PATH:/c/Users/z2398/AppData/Roaming/npm"`。
- 這個資料夾原本是使用者一個大的本機工作目錄（`C:\Users\z2398\Documents\Claude code`）底下的子資料夾，現在已經獨立成自己的 git repo 並推上 GitHub、接上自動部署，方便手機端的 Claude 直接編輯這個 repo、推 push 後自動上線，不用再手動跑部署指令。
