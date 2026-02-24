
# GCP Dataform 自動化執行 CI/CD 建置手冊

## 概述

本文件旨在說明如何透過 GitHub Actions，在程式碼合併或推播至特定分支時，自動觸發 GCP Dataform 進行編譯（Compilation）與執行（Invocation），將最新的 SQLX 邏輯部署並運行於 BigQuery 中。

---

## 事前準備

在開始設定之前，請確認你擁有以下資訊：

1. **GCP Project ID**: 專案的真實 ID（例如：`XXX-project`）
2. **Dataform Location**: 儲存庫所在的 GCP 區域（例如：`asia-east1`）
3. **Dataform Repository Name**: 在 GCP Dataform 介面上建立的儲存庫名稱（請注意，**不是** GitHub 的網址，例如：`WorkspaceTest`）
4. **GCP 權限**: 擁有設定 IAM 服務帳號與金鑰的權限。

---

## 第一階段：GCP IAM 與服務帳號設定

為了讓 GitHub Actions 能順利呼叫 GCP API 並執行 BigQuery，我們需要配置一個具備正確權限的服務帳號（Service Account, SA），並且必須打通 Dataform 的隱藏後台權限。

### 1. 建立 / 確認服務帳號

建議建立一個專門給 GitHub Actions 使用的 SA，例如：`bigquery-github-action@<project-id>.iam.gserviceaccount.com`。

### 2. 賦予必要的 IAM 角色

請至 **GCP 控制台 -> IAM 與管理 -> IAM**，確保該 SA 擁有以下角色：

* **BigQuery 資料編輯者 (BigQuery Data Editor)**：允許建立與修改 BigQuery 資料表。
* **BigQuery 工作使用者 (BigQuery Job User)**：允許消耗運算資源執行 SQL。
* **服務帳戶使用者 (Service Account User)**：允許服務帳號在呼叫 API 時「扮演自己」去執行任務的必要權限。

### 3. 產出 JSON 金鑰並存入 GitHub Secrets

1. 進入 **IAM 與管理 -> 服務帳戶**，點擊該帳號，進入「金鑰」分頁。
2. 點擊「新增金鑰」->「建立新的金鑰」-> 選擇 **JSON** 格式並下載。
3. 進入你的 GitHub 儲存庫 -> **Settings -> Secrets and variables -> Actions**。
4. 新增一個 Secret：
* **Name**: `GCP_SA_KEY`
* **Secret**: 貼上剛剛下載的完整 JSON 內容。



### 4. 設定 Dataform 隱藏後台帳號權限 (終極魔王關卡)

Dataform 系統背後有一個由 Google 自動管理的隱藏服務帳號，它必須擁有接手你的 SA 的權限，否則在最後一步去 BigQuery 倒資料時會被拒絕。

1. 進入 GCP 控制台的 **IAM 與管理 -> IAM** 頁面。
2. 在畫面右上方，**務必勾選「包含 Google 提供的角色授權」 (Include Google-provided role grants)**。
3. 在列表中搜尋：`service-[你的專案編號]@gcp-sa-dataform.iam.gserviceaccount.com`。
4. 點擊編輯，並為它新增 **「服務帳戶權杖建立者」 (Service Account Token Creator)** 角色。
5. 儲存並等待 3~5 分鐘生效。

---

## 第二階段：GitHub Actions YAML 腳本建置

請在你的 GitHub 專案中建立路徑與檔案：`.github/workflows/dataform-run.yml`，並填入以下內容。

```yaml
name: 自動執行 Dataform SQLX

on:
  push:
    branches:
      - test  # 觸發的分支，可依環境更改為 main 或 dev

env:
  PROJECT_ID: jchuang-project
  LOCATION: asia-east1
  REPOSITORY: ding
  # 執行任務的 GCP 服務帳號 Email
  SERVICE_ACCOUNT_EMAIL: bigquery-github-action@jchuang-project.iam.gserviceaccount.com

jobs:
  run-dataform:
    runs-on: ubuntu-latest
    steps:
      - name: 取得程式碼
        uses: actions/checkout@v4

      - name: GCP 權限驗證 (Authenticate to Google Cloud)
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: 設定 Google Cloud SDK
        uses: google-github-actions/setup-gcloud@v2

      - name: 透過 REST API 執行 Dataform
        run: |
          echo "1. 取得 Google Cloud Access Token..."
          ACCESS_TOKEN=$(gcloud auth print-access-token)
          
          echo "2. 呼叫 API：建立 Compilation Result (鎖定本次 Push 的 Commit)..."
          # 使用 ${{ github.sha }} 確保 Dataform 精準編譯最新推播的程式碼版本
          COMPILATION_RESPONSE=$(curl -s -X POST "https://dataform.googleapis.com/v1/projects/${{ env.PROJECT_ID }}/locations/${{ env.LOCATION }}/repositories/${{ env.REPOSITORY }}/compilationResults" \
            -H "Authorization: Bearer $ACCESS_TOKEN" \
            -H "Content-Type: application/json" \
            -d "{\"gitCommitish\": \"${{ github.sha }}\"}")
            
          COMPILATION_RESULT=$(echo $COMPILATION_RESPONSE | jq -r '.name // empty')
          
          if [ -z "$COMPILATION_RESULT" ]; then
            echo "❌ 編譯失敗！API 回傳內容："
            echo "$COMPILATION_RESPONSE"
            exit 1
          fi
          echo "✅ 編譯成功！Compilation Result ID: $COMPILATION_RESULT"
          
          echo "3. 呼叫 API：建立 Workflow Invocation (送進 BigQuery 執行)..."
          # 組裝 Payload，必須明確指定使用的服務帳號
          PAYLOAD=$(cat <<EOF
          {
            "compilationResult": "$COMPILATION_RESULT",
            "invocationConfig": {
              "serviceAccount": "${{ env.SERVICE_ACCOUNT_EMAIL }}"
            }
          }
          EOF
          )

          INVOCATION_RESPONSE=$(curl -s -X POST "https://dataform.googleapis.com/v1/projects/${{ env.PROJECT_ID }}/locations/${{ env.LOCATION }}/repositories/${{ env.REPOSITORY }}/workflowInvocations" \
            -H "Authorization: Bearer $ACCESS_TOKEN" \
            -H "Content-Type: application/json" \
            -d "$PAYLOAD")
          
          INVOCATION_ID=$(echo $INVOCATION_RESPONSE | jq -r '.name // empty')
          
          if [ -z "$INVOCATION_ID" ]; then
            echo "❌ 執行觸發失敗！API 回傳內容："
            echo "$INVOCATION_RESPONSE"
            exit 1
          fi
          echo "Dataform 執行已成功觸發！Invocation ID: $INVOCATION_ID"


```

### YAML 設定詳解與最佳實踐

* **不使用 `gcloud dataform` 指令的原因**：GitHub Actions 預設的環境可能缺少 Dataform 核心組件。直接使用 `curl` 呼叫 REST API 最為輕量且穩定。
* **為何使用 `${{ github.sha }}` 而不是分支名稱 (`test`)**：若只給分支名稱，GCP 可能會因為快取機制而抓到舊版本程式碼。傳遞本次觸發的 Git Commit Hash (`github.sha`)，能 100% 保證編譯的是最新版本。
* **`invocationConfig.serviceAccount` 的必要性**：當 GCP 專案開啟嚴格權限檢查時，API 不接受匿名執行，必須在 Payload 中明確告知要使用哪個 SA 身分去跑 BigQuery。

---

## 常見錯誤代碼與踩坑排查 (Troubleshooting)

如果你在 Action Log 中看到以下錯誤，請對照解決：

### ❌ 錯誤 1：`404 NOT_FOUND` (parent resource not found)

* **原因**：API 找不到你指定的 Dataform 儲存庫。
* **解法**：檢查 YAML 檔中的 `PROJECT_ID`、`LOCATION`、`REPOSITORY` 變數是否打錯。特別注意 `REPOSITORY` 必須是 GCP Dataform 介面上的名稱，而非 GitHub 儲存庫名稱。

### ❌ 錯誤 2：`404 NOT_FOUND` (Can't find package.json)

* **原因**：Dataform 編譯所需的核心檔案（`package.json` 與 `workflow_settings.yaml`）不在 GitHub 專案的根目錄，或者根本沒有推上儲存庫。
* **解法**：確保這些設定檔位於 Git 專案最外層。若你將 Dataform 專案包在某個子資料夾內，需至 GCP Dataform 介面修改「Git 設定」中的「存放區中的路徑 (Directory path)」。

### ❌ 錯誤 3：`400 INVALID_ARGUMENT` (Service account must be set when strict act as checks are enabled)

* **原因**：沒有在執行 API 的 Payload 中指定 Service Account，或者你填入了個人的 Email（例如 `@gmail.com` 或公司信箱）。
* **解法**：確保 YAML 檔中帶入了 `"serviceAccount": "${{ env.SERVICE_ACCOUNT_EMAIL }}"`，且 Email 必須是 `.gserviceaccount.com` 結尾的標準 GCP 服務帳號。

### ❌ 錯誤 4：`403 PERMISSION_DENIED` (The caller does not have permission to act as service account...)

* **原因**：這是一個「身分代持 (Impersonation)」的權限問題。呼叫 API 的服務帳號（GitHub 登入用的）沒有權限扮演「它自己」。
* **解法**：至 GCP IAM 介面，找到該服務帳號，點擊編輯，並新增 **「服務帳戶使用者 (Service Account User)」** 角色。設定後務必 **等待 5~10 分鐘** 讓 GCP 快取更新。

### ❌ 錯誤 5：隱藏魔王權限錯誤 (`cannot actAs... please grant Service Account Token Creator role`)

* **原因**：Dataform 系統背後的隱藏 Google 服務帳號，沒有權限去「接手並扮演」你指定的執行帳號。
* **解法**：參考本文「第一階段」的第 4 步驟，勾選「包含 Google 提供的角色授權」，找出隱藏的 Dataform 帳號並賦予 **「服務帳戶權杖建立者」** 權限。

---

## 常見疑問 Q&A

### ❓ 疑問一：GitHub Action 顯示成功 (綠燈)，但 BigQuery 裡面卻沒有建立 Table？

* **解答**：API 確實成功呼叫了 Dataform，但 Dataform 掃描程式碼後，編譯了 0 個檔案。因為過程沒有報錯，所以回傳成功。
* **排查與解法**：
1. 確保所有的 `.sqlx` 檔案都放在 **`definitions/`** 資料夾內（Dataform 有硬性規定，放在資料夾外會被直接無視）。
2. 確認副檔名嚴格為 `.sqlx`。
3. 確認 `workflow_settings.yaml` 中有設定 `defaultDataset`，或是在 SQLX 檔的 `config` 區塊有明確指定 `schema`。



### ❓ 疑問二：執行成功了，但 Dataform 網頁介面上的 Code 沒有同步更新？

* **解答**：Dataform 網頁介面是工程師的「個人工作區 (Workspace)」，不會隨著 CI/CD 自動覆蓋。CI/CD 腳本是直接從 GitHub 抓取最新程式碼編譯並丟給 BigQuery 執行。
* **如何驗證**：請直接至 BigQuery 檢查資料表是否已更新，或進入 GCP Dataform -> 選擇儲存庫 -> 點擊上方 **「工作流程叫用 (Workflow Invocations)」** 標籤頁查看實際的執行紀錄與 SQL。
