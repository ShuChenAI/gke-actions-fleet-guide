# ShuChenAI：GKE Actions Fleet 使用指南

這份指南說明如何讓獲授權的 ShuChenAI repository 使用 GKE Actions Fleet 執行 GitHub Actions。使用方式是在 job 的 `runs-on` 指定 runner group 與規格 label。Fleet 在同一個 GKE cluster 提供具名 AMD64／ARM64 runners，有工作才擴容，閒置時 worker pools 縮到 0。

原始碼與部署工具：[luisleo526/gke-actions-fleet](https://github.com/luisleo526/gke-actions-fleet)。一般使用者不需要 GCP ADC、GCP service-account key 或 runner 管理 App 的私鑰。

## 1. 先取得使用權

請 repository 維護者向 Fleet 管理者申請：

- 將 repository 加入 organization runner group **`gke-runners`** 的允許清單。
- 將要使用的 workflow **完整路徑與受信任 branch ref** 加入 Selected workflows，例如：

  ```text
  ShuChenAI/YOUR-REPO/.github/workflows/fleet-manual.yml@refs/heads/main
  ```

- 若是公開 repository，需要管理者另外確認 runner group 的 public-repository 使用權與 PR 保護設定。

`YOUR-REPO`、workflow 檔名及 branch 都必須與實際設定一致。使用 `dev`／`master` 的 repository，應填入對應 ref。知道 runner 名稱並不會自動取得使用權；公開這份文件也不會開放一個任何人都能使用的公共 runner 服務。

## 2. 選擇規格與架構

| CPU | RAM | AMD64 label | ARM64 label | AMD64 / ARM64 配置上限 |
|---:|---:|---|---|---:|
| 2 | 4 GiB | `gke-2cpu-4gb-amd64` | `gke-2cpu-4gb-arm64` | 16 / 16 |
| 4 | 8 GiB | `gke-4cpu-8gb-amd64` | `gke-4cpu-8gb-arm64` | 16 / 16 |
| 8 | 16 GiB | `gke-8cpu-16gb-amd64` | `gke-8cpu-16gb-arm64` | 16 / 8 |
| 16 | 32 GiB | `gke-16cpu-32gb-amd64` | `gke-16cpu-32gb-arm64` | 16 / 8 |
| 32 | 64 GiB | `gke-32cpu-64gb-amd64` | `gke-32cpu-64gb-arm64` | 16 / 8 |

AMD64 合計 80、ARM64 合計 56，上限獨立計算；同一規格／架構的容量由獲授權 repositories／organizations 共用，不是每個 repo 各有一份。這些是配置上限，實際可用量仍受雲端 quota、區域機型容量及其他排隊工作影響。

每個 job 有獨立的 **480 GiB 工作磁碟**，workspace 與 Docker data 共用。Job 結束、runner pod 刪除後，工作磁碟會自動清除；VM 閒置縮容與磁碟刪除是非同步的。跨 job 資料請明確使用 artifacts／適當的 cache，不能依賴本機工作區保留。

工作磁碟預設 480 GiB，既有 runner label 不變。管理者可透過全域 `workspace_gib` 或單一 profile 的同名參數調整容量；新 runner 提供 `RUNNER_WORKSPACE_GIB`。相較 80 GiB，PD Balanced 每個磁碟小時約增加 US$0.055（依地區價格與磁碟存續時間計費）。詳細設定與線上擴容見[主專案文件](https://github.com/luisleo526/gke-actions-fleet#工作磁碟容量與費用)。名稱尾綴不會自動調整容量。

## 3. 手動試跑

將 [examples/manual.yml](examples/manual.yml) 複製到目標 repository 的 `.github/workflows/fleet-manual.yml`。`workflow_dispatch` 的 workflow 檔案必須先存在 repository 的預設分支；再從 Actions 頁面執行 **Run workflow**，選擇已獲管理者授權、且包含受信任 workflow 的 branch。

完整範例：

```yaml
name: Fleet manual check
on:
  workflow_dispatch:
permissions:
  contents: read
jobs:
  check:
    runs-on:
      group: gke-runners
      labels: gke-8cpu-16gb-amd64
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@d23441a48e516b6c34aea4fa41551a30e30af803 # v6
        with:
          persist-credentials: false
      - name: Verify runner
        shell: bash
        run: |
          set -euo pipefail
          uname -m
          git rev-parse HEAD
          printf 'Profile: %s; CPU quota: %s; RAM bytes: %s\n' \
            "$RUNNER_PROFILE" "$RUNNER_CPU_LIMIT" "$RUNNER_MEMORY_LIMIT"
```

在已獲授權的非 PR 工作中，可以保留原本的 build/test steps，改用上述 `runs-on`。使用這份規格 label 時，請一併移除舊 fleet 的動態 job labels；其他 labels 必須先與管理者確認。PR 工作請使用第 6 節的完整身分檢查流程，勿直接加上未受保護的 PR trigger。

## 4. 同時測試 AMD64 與 ARM64

使用 [examples/dual-arch.yml](examples/dual-arch.yml)，或在 job 設定：

```yaml
strategy:
  fail-fast: false
  matrix:
    architecture: [amd64, arm64]
runs-on:
  group: gke-runners
  labels: ${{ format('gke-8cpu-16gb-{0}', matrix.architecture) }}
```

ARM64 是原生 ARM 執行，不是自動模擬 AMD64。使用的 actions、Docker images、下載的 binaries 及 native dependencies 都需要支援所選架構。macOS／Windows jobs 請繼續使用適合的 OS runner。

## 5. Repository secrets 與 GitHub Environments

Fleet 使用標準 GitHub self-hosted runner。Repository／獲授權 organization secrets、variables、`GITHUB_TOKEN` permissions、OIDC、artifacts/cache、reusable workflows、Docker job/service containers 都沿用 GitHub Actions 的正常行為。

完整範例：[examples/secrets-environment.yml](examples/secrets-environment.yml)。在要使用的 GKE job 本身指定：

```yaml
runs-on:
  group: gke-runners
  labels: gke-4cpu-8gb-amd64
environment: staging
permissions:
  contents: read
  id-token: write
env:
  API_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
  API_URL: ${{ vars.API_URL }}
```

- `environment: staging` 選擇 GitHub Environment，其 secrets、variables、required reviewers 與 deployment branch rules 對該 job 生效。
- `env:` 才是傳給 job 程序的環境變數。Secret 必須透過 `${{ secrets.NAME }}` 明確傳給 `env` 或 action 的 `with`，不會自動變成 `.env` 檔或全部程序的環境變數。
- Repository secrets 可在同一個 job 取用；organization secrets 需先授權該 repo。另一個 job 的 environment 不會自動繼承。
- Reusable workflow 若需要 caller secrets，可由受信任 caller 宣告 `secrets: inherit`；environment secrets 由實際宣告 environment 的 job 取得。
- OIDC 需要 `id-token: write` 及雲端的 federation 設定；這不會自動授予 operator ADC 或 Kubernetes 權限。
- 標準 GitHub 對 fork／Dependabot secrets 的限制仍適用。PR 保留下一節的成員檢查，不要把會員驗證 App／runner 管理 App 的私鑰傳入測試 job。

若管理者已配置一般／保護兩條路徑，持有部署、發佈或 baseline promotion 憑證的工作應依核准政策選用保護 group 與獨立 `gkp-*` label；不要為了解除排隊而改送一般 group。

預設 `allow_sudo: true`，可在 runner container 內安裝套件；管理者可為特定 profile 設為 false。Runner 可寫層沒有固定 2 GiB cap，但仍受節點 OS 磁碟限制；大型 workspace、暫存檔及 Docker data 使用獨立 480 GiB job disk。完整能力與界線見[主專案相容性文件](https://github.com/luisleo526/gke-actions-fleet/blob/main/docs/compatibility.md)。

## 6. 只允許 organization member 的 PR 使用 Fleet

完整範例：[examples/member-only-pr.yml](examples/member-only-pr.yml)。請將它複製到目標 repository 的 `.github/workflows/gke-member-pr.yml`，再由管理者完成以下設定：

1. 使用獨立的 GitHub App，僅授予 **Organization Members: read**，安裝至 ShuChenAI。不要使用 runner 管理 App 的私鑰。
2. 設定 Actions variable **`GKE_MEMBERSHIP_APP_ID`**。
3. 建立 **`gke-pr-admission`** environment，deployment branch policy 使用 Selected branches，只允許受保護的 base branch。
4. 將 **`GKE_MEMBERSHIP_APP_PRIVATE_KEY`** 只存為該 environment 的 secret。不要貼入 YAML、commit、issue，也不要另外複製為 repository／organization secret。
5. 在 `gke-runners` 的 Selected workflows 加入正確的受信任 ref，例如：

   ```text
   ShuChenAI/YOUR-REPO/.github/workflows/gke-member-pr.yml@refs/heads/main
   ```

6. 保護該 base branch 的 workflow 變更。若改用 `dev`／`master`，同步修改 PR trigger、environment branch policy 與 runner-group workflow ref。

這份範例的行為是：

- `pull_request_target` 使用受信任 base branch 的 workflow。
- 身分檢查在 GitHub-hosted runner 上執行，**不 checkout、不執行 PR 程式碼**。
- 即時確認 PR 作者、事件觸發者與重新執行者都是 active organization member。非成員、pending invitation 或 API／憑證失敗，都不會啟動 GKE 工作。
- 通過後，GKE job 才 checkout 核准的 PR head SHA；只給 `contents: read`，不繼承身分 App／runner App 的私鑰。

外部 PR 仍可能產生輕量的 GitHub-hosted admission run／紀錄，但不會取得 GKE runner。只加 `author_association` 或一條 `if:` 並不足以防止 PR 自行新增另一條直接使用 runner 的 workflow；runner group 的 repository/workflow allowlist 也必須設定。

此範例需完成 App、environment 與 allowlist 設定後才可使用；缺少設定時會拒絕執行。範例中的 GKE steps 只驗證執行環境，請加入目標專案的實際 build/test 指令。完整原理與驗收項目見[主專案的 PR 存取控制文件](https://github.com/luisleo526/gke-actions-fleet/blob/main/docs/member-only-pr.md)。

## 7. 遷移時要注意

- Fleet image 不等同 `ubuntu-latest` 的完整工具集合。請明確設定 Python／Node 等版本；預設可在 ephemeral runner container 內執行 `sudo -n apt-get`；常用套件也可與管理者協作預裝至 runner image。這不包含 node／VM 的 systemd、kernel 或叢集管理權限。
- Runner 與 DinD 的 CPU／RAM limits 依規格設定，兩者共用底層 VM；重型 host 與 Docker 工作同時執行時，要一起評估記憶體需求。
- 原生編譯平行度建議使用 `RUNNER_CPU_LIMIT`；`nproc` 可能回報較大底層 VM 的核心數。
- 冷啟動包含 VM、image pull 與磁碟建立，第一批工作可能較慢。請明確設定 `timeout-minutes`，不要把冷啟動時間視為固定 SLA。
- Docker 使用每個 runner 獨立的 DinD；服務容器與特殊網路假設需要實測。需要 GCP 等雲端權限的工作，另行設定最小權限 OIDC，不使用操作人員 ADC。
- Fleet 僅供同一信任邊界內的獲授權工作。Privileged DinD 不是敵意多租戶沙箱；不要將未審查的 fork code 直接送進 worker。

## 8. 工作一直排隊時

依序提供以下資訊給 Fleet 管理者，不要附上任何 token／私鑰：

1. Workflow run URL、repo 名稱與 job 名稱。
2. 完整的 `runs-on` group／label，以及實際 workflow path／branch ref。
3. 確認該 repo 與 workflow 已獲授權。
4. 若是 PR，確認 membership job 的結果與 environment 是否配置。
5. 由管理者確認 quota、機型容量、pool 上限與排隊狀態。

一般使用者不需要自行建立或刪除 GKE nodes。部署與維運操作見[主專案 README](https://github.com/luisleo526/gke-actions-fleet#readme)及[營運手冊](https://github.com/luisleo526/gke-actions-fleet/blob/main/docs/operations.md)。

範例引自 GKE Actions Fleet，依 [MIT License](LICENSE) 提供。
