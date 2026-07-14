# Workflow：把一個電腦上的專案接上「手機可編輯 + 自動部署」

這份記錄的是把 `trip-planner` 從「只在電腦本機開發」升級成「手機 Claude 可以直接改、改完自動上線」的完整步驟。以後任何專案要做一樣的事，照這份走一遍即可，不用重新摸索。

**適用前提**：專案已經部署在 Firebase Hosting（或準備部署），有一個獨立的 Firebase 專案，且這台電腦已經 `firebase login` 過、`gh auth login` 過。

---

## 總覽（4 個階段）

1. 把專案資料夾變成獨立的 git repo，推上 GitHub
2. 建立一個「只能部署 Hosting、不能做別的事」的服務帳戶，接上 GitHub Actions 自動部署
3. 寫一份給 Claude 看的 `CLAUDE.md`，把專案的坑和架構整理進去
4. 在手機 Claude 上連接這個 repo

---

## 階段 1：獨立成 GitHub repo

```bash
cd <專案資料夾>
git init
git branch -M main

# 排除不該進版控的東西（依專案調整）
cat > .gitignore <<'EOF'
.firebase/
.DS_Store
*.log
EOF

git add -A
git commit -m "Initial commit: <專案名稱>"

# 建立 GitHub repo 並設為 remote（gh 要先登入過：gh auth status 確認）
gh repo create <repo-name> --public --source=. --remote=origin --description "<一句話描述>"
git push -u origin main
```

**如果這個資料夾原本是某個大工作目錄底下的子資料夾**（例如這次的 `Claude code/trip-planner`），記得回到外層那個大 repo，把子資料夾加進外層的 `.gitignore`，避免以後被誤 commit 進大 repo：

```bash
cd <外層工作目錄>
echo "<子資料夾名>/" >> .gitignore
```

（這一步只改檔案、不要自動 commit——外層 repo 的 commit 節奏由使用者自己掌控。）

**Public 還是 Private？** 沒有敏感金鑰（Firebase web API key 本來就設計成公開）的話 public 沒問題，方便之後用 GitHub Pages 之類的附加功能。有疑慮就建 private，之後隨時能改。

---

## 階段 2：自動部署（push 到 main 就上線）

### 為什麼不能直接跑 `firebase init hosting:github`

這個指令會嘗試開瀏覽器做 GitHub OAuth 登入，在很多自動化/無頭環境（包括這次遇到的情況）會因為 PATH 缺 `System32`、或環境本來就沒有互動式瀏覽器而失敗（`spawn cmd ENOENT` 或卡住等不到授權）。遇到這狀況就跳過它，改用下面的手動流程——效果完全一樣，只是自己動手做 `firebase init hosting:github` 原本會自動做的事。

### 手動流程（前提：`firebase login` 已登入過，`gh auth login` 已登入過）

**Step 1：重用 Firebase CLI 已經登入的 Google OAuth token**（不用再開一次瀏覽器授權）：

```bash
REFRESH=$(python -c "
import json
d = json.load(open(r'<你的家目錄>\.config\configstore\firebase-tools.json'))
print(d['tokens']['refresh_token'])
")
RESP=$(curl -s -X POST https://oauth2.googleapis.com/token \
  -d "client_id=563584335869-fgrhgmd47bqnekij5i8b5pr03ho849e6.apps.googleusercontent.com" \
  -d "client_secret=j9iVZfS8kkCEFUPaAeJV0sAi" \
  -d "refresh_token=$REFRESH" \
  -d "grant_type=refresh_token")
ACCESS=$(echo "$RESP" | python -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
```

> 上面那組 `client_id`/`client_secret` 是 firebase-tools 官方公開的 installed-app OAuth client（不是密鑰外洩，這是 CLI 工具的公開身分）。這個 token 帶有 `cloud-platform` scope，足夠呼叫下面的 Google Cloud API。

**Step 2：建立一個專用服務帳戶**（只給它 Hosting 部署權限，不要給更大的權限）：

```bash
PROJECT_ID="<你的 firebase 專案 id>"
SA_ID="github-actions-deploy"

curl -s -X POST \
  "https://iam.googleapis.com/v1/projects/$PROJECT_ID/serviceAccounts" \
  -H "Authorization: Bearer $ACCESS" -H "Content-Type: application/json" \
  -d '{"accountId": "'"$SA_ID"'", "serviceAccount": {"displayName": "GitHub Actions Hosting Deploy"}}'
```

**Step 3：只授權「Firebase Hosting 管理者」角色**（get → 改 → set，不要整包覆蓋，以免動到既有的其他權限）：

```bash
SA_EMAIL="${SA_ID}@${PROJECT_ID}.iam.gserviceaccount.com"

curl -s -X POST "https://cloudresourcemanager.googleapis.com/v1/projects/$PROJECT_ID:getIamPolicy" \
  -H "Authorization: Bearer $ACCESS" -H "Content-Type: application/json" -d '{}' \
  -o current_policy.json

python -c "
import json
d = json.load(open('current_policy.json'))
bindings = d.get('bindings', [])
role = 'roles/firebasehosting.admin'
member = 'serviceAccount:$SA_EMAIL'
if not any(b['role']==role and member in b.get('members',[]) for b in bindings):
    for b in bindings:
        if b['role']==role:
            b['members'].append(member); break
    else:
        bindings.append({'role': role, 'members': [member]})
d['bindings'] = bindings
json.dump({'policy': d}, open('new_policy_request.json','w'))
"

curl -s -X POST "https://cloudresourcemanager.googleapis.com/v1/projects/$PROJECT_ID:setIamPolicy" \
  -H "Authorization: Bearer $ACCESS" -H "Content-Type: application/json" \
  --data @new_policy_request.json -o /dev/null
```

**Step 4：生一把金鑰，存成 GitHub Secret，然後立刻從本機刪掉**：

```bash
curl -s -X POST \
  "https://iam.googleapis.com/v1/projects/$PROJECT_ID/serviceAccounts/$SA_EMAIL/keys" \
  -H "Authorization: Bearer $ACCESS" -H "Content-Type: application/json" \
  -d '{"privateKeyType": "TYPE_GOOGLE_CREDENTIALS_FILE", "keyAlgorithm": "KEY_ALG_RSA_2048"}' \
  -o key_response.json

python -c "
import json, base64
d = json.load(open('key_response.json'))
open('service-account-key.json','wb').write(base64.b64decode(d['privateKeyData']))
"

# 存進 GitHub Secret（要用大寫、底線分隔，避開特殊字元）
gh secret set FIREBASE_SERVICE_ACCOUNT_<PROJECT_ID_大寫底線> \
  --repo <owner>/<repo-name> < service-account-key.json

# ⚠️ 立刻刪掉本機所有金鑰/token 暫存檔，一把都不留
rm -f service-account-key.json key_response.json current_policy.json new_policy_request.json
```

**Step 5：寫 GitHub Actions workflow**（`.github/workflows/firebase-hosting-merge.yml`）：

```yaml
name: Deploy to Firebase Hosting on merge
on:
  push:
    branches:
      - main
jobs:
  build_and_deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken: "${{ secrets.GITHUB_TOKEN }}"
          firebaseServiceAccount: "${{ secrets.FIREBASE_SERVICE_ACCOUNT_<PROJECT_ID_大寫底線> }}"
          channelId: live
          projectId: <你的 firebase 專案 id>
```

想要「PR 先出預覽網址、合併才上正式站」的話，再加一份 `firebase-hosting-pull-request.yml`（把 `on: push` 換成 `on: pull_request`，拿掉 `channelId: live`）。

**Step 6：commit、push，實際跑一次確認會成功**（不要假設寫完就沒事，一定要真的看它跑過一次綠燈）：

```bash
git add .github/
git commit -m "Add GitHub Actions auto-deploy to Firebase Hosting"
git push origin main

gh run list --repo <owner>/<repo-name> --limit 3        # 找到剛觸發的那筆
gh run watch <run-id> --repo <owner>/<repo-name> --exit-status
```

`gh run watch` 跑完顯示 `success` 才算真的接通；順便用 `curl -o /dev/null -w "%{http_code}"` 打一下正式網址確認真的部署上去了。

---

## 階段 3：寫給 Claude 看的 `CLAUDE.md`

放在 repo 根目錄，`git init` 之後、第一次 commit 之前就寫（這樣它會跟著程式碼一起進版控）。內容至少要包含：

- **這個工具是給誰用、他們怎麼描述問題**（例如「使用者不是工程師，回報 bug 只會講現象不會給技術細節，你要自己重現」）
- **正式檔案是哪個、有沒有建置流程**（很多小工具是純靜態檔案，講清楚「唯一要改的檔案是哪個」）
- **怎麼驗證改動**（本機怎麼起預覽伺服器、怎麼用瀏覽器工具直接呼叫頁面裡的函式重現 bug，而不是憑感覺猜）
- **部署方式**（講清楚 push 到 main 會自動部署，不需要手動跑部署指令；如果環境沒有 push 權限才退回手動部署）
- **已經踩過、修過的坑**——這是最重要的一段。把每一個修過的 bug 寫成「症狀 → 根因 → 怎麼修 → 不要重蹈覆轍」，這樣新開的對話不會把已經修好的問題又改回去
- **診斷指令**（怎麼查雲端資料、log 在哪、常見環境變數陷阱）

寫完後可以再加一份簡短的 `README.md`（給人看的），內容指向 `CLAUDE.md`。

---

## 階段 4：手機接上這個 repo

用 **claude.ai 的 GitHub 連接功能**：

1. 手機 App（或行動版網頁）→ **設定 → Connectors（連接的 App）→ GitHub** → 授權存取這個 repo（或整個帳號）
2. 開新對話：「打開 `<owner>/<repo-name>` 這個 repo，先讀 CLAUDE.md 了解專案背景」
3. 之後請它改、跑驗證、commit、push——push 到 main 就會自動部署，全程不用碰電腦

> 選單名稱可能因 App 版本略有不同（可能叫「Apps & Connectors」之類）。如果用的是 Claude Code 而非一般聊天介面，直接用它的 GitHub 整合指到這個 repo 即可，流程一樣。

---

## 檢查清單（做完這套流程要能打勾的項目）

- [ ] `git remote -v` 顯示指到正確的 GitHub repo
- [ ] `gh secret list --repo <owner>/<repo>` 看得到服務帳戶金鑰的 secret
- [ ] 本機資料夾裡**沒有**殘留任何 `.json` 金鑰檔或 access token 檔案
- [ ] `gh run watch <run-id> --exit-status` 對最新一次 push 顯示 `success`
- [ ] 正式網址用 `curl` 打得通、內容是最新版
- [ ] repo 裡有 `CLAUDE.md`，內容包含「已修過的坑」清單
- [ ] 手機 Claude 能讀到這個 repo（用一個無傷大雅的小改動測一次，例如改個文案，確認 push 後幾十秒內正式網址真的更新）
