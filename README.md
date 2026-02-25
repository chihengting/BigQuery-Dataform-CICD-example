
# GCP Dataform 自動化執行 CI/CD 建置手冊

## 概述

* 本文件旨在說明如何透過 GitHub Actions，在程式碼合併或推播至特定分支時，
* 自動觸發 GCP Dataform 進行編譯（Compilation）與執行（Invocation），
* 將最新的 SQLX 邏輯部署並運行於 BigQuery 中。

---

## 事前準備

在開始設定之前，請確認具備以下資訊與權限：

1. **GCP Project ID**: 專案的真實 ID（例如：`jchuang-project`）
2. **Dataform Location**: 儲存庫所在的 GCP 區域（例如：`asia-east1`）
3. **Dataform Repository Name**: 在 GCP Dataform 介面上建立的儲存庫名稱（請注意，此為 GCP 上的名稱，非 GitHub 儲存庫名稱，例如：`ding`）
4. **GCP 權限**: 擁有設定 IAM 服務帳號與建立 JSON 金鑰的權限。

---

## 第一階段：GCP IAM 與服務帳號設定

為使 GitHub Actions 具備呼叫 GCP API 並執行 BigQuery 之權限，需配置專用的服務帳號（Service Account, SA），並開通 Dataform 系統後台帳號的連動權限。

### 1. 建立 / 確認服務帳號

建議建立專供 GitHub Actions 使用的 SA，例如：`bigquery-github-action@<project-id>.iam.gserviceaccount.com`。

### 2. 賦予必要的 IAM 角色

請至 **GCP 控制台 -> IAM 與管理 -> IAM**，確保該 SA 擁有以下角色：

* **BigQuery 資料編輯者 (BigQuery Data Editor)**：允許建立與修改 BigQuery 資料表。
* **BigQuery 工作使用者 (BigQuery Job User)**：允許消耗運算資源執行 SQL。
* **服務帳戶使用者 (Service Account User)**：允許服務帳號在呼叫 API 時「扮演自己」去執行任務之必要權限。

### 3. 產出 JSON 金鑰並存入 GitHub Secrets

1. 進入 **IAM 與管理 -> 服務帳戶**，點擊該帳號，進入「金鑰」分頁。
2. 點擊「新增金鑰」->「建立新的金鑰」-> 選擇 **JSON** 格式並下載。
3. 進入 GitHub 儲存庫 -> **Settings -> Secrets and variables -> Actions**。
4. 新增一個 Secret：
* **Name**: `GCP_SA_KEY`
* **Secret**: 貼上完整 JSON 內容。



### 4. 設定 Dataform 系統後台帳號權限

Dataform 系統具備一個由 Google 自動管理的隱藏服務帳號，該帳號必須擁有接手自訂 SA 的權限，否則將無法於 BigQuery 中建立資料表。

1. 進入 GCP 控制台的 **IAM 與管理 -> IAM** 頁面。
2. 於畫面右上方，**勾選「包含 Google 提供的角色授權」 (Include Google-provided role grants)**。
3. 於列表中搜尋：`service-[專案編號]@gcp-sa-dataform.iam.gserviceaccount.com`。
4. 點擊編輯，並新增 **「服務帳戶權杖建立者」 (Service Account Token Creator)** 角色。
5. 儲存設定並等待 3 至 5 分鐘使其生效。

---

## 第二階段：GitHub Actions 腳本建置

為節省 BigQuery 運算成本，僅針對本次 Commit 中有修改的 `.sqlx` 檔案進行執行。

### 前提與注意事項 (嚴格規範)

必須遵守以下兩項規範，否則執行階段將發生錯誤：

1. **檔案名稱與資料表名稱必須完全一致**：腳本依賴實體檔名來推斷 Dataform 目標。例如，檔案命名為 `test_diff.sqlx`，其內容的 `config { name: "test_diff" }` 亦必須為 `test_diff`。
2. **目標資料集 (Dataset) 設定必須正確**：需於環境變數 `TARGET_DATASET` 中明確指定這些資料表所屬的 Schema。該值必須與 Dataform 實際編譯目標一致（通常為 `workflow_settings.yaml` 中的 `defaultDataset` 或 `.sqlx` 內指定的 `schema`）。

### YAML 設定檔內容

請於 GitHub 專案中建立 `.github/workflows/dataform-run.yml`，並填入以下內容：

```yaml
name: 自動執行 Dataform SQLX (僅異動檔案)

on:
  push:
    branches:
      - test  # 觸發的分支，可依環境更改

env:
  PROJECT_ID: jchuang-project
  LOCATION: asia-east1
  REPOSITORY: ding
  SERVICE_ACCOUNT_EMAIL: bigquery-github-action@jchuang-project.iam.gserviceaccount.com
  # 必須與 workflow_settings.yaml 的 defaultDataset 或 SQLX 中的 schema 一致
  TARGET_DATASET: ding_test 

jobs:
  run-dataform:
    runs-on: ubuntu-latest
    steps:
      - name: 取得程式碼
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 必須設為 0，以獲取完整 git 歷史進行差異比對

      - name: 偵測有異動的 .sqlx 檔案
        id: changed-files
        uses: tj-actions/changed-files@v44
        with:
          files: '**/*.sqlx'
          separator: ' '

      - name: GCP 權限驗證 (Authenticate to Google Cloud)
        if: steps.changed-files.outputs.any_changed == 'true'
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: 透過 REST API 執行 Dataform
        if: steps.changed-files.outputs.any_changed == 'true'
        run: |
          echo "1. 取得 Google Cloud Access Token..."
          ACCESS_TOKEN=$(gcloud auth print-access-token)
          
          echo "2. 呼叫 API：建立 Compilation Result..."
          COMPILATION_RESPONSE=$(curl -s -X POST "https://dataform.googleapis.com/v1/projects/${{ env.PROJECT_ID }}/locations/${{ env.LOCATION }}/repositories/${{ env.REPOSITORY }}/compilationResults" \
            -H "Authorization: Bearer $ACCESS_TOKEN" \
            -H "Content-Type: application/json" \
            -d "{\"gitCommitish\": \"${{ github.sha }}\"}")
            
          COMPILATION_RESULT=$(echo $COMPILATION_RESPONSE | jq -r '.name // empty')
          
          if [ -z "$COMPILATION_RESULT" ]; then
            echo "編譯失敗！API 回傳內容："
            echo "$COMPILATION_RESPONSE"
            exit 1
          fi
          echo "編譯成功！Compilation ID: $COMPILATION_RESULT"
          
          echo "3. 動態組合異動檔案的 Target 清單..."
          TARGETS_JSON="[]"
          
          for FILE in ${{ steps.changed-files.outputs.all_changed_files }}; do
            TABLE_NAME=$(basename "$FILE" .sqlx)
            echo "偵測到異動表: $TABLE_NAME"
            
            TARGET_JSON=$(jq -n \
              --arg db "${{ env.PROJECT_ID }}" \
              --arg schema "${{ env.TARGET_DATASET }}" \
              --arg name "$TABLE_NAME" \
              '{database: $db, schema: $schema, name: $name}')
            
            TARGETS_JSON=$(echo "$TARGETS_JSON" | jq ". += [$TARGET_JSON]")
          done
          
          echo "生成的執行目標：$TARGETS_JSON"

          echo "4. 呼叫 API：建立 Workflow Invocation..."
          PAYLOAD=$(jq -n \
            --arg comp "$COMPILATION_RESULT" \
            --arg sa "${{ env.SERVICE_ACCOUNT_EMAIL }}" \
            --argjson targets "$TARGETS_JSON" \
            '{
              compilationResult: $comp,
              invocationConfig: {
                serviceAccount: $sa,
                includedTargets: $targets,
                transitiveDependenciesIncluded: true
              }
            }')

          INVOCATION_RESPONSE=$(curl -s -X POST "https://dataform.googleapis.com/v1/projects/${{ env.PROJECT_ID }}/locations/${{ env.LOCATION }}/repositories/${{ env.REPOSITORY }}/workflowInvocations" \
            -H "Authorization: Bearer $ACCESS_TOKEN" \
            -H "Content-Type: application/json" \
            -d "$PAYLOAD")
          
          INVOCATION_ID=$(echo $INVOCATION_RESPONSE | jq -r '.name // empty')
          
          if [ -z "$INVOCATION_ID" ]; then
            echo "執行觸發失敗！API 回傳內容："
            echo "$INVOCATION_RESPONSE"
            exit 1
          fi
          echo "Dataform 執行已成功觸發！Invocation ID: $INVOCATION_ID"

```

### YAML 設定詳解與最佳實踐

* **不使用 `gcloud dataform` 指令的原因**：GitHub Actions 預設的環境可能缺少 Dataform 核心組件。直接使用 `curl` 呼叫 REST API 最為輕量且穩定。
* **精準編譯 (`${{ github.sha }}`)**：避免 GCP 快取延遲，傳遞本次推播的 Git Commit Hash，以確保編譯最新版本。
* **連鎖更新機制 (`transitiveDependenciesIncluded: true`)**：設定為 `true` 以確保當上游資料表發生異動時，依賴該表的所有下游資料表也會自動重跑，維持資料一致性。

---

## 疑難排解與錯誤代碼 (Troubleshooting)

若於 Action Log 中發現錯誤，請依照以下說明排查：

### 錯誤 1：`404 NOT_FOUND` (parent resource not found)

* **原因**：API 找不到指定的 Dataform 儲存庫。
* **解法**：檢查 YAML 中的 `PROJECT_ID`、`LOCATION`、`REPOSITORY` 變數。特別注意 `REPOSITORY` 必須為 GCP Dataform 介面上的名稱，而非 GitHub 儲存庫名稱。

### 錯誤 2：`404 NOT_FOUND` (Can't find package.json)

* **原因**：Dataform 編譯所需的核心檔案（`package.json` 與 `workflow_settings.yaml`）不在 GitHub 專案的根目錄。
* **解法**：確保設定檔位於 Git 專案最外層。若專案包含於子資料夾內，需至 GCP Dataform 介面修改「Git 設定」中的「存放區中的路徑 (Directory path)」。

### 錯誤 3：`400 INVALID_ARGUMENT` (Service account must be set when strict act as checks are enabled)

* **原因**：未於 API Payload 中指定 Service Account，或使用了非服務帳號格式之 Email。
* **解法**：確保 YAML 檔已帶入 `"serviceAccount"`，且 Email 以 `.gserviceaccount.com` 結尾。

### 錯誤 4：`403 PERMISSION_DENIED` (The caller does not have permission to act as service account)

* **原因**：呼叫 API 的服務帳號缺少扮演自身的權限。
* **解法**：至 GCP IAM 介面，為該服務帳號新增 **「服務帳戶使用者 (Service Account User)」** 角色。設定後需等待 5 至 10 分鐘讓快取更新。

### 錯誤 5：系統後台權限錯誤 (`cannot actAs... please grant Service Account Token Creator role`)

* **原因**：Dataform 系統隱藏帳號缺少接手服務帳號的權限。
* **解法**：參考第一階段第 4 步驟，勾選「包含 Google 提供的角色授權」，為隱藏的 Dataform 帳號新增 **「服務帳戶權杖建立者」** 權限。

### 錯誤 6：目標不存在 (`Requested target does not exist in compilation result`)

* **原因**：腳本動態生成的執行目標（Database + Schema + Name），於 Dataform 實際編譯的結果中不存在。
* **解法**：
1. 檢查異動的 `.sqlx` 檔案名稱是否與設定區塊中的 `config { name: "..." }` 完全一致。
2. 檢查 YAML 腳本中的 `TARGET_DATASET` 是否與 `workflow_settings.yaml` 內的 `defaultDataset` 或檔案內自訂的 `schema` 一致。



---

## 常見疑問 Q&A

### 疑問一：GitHub Action 顯示成功，但 BigQuery 內部未建立資料表？

* **解答**：此為 API 執行成功，但 Dataform 編譯檔案數為 0 導致的無效執行。
* **排查方式**：
1. 確保所有 `.sqlx` 檔案皆放置於 **`definitions/`** 資料夾內。
2. 確認副檔名嚴格為 `.sqlx`。
3. 確認 `workflow_settings.yaml` 中有設定 `defaultDataset`。



### 疑問二：執行成功後，Dataform 網頁介面的程式碼並未同步更新？

* **解答**：Dataform 網頁介面屬工程師的「個人工作區 (Workspace)」，不會透過 CI/CD 自動覆蓋。CI/CD 機制為直接從 GitHub 讀取原始碼進行編譯與執行。

* **驗證方式**：請直接至 BigQuery 檢查資料表內容，或於 GCP Dataform 選擇儲存庫，點擊上方 **「工作流程叫用 (Workflow Invocations)」** 查看實際執行紀錄。
