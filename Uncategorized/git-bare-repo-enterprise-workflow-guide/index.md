+++
title = "企業級 Git 裸儲存庫實戰：氣隙環境的四層架構開發流程"
description = "深入探討企業氣隙環境下，如何使用 Git 裸儲存庫與 USB 隨身碟建立安全的四層架構。外網開發機透過 USB 與內網中介機同步，最終推送到內網 Git 伺服器。包含完整 PowerShell 自動化腳本與實戰經驗。"
date = "2025-11-07T06:09:59.000Z"
updated = "2025-11-10T12:28:47.456Z"
draft = false

[extra]
withAI = "使用 GitHub Copilot 搭配 Claude Sonnet 4.5 寫作"
card = "preview.png"
featured = true

  [extra.preview]
  url = "https://gemini.google.com/share/f70d088f8fc3"
  withAI = true
  description = "Made with Gemini 2.5 Flash Image (Nano Banana)"

  [extra.comments]
  id = "115524629015951412"

[taxonomies]
tags = [ "DevOps", "Git" ]
licenses = [ "GFDL 1.3" ]
+++

> 本文重寫自我的 AI Blog
>
> [以 USB 和 Git 裸儲存庫：離線程式碼同步的終極指南 - 琳聽智者漫談](https://xn--uy0a.tw/Git/git-bare-repo-usb-offline-code-sync-guide/)

在企業環境中，基於資安考量，內外網嚴格隔離是常見的安全策略。開發者會需要在開發工作站安裝各種開發工具和套件，這些行為讓開發環境相對「危險」。而 Git 儲存庫則託管在受保護的內網伺服器上，兩者之間無法直接連線。在這種嚴格的網路隔離政策下，如何維持有效的版本控制和程式碼同步，成為實際的開發挑戰。

本文將分享一套經過實戰驗證的解決方案，透過 Git 裸儲存庫（Bare Repository）與 USB 隨身碟，在[氣隙環境](https://聆.tw/SystemAdmin/air-gap-network-environment/)建立起符合資安要求的四層架構工作流程。核心概念是利用「物理隔離」來橋接內外網：外網開發機無法連接內網，只能透過 USB 隨身碟作為資料載體，由具有內網權限的中介機器負責最終推送。這個額外的中介層除了滿足資安政策要求，同時提供了一個重要的檢查點，讓變更在推送到內網儲存庫前，能經過最後一道人工驗證。

<!-- more -->

## 為什麼是內外網隔離架構

{% chat(speaker="user") %}
為什麼不能直接讓開發機連到內網的 Git 伺服器？這樣不是簡單很多嗎？
{% end %}

{% chat(speaker="jim") %}
這個問題的答案涉及不同企業的資安考量。開發機器需要安裝各種開發工具、IDE、套件管理器，甚至可能需要從網路下載第三方函式庫。這些操作讓開發環境相對「不乾淨」，存在潛在的安全風險。

如果讓這些開發機直接連接內網，就等於為潛在的威脅打開了一扇門。一旦開發機被入侵，攻擊者可能透過內網連線進一步滲透其他系統。因此，企業會透過嚴格的隔離來保護內網資產。

這種設計確實增加了開發流程的複雜度，但在企業環境中，「安全」的優先級遠高於「方便」。當然，理想狀況是能切一個獨立的 DMZ 開發網段，讓開發機和內網 Git 伺服器能在受控環境下通訊。但在資源或政策限制下，USB 方案或許是一種現實的折衷選擇。
{% end %}

## 架構設計與資料流向

讓我們先理解整個系統的架構。

<figure>
{{ image(url="external-network-flow.svg", alt="外網環境：開發機與 USB 同步流程圖", no_hover=true) }}
<figcaption>外網環境：開發機與 USB 同步流程圖</figcaption>
</figure>

<figure>
{{ image(url="internal-network-flow.svg", alt="內網環境：中介機操作 USB 與內網 Git 伺服器同步流程圖", no_hover=true) }}
<figcaption>內網環境：中介機操作 USB 與內網 Git 伺服器同步流程圖</figcaption>
</figure>

### 四層架構說明

這個工作流程包含四個獨立的層級：

1. **外網開發機（A 電腦）**：這是主要的開發環境，包含完整的 Git 工作目錄。開發者在這裡編寫程式碼、提交變更、建立分支。由於需要安裝各種開發工具，這台機器被視為相對危險，嚴格禁止連接內網。

2. **USB 隨身碟（裸儲存庫）**：作為物理隔離環境下的資料載體，儲存著 Git 的完整歷史記錄。使用裸儲存庫而非標準儲存庫，能有效延長 USB 壽命並符合 Git 的設計哲學。這是連接外網與內網的物理橋樑。

3. **內網中介機（B 電腦）**：這是關鍵的安全檢查點。這台機器同時擁有 USB 存取權限和內網連線權限，但不作為開發環境使用。它的角色是接收來自外網開發機的變更，進行檢視後，才推送到內網 Git 伺服器。

4. **內網 Git 伺服器**：團隊的中央儲存庫，位於受保護的內網環境中。可能是 GitHub Enterprise Server、GitLab 自架版、Azure DevOps Server 等企業級解決方案。這是整個架構的最終目的地，也是團隊協作的核心。

### 為什麼需要中介機這一層

{% chat(speaker="user") %}
我看過一些 Git Bare Repo 的教學，都是直接從 USB 鏡像推送到遠端儲存庫。為什麼要多加這個中介機？不是讓流程更複雜了嗎？
{% end %}

{% chat(speaker="jim") %}
這是個好問題。確實，技術上可以省略中介機，讓 USB 直接作為推送來源。但在企業環境中，這個中介層的價值在於「人工檢查點」。

當你使用 `git clone --mirror` 建立鏡像儲存庫到 USB 上時，這個儲存庫會包含所有的 refs，包括所有分支和標籤。如果不小心執行了 `git push --mirror` 將 USB 內容鏡像推送，很可能會意外覆蓋或刪除內網儲存庫的分支。在團隊協作環境中，這種錯誤的代價很高。

透過內網中介機，你可以：

1. 先從 USB 獲取變更（`git fetch usb`）
2. 在推送前檢視變更內容（`git log`, `git diff`）
3. 確認沒有敏感資訊洩漏、檢視將要推送的提交
4. 確認無誤後才推送到內網（`git push origin`）

這個額外的步驟在技術上提供緩衝，在組織流程上則是一道保險。特別是在嚴格的企業環境中，這種「雙重確認」機制能避免許多人為錯誤。

四層架構的核心價值就在於此：不只是物理隔離（外網 ↔ USB ↔ 內網），更在於邏輯上的驗證層級（開發 → 傳輸 → 檢視 → 推送）。
{% end %}

## Git 裸儲存庫的核心概念

在深入實作之前，我們需要先理解「裸儲存庫」的本質。

{% chat(speaker="user") %}
我聽過裸儲存庫這個詞，但一直不太清楚它和一般的 Git 儲存庫有什麼差別？
{% end %}

{% chat(speaker="jim") %}
簡單來說，裸儲存庫就是把一般儲存庫中 `.git` 資料夾的內容，直接作為整個儲存庫的根目錄。它不包含工作目錄，所以你看不到專案的實際檔案，只有 Git 的 Metadata。

舉個例子，當你執行 `git clone https://example.com/repo.git` 時，會建立一個標準儲存庫，結構像這樣：

```
my-project/
├── .git/          (版本控制資料)
├── src/           (實際程式碼)
├── README.md
└── package.json
```

而當你執行 `git clone --bare https://example.com/repo.git` 或 `git clone --mirror` 時，建立的是裸儲存庫，結構像這樣：

```
my-project.git/
├── HEAD
├── config
├── objects/
├── refs/
└── hooks/
```

這就是原本 `.git` 資料夾裡的內容。裸儲存庫不能直接在裡面編輯程式碼或提交變更，但它是理想的「中繼站」，專門用來接收和發送 Git 資料。
{% end %}

### 為什麼在 USB 上使用裸儲存庫

這個選擇背後有兩個重要考量：

#### 延長 USB 隨身碟壽命

USB 隨身碟使用的快閃記憶體有寫入次數限制。當你在標準儲存庫中執行 `git status`、`git add`、`git commit` 等指令時，Git 會頻繁讀寫 `.git` 資料夾內的索引檔案、物件資料庫和日誌。這些零碎的讀寫操作會加速快閃記憶體的耗損。

相對地，裸儲存庫的使用模式是「批次寫入」。你只在同步時執行 `git push`，這是一次性的資料傳輸，而非持續的零碎讀寫。這種使用方式可以延長 USB 隨身碟的使用壽命。

#### 符合 Git 的設計哲學

Git 的分散式架構中，裸儲存庫扮演著「中央集散地」的角色。它被設計來接收推送、提供複製來源，而不是直接在其中進行開發。這正好符合我們在 USB 上的使用情境。

{% chat(speaker="jim") %}
不過要記住，即使使用裸儲存庫能延長壽命，USB 隨身碟仍然是會故障的設備。一定要養成定期同步到內網儲存庫的習慣，不要讓 USB 成為唯一的程式碼備份點。

另外在企業環境中，這個 USB 隨身碟包含完整的專案歷史，遺失可能造成資訊安全問題。一定要妥善保管。
{% end %}

## 完整設定流程

現在讓我們逐步建立這套工作流程。我會盡可能詳細地說明每個步驟的目的和注意事項。

### 步驟一：在內網中介機建立環境

這是整個流程的起點。我們需要在有內網權限的機器上，同時建立本地完整儲存庫和 USB 裸儲存庫。

#### 1.1 準備 PowerShell 環境

在 Windows 企業環境中，PowerShell 的執行政策可能會阻止腳本執行。首先需要調整執行政策：

```powershell
# 以系統管理員權限開啟 PowerShell
# 設定執行政策為 RemoteSigned
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

`RemoteSigned` 政策允許執行本地建立的腳本，但要求從網路下載的腳本必須經過數位簽章。這在企業環境中是常見且合理的設定。

#### 1.2 建立本地完整儲存庫

```powershell
# 假設你的工作目錄在 C:\Development
cd C:\Development

# 從內網 Git 伺服器複製儲存庫
# 將 <internal-repository-url> 替換為你的內網儲存庫位址
git clone https://git.internal.company.com/team/project.git
```

這個本地儲存庫將成為你的「檢查站」。所有從外網開發機來的變更，都會先在這裡進行檢視和驗證。

#### 1.3 建立 USB 裸儲存庫

將 USB 隨身碟連接到內網中介機。假設磁碟機代號為 `D:`。

```powershell
# 移動到 USB 隨身碟
cd D:\

# 使用 --mirror 參數建立鏡像儲存庫
git clone --mirror https://git.internal.company.com/team/project.git
```

執行後會在 USB 上建立 `project.git` 資料夾。依照慣例，裸儲存庫的資料夾名稱會以 `.git` 結尾，這能清楚表明它是裸儲存庫而非標準儲存庫。

{% chat(speaker="user") %}
我看到你用的是 `--mirror` 參數，這和 `--bare` 有什麼不同嗎？
{% end %}

{% chat(speaker="jim") %}
`--mirror` 可以視為 `--bare` 的加強版。它不僅建立裸儲存庫，還會：

1. 設定遠端儲存庫的 fetch refspec，使其能同步所有 refs（包括 `refs/heads/*`、`refs/tags/*`、`refs/remotes/*`）
2. 自動設定 `remote.origin.mirror` 為 true

在這個場景中使用 `--mirror` 的好處是，它能確保 USB 上的儲存庫是內網遠端的完整鏡像。雖然在本文的流程中我們不會直接從 USB 推送到內網（而是透過中介層），但保持完整的鏡像狀態能讓同步腳本的邏輯更簡單。
{% end %}

#### 1.4 設定雙向遠端連結

現在我們要讓本地完整儲存庫能同時與 USB 和內網伺服器溝通：

```powershell
# 回到內網中介機的本地儲存庫
cd C:\Development\project

# 新增 USB 裸儲存庫作為第二個遠端來源
git remote add usb D:\project.git

# 確認遠端設定
git remote -v
```

執行後你會看到類似這樣的輸出：

```
origin  https://git.internal.company.com/team/project.git (fetch)
origin  https://git.internal.company.com/team/project.git (push)
usb     D:\project.git (fetch)
usb     D:\project.git (push)
```

這表示你的本地儲存庫現在有兩個遠端來源：

- `origin`：指向內網的 Git 伺服器
- `usb`：指向 USB 上的裸儲存庫

### 步驟二：在外網開發機設定開發環境

將 USB 隨身碟從內網中介機拔除，連接到外網開發機。

#### 2.1 設定檔案協定權限

Git 預設會限制透過檔案協定（file protocol）存取儲存庫，這是基於安全考量。在受控的企業環境中，我們需要明確允許這個行為：

```powershell
# 允許透過檔案協定複製和推送
git config --global protocol.file.allow always
```

這個設定只需要執行一次，會永久儲存在全域 Git 設定中。

#### 2.2 從 USB 複製儲存庫

```powershell
# 假設 USB 在外網開發機上的磁碟機代號為 E:
# 移動到你的工作目錄
cd C:\Development

# 從 USB 裸儲存庫複製
git clone E:\project.git
```

Git 會自動將 USB 設定為名為 `origin` 的遠端來源。這樣的設定很直覺：對外網開發機來說，USB 就是它的「遠端儲存庫」。

#### 2.3 開始開發工作

現在你可以在外網環境中正常使用 Git 的所有功能：

```powershell
# 進入專案目錄
cd project

# 建立功能分支
git checkout -b feature/new-awesome-feature

# 進行開發工作...
# 編輯檔案、測試、除錯...

# 提交變更
git add .
git commit -m "feat: implement awesome new feature"

# 繼續開發...
git add .
git commit -m "fix: handle edge case in awesome feature"
```

{% chat(speaker="jim") %}
在外網環境開發時，我建議養成頻繁提交的習慣。由於無法即時推送到內網備份，本地提交就是你唯一的安全網。使用描述性的提交訊息，未來回顧時會更容易理解。

另外要特別注意：絕對不要在外網環境將程式碼推送到外部的 Git 平台（如 GitHub.com、GitLab.com）。程式碼只能存在於：外網開發機、USB、內網中介機、內網 Git 伺服器這四個地方。
{% end %}

#### 2.4 推送變更到 USB

完成一個階段的開發後，將變更推送到 USB：

```powershell
# 推送當前分支到 USB
git push origin feature/new-awesome-feature

# 或者使用 HEAD 推送當前分支
git push origin HEAD
```

這時候你的變更已經安全地儲存在 USB 上，可以準備帶回內網中介機進行下一步同步。

### 步驟三：在內網中介機同步到內網伺服器

將 USB 隨身碟帶回內網中介機。這是整個流程中最關鍵的環節，我們要透過本地完整儲存庫，安全地將變更推送到內網。

#### 3.1 從 USB 獲取變更

```powershell
# 進入內網中介機的本地完整儲存庫
cd C:\Development\project

# 從 USB 獲取所有變更
git fetch usb

# 查看獲取到的分支
git branch -r | Select-String "usb/"
```

這時候你會看到從 USB 獲取到的所有分支，例如 `usb/feature/new-awesome-feature`。

#### 3.2 檢視並建立本地分支

在推送到內網之前，先檢視變更內容：

```powershell
# 查看特定分支的提交記錄
git log usb/feature/new-awesome-feature --oneline -10

# 查看與 master 的差異
git diff master..usb/feature/new-awesome-feature

# 確認無誤後，建立本地追蹤分支
git checkout -b feature/new-awesome-feature usb/feature/new-awesome-feature
```

這個步驟提供了最後的檢查機會。你可以確認：

- 提交訊息是否清楚明確
- 沒有包含敏感資訊（密碼、API 金鑰等）
- 程式碼品質符合團隊標準
- 沒有意外的檔案變更
- 沒有違反企業資安政策的內容

#### 3.3 推送到內網 Git 伺服器

確認無誤後，推送到內網：

```powershell
# 推送到內網 Git 伺服器
git push origin feature/new-awesome-feature
```

至此，完整的同步流程結束。你的變更已經安全地從外網開發機，經過 USB 和內網中介機的檢查，最終抵達內網 Git 伺服器。

## 日常工作流程

理解了完整的設定流程後，讓我們看看日常開發中的典型操作循環。

### 開發週期開始：從內網同步最新狀態

每天開始工作前，應該確保 USB 上的內容是最新的。

**在內網中介機操作：**

```powershell
# 更新本地儲存庫
cd C:\Development\project
git checkout master
git pull origin master

# 同步到 USB（確保 USB 已連接）
cd D:\project.git
git remote update
```

這樣就能確保 USB 上的裸儲存庫包含最新的團隊變更。

### 外網開發：安全地更新本地 master

將 USB 帶到外網開發機後，第一件事是更新本地的 master 分支：

```powershell
cd C:\Development\project

# 從 USB 獲取更新
git fetch origin

# 使用 --ff-only 進行快速前進合併
git checkout master
git merge --ff-only origin/master
```

{% chat(speaker="user") %}
為什麼要用 `--ff-only` 參數？直接 `git pull` 不是更簡單嗎？
{% end %}

{% chat(speaker="jim") %}
`--ff-only` 是一個安全措施。它表示「只有在可以快速前進（fast-forward）的情況下才合併」。如果你的本地 master 有未同步的提交，快速前進就不可能發生，合併會失敗並給出錯誤訊息。

這強迫你意識到「master 分支不同步」這個問題，而不是讓 Git 自動建立合併提交。在團隊協作中，本地 master 應該永遠是內網 master 的鏡像，不應該有獨立的提交。如果 `--ff-only` 失敗，表示有些地方出錯了，你需要手動處理。

當然，如果你確定要放棄本地 master 的變更，也可以使用：

```powershell
git reset --hard origin/master
```

但這要非常小心，因為會永久丟失本地提交。
{% end %}

### 建立功能分支並開發

```powershell
# 基於最新的 master 建立功能分支
git checkout -b feature/user-authentication master

# 進行開發...
# 多次提交...

# 完成後推送到 USB
git push origin feature/user-authentication
```

### 同步回內網並建立 Pull Request

將 USB 帶回內網中介機：

```powershell
# 在內網中介機
cd C:\Development\project
git fetch usb
git checkout -b feature/user-authentication usb/feature/user-authentication

# 最後的程式碼審查...
git log --oneline -10
git diff master..feature/user-authentication

# 推送到內網
git push origin feature/user-authentication
```

接著透過你們使用的 Git 平台（GitHub Enterprise、GitLab、Azure DevOps 等）建立 Pull Request，讓團隊成員進行 Code Review。

## 自動化腳本

手動執行這些 Git 指令容易出錯，特別是在內網中介機上的同步操作。接下來我會分享兩個 PowerShell 腳本，分別處理單一儲存庫和批次處理多個儲存庫的場景。

### 腳本一：同步單一儲存庫（Sync-From-Remote.ps1）

這個腳本的主要功能是將內網 Git 伺服器的最新狀態完整同步到 USB 的鏡像儲存庫。它會自動偵測儲存庫類型（鏡像或標準裸儲存庫），並使用適當的同步策略。

<details>
<summary>點擊展開完整腳本</summary>

```powershell
# Sync-From-Remote.ps1
# 從內網 Git 伺服器同步最新變更到 USB 裸儲存庫的自動化腳本
#
# 使用方式:
#   .\Sync-From-Remote.ps1 'D:\your-project.git'
#
# 注意:
# - 此腳本會將內網遠端狀態完全同步到 USB，包括刪除遠端已不存在的分支和標籤
# - 支援鏡像儲存庫 (--mirror) 和標準裸儲存庫
# - 執行前請確保 USB 儲存庫是裸儲存庫且可以安全覆蓋

param (
    [Parameter(Mandatory=$true, Position=0)]
    [string]$UsbRepoPath
)

# 檢查路徑是否存在
if (-not (Test-Path $UsbRepoPath)) {
    Write-Error "錯誤：找不到指定的 USB 儲存庫路徑: $UsbRepoPath"
    exit 1
}

try {
    Write-Host "正在進入 USB 裸儲存庫目錄: $UsbRepoPath" -ForegroundColor Cyan
    Push-Location -Path $UsbRepoPath
    
    # 偵測儲存庫類型
    # 鏡像儲存庫不包含 refs/remotes 目錄
    $isMirrorRepo = -not (Test-Path "refs/remotes")
    
    if ($isMirrorRepo) {
        Write-Host "偵測到鏡像儲存庫，使用鏡像同步模式" -ForegroundColor Yellow
        
        # 對於鏡像儲存庫，使用 git remote update 同步所有 refs
        Write-Host "正在從內網遠端同步所有 refs..." -ForegroundColor Cyan
        git remote update
        
        # 驗證同步結果
        $localHeads = git for-each-ref --format='%(refname:short)' refs/heads
        if ($localHeads.Count -eq 0) {
            Write-Warning "同步後未找到任何分支"
        } else {
            Write-Host "成功同步 $($localHeads.Count) 個分支：$($localHeads -join ', ')" -ForegroundColor Green
        }
    }
    else {
        Write-Host "偵測到標準裸儲存庫，使用標準同步模式" -ForegroundColor Yellow
        
        # 獲取遠端分支和標籤，並清理已刪除的遠端 refs
        Write-Host "正在抓取遠端分支和標籤..." -ForegroundColor Cyan
        git fetch origin --prune --tags

        # 同步遠端分支到本地 heads
        Write-Host "正在同步分支..." -ForegroundColor Cyan
        $remoteBranchRefs = git for-each-ref --format='%(refname:short)' refs/remotes/origin
        foreach ($remoteRef in $remoteBranchRefs) {
            # 跳過 HEAD 參照
            if ($remoteRef -like 'origin/HEAD') { continue }
            
            $localHead = $remoteRef -replace '^origin/',''
            Write-Host "  同步分支: $localHead" -ForegroundColor Gray
            git update-ref -m "sync from internal remote" "refs/heads/$localHead" "refs/remotes/origin/$localHead"
        }

        # 移除本地已不存在於遠端的分支
        Write-Host "正在清理已刪除的遠端分支..." -ForegroundColor Cyan
        $localHeads = git for-each-ref --format='%(refname:short)' refs/heads
        foreach ($localHead in $localHeads) {
            if (-not (git show-ref -q --verify "refs/remotes/origin/$localHead")) {
                Write-Host "  刪除分支: $localHead" -ForegroundColor Yellow
                git update-ref -d "refs/heads/$localHead"
            }
        }
    }

    # 標籤同步（對兩種儲存庫類型都適用）
    Write-Host "正在同步標籤..." -ForegroundColor Cyan
    
    # 取得遠端所有標籤
    $remoteTags = git ls-remote --tags origin --refs | ForEach-Object {
        ($_ -split "`t")[1] -replace '^refs/tags/',''
    } | Where-Object { $_ -ne '' } | Sort-Object -Unique

    # 取得本地所有標籤
    $localTags = git for-each-ref --format='%(refname:short)' refs/tags | Sort-Object -Unique

    # 刪除本地存在但遠端已不存在的標籤
    foreach ($localTag in $localTags) {
        if (-not ($remoteTags -contains $localTag)) {
            Write-Host "  刪除標籤: $localTag" -ForegroundColor Yellow
            git tag -d $localTag
        }
    }

    Write-Host "同步完成！" -ForegroundColor Green
}
catch {
    Write-Error "發生錯誤: $_"
    exit 1
}
finally {
    Pop-Location
    Write-Host "已返回原始目錄" -ForegroundColor Gray
}
```

</details>

#### 基本使用

```powershell
# 同步單一儲存庫
.\Sync-From-Remote.ps1 'D:\your-project.git'
```

#### 腳本特色

1. **自動偵測儲存庫類型**：腳本會檢查 `refs/remotes` 目錄是否存在，自動判斷是鏡像儲存庫還是標準裸儲存庫，並採用對應的同步策略。

2. **完整同步**：同步新的變更，同時清理遠端已刪除的分支和標籤，讓 USB 上的狀態與內網完全一致。

3. **安全性檢查**：在開始執行前會驗證路徑是否存在，並在整個過程中提供詳細的操作回饋。

4. **錯誤處理**：使用 `try-catch-finally` 結構，即使發生錯誤也能確保返回原始目錄。

{% chat(speaker="jim") %}
執行這個腳本前要注意：它會將 USB 上的儲存庫狀態強制對齊內網遠端。如果 USB 上有未推送的變更，這些變更會被丟棄。所以執行前一定要確認所有重要的變更都已經推送到內網。

另外，Windows 的 PowerShell 預設會要求腳本檔案以「UTF-8 with BOM」編碼儲存。如果腳本執行時出現編碼錯誤，可以用記事本或 VS Code 重新儲存檔案，確保使用正確的編碼。
{% end %}

### 腳本二：批次同步所有儲存庫（Sync-All-From-Remote.ps1）

當你的 USB 上有多個專案儲存庫時，逐一手動同步會很繁瑣。這個腳本能自動掃描指定目錄下所有以 `.git` 結尾的資料夾，並批次執行同步。

<details>
<summary>點擊展開完整腳本</summary>

```powershell
# Sync-All-From-Remote.ps1
# 批次同步所有 USB 裸儲存庫從內網 Git 伺服器的自動化腳本
#
# 使用方式:
#   .\Sync-All-From-Remote.ps1
#   .\Sync-All-From-Remote.ps1 'D:\'
#
# 注意:
# - 此腳本會掃描指定目錄下所有以 .git 結尾的資料夾
# - 會對每個儲存庫執行 Sync-From-Remote.ps1
# - 若某個儲存庫同步失敗，會記錄錯誤但繼續處理其他儲存庫

param (
    [Parameter(Mandatory=$false, Position=0)]
    [string]$BaseDirectory = (Get-Location).Path
)

# 檢查基礎目錄是否存在
if (-not (Test-Path $BaseDirectory)) {
    Write-Error "錯誤：找不到指定的基礎目錄: $BaseDirectory"
    exit 1
}

# 檢查 Sync-From-Remote.ps1 是否存在
$syncScript = Join-Path $PSScriptRoot "Sync-From-Remote.ps1"
if (-not (Test-Path $syncScript)) {
    Write-Error "錯誤：找不到 Sync-From-Remote.ps1 腳本在: $syncScript"
    Write-Host "請確保 Sync-From-Remote.ps1 與此腳本位於同一目錄" -ForegroundColor Yellow
    exit 1
}

Write-Host "`n========================================" -ForegroundColor Cyan
Write-Host "  批次同步所有儲存庫從內網遠端" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "基礎目錄: $BaseDirectory`n" -ForegroundColor Gray

# 查找所有 .git 目錄
$gitDirs = Get-ChildItem -Path $BaseDirectory -Directory -Filter "*.git"

if ($gitDirs.Count -eq 0) {
    Write-Warning "在 $BaseDirectory 中沒有找到任何 .git 儲存庫"
    exit 0
}

Write-Host "找到 $($gitDirs.Count) 個儲存庫：" -ForegroundColor Yellow
foreach ($dir in $gitDirs) {
    Write-Host "  ✓ $($dir.Name)" -ForegroundColor Gray
}
Write-Host ""

# 統計變數
$successCount = 0
$errorCount = 0
$errors = @()
$startTime = Get-Date

# 處理每個儲存庫
foreach ($gitDir in $gitDirs) {
    $repoPath = $gitDir.FullName
    $repoName = $gitDir.Name
    
    Write-Host "----------------------------------------" -ForegroundColor DarkGray
    Write-Host "正在同步: $repoName" -ForegroundColor Cyan
    Write-Host "----------------------------------------" -ForegroundColor DarkGray
    
    try {
        # 執行 Sync-From-Remote.ps1 腳本
        & $syncScript $repoPath
        
        if ($LASTEXITCODE -eq 0) {
            Write-Host "✓ $repoName 同步成功`n" -ForegroundColor Green
            $successCount++
        } else {
            Write-Host "✗ $repoName 同步失敗 (退出代碼: $LASTEXITCODE)`n" -ForegroundColor Red
            $errors += "$repoName - 退出代碼: $LASTEXITCODE"
            $errorCount++
        }
    }
    catch {
        Write-Host "✗ $repoName 同步時發生例外: $_`n" -ForegroundColor Red
        $errors += "$repoName - 例外: $_"
        $errorCount++
    }
}

# 計算總耗時
$endTime = Get-Date
$duration = $endTime - $startTime

# 顯示總結
Write-Host "`n========================================" -ForegroundColor Cyan
Write-Host "  同步作業完成" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "總耗時: $($duration.ToString('mm\:ss'))" -ForegroundColor Gray
Write-Host "成功: $successCount 個儲存庫" -ForegroundColor Green
Write-Host "失敗: $errorCount 個儲存庫" -ForegroundColor $(if ($errorCount -gt 0) { "Red" } else { "Green" })

if ($errors.Count -gt 0) {
    Write-Host "`n錯誤詳情：" -ForegroundColor Red
    foreach ($errorMsg in $errors) {
        Write-Host "  • $errorMsg" -ForegroundColor Red
    }
    Write-Host ""
}

# 如果有錯誤，以非零退出代碼結束
if ($errorCount -gt 0) {
    exit 1
}

Write-Host "所有儲存庫已成功同步！`n" -ForegroundColor Green
```

</details>

#### 基本使用

```powershell
# 在當前目錄下尋找並同步所有 .git 儲存庫
.\Sync-All-From-Remote.ps1

# 指定特定目錄
.\Sync-All-From-Remote.ps1 'D:\'
```

#### 腳本特色

1. **自動掃描**：會自動找出指定目錄下所有以 `.git` 結尾的資料夾，不需要手動指定每個儲存庫。

2. **批次處理**：逐一處理找到的儲存庫，即使某個儲存庫同步失敗，也會繼續處理剩下的儲存庫。

3. **詳細報告**：執行完畢後會顯示總結資訊，包括成功和失敗的數量、總耗時，以及失敗儲存庫的詳細錯誤訊息。

4. **依賴檢查**：腳本會檢查 `Sync-From-Remote.ps1` 是否存在於同一目錄，確保能正確呼叫同步功能。

{% chat(speaker="user") %}
這兩個腳本都很實用，但我注意到它們都是「從內網同步到 USB」。那從「USB 同步到內網」不需要腳本嗎？
{% end %}

{% chat(speaker="jim") %}
很好的觀察！確實，我只提供了「下載」方向的自動化腳本，而沒有提供「上傳」方向的。我是故意這麼設計的。

從內網同步到 USB 是一個「安全」的操作。它只是把已經在團隊內網儲存庫中的內容複製到 USB，不會影響任何人的工作。所以我們可以放心地自動化這個過程。

但從 USB 同步到內網就不一樣了。這涉及到推送新的變更到團隊共用的儲存庫，應該經過人工檢視。前面提到的「三層架構的中介層價值」就在這裡體現：透過手動的 `git fetch usb`、`git checkout`、`git log`、`git diff` 等指令，你有機會在推送前檢視和驗證變更內容。

自動化應該用在重複性高、風險低的操作上。對於可能影響團隊的關鍵步驟，保持手動才是明智的選擇。請千萬別想著自己寫上傳腳本，這不是留給你的回家作業！😅
{% end %}

## 常見問題與疑難排解

在實際使用這套工作流程時，可能會遇到一些問題。讓我們看看最常見的情況和解決方案。

### 問題一：USB 複製的儲存庫顯示為空

**症狀：** 在外網開發機執行 `git clone E:\your-project.git` 後，進入專案目錄發現沒有任何檔案，或是執行 `git branch` 沒有顯示任何分支。

**可能原因：**

1. USB 上的裸儲存庫損壞或不完整
2. 檔案協定權限未正確設定

**解決方案：**

```powershell
# 在外網開發機
# 1. 確認檔案協定權限
git config --global protocol.file.allow always

# 2. 檢查 USB 儲存庫結構
cd E:\your-project.git
dir refs\heads  # 應該能看到分支檔案

# 如果 refs\heads 是空的，表示 USB 儲存庫有問題
# 需要回到內網中介機重新建立

# 在內網中介機
# 3. 重新建立 USB 鏡像儲存庫
Remove-Item -Recurse -Force D:\your-project.git
git clone --mirror https://git.internal.company.com/team/project.git D:\your-project.git
```

### 問題二：推送時發生衝突

**症狀：** 在內網中介機將 USB 的變更推送到內網時，Git 回報衝突或拒絕推送。

**可能原因：** USB 上的儲存庫狀態過舊，與內網的最新狀態有分歧。

**解決方案：**

這是一個棘手的情況，因為你的外網開發工作已經完成，但基礎版本過舊。推薦的處理流程是：

```powershell
# 在內網中介機
cd C:\Development\project

# 1. 更新本地 master 到最新狀態
git checkout master
git pull --ff-only origin master

# 2. 從 USB 建立功能分支
git fetch usb
git checkout -b feature/my-work usb/feature/my-work

# 3. Rebase 到最新的 master
git rebase master

# 這時候 Git 會逐一應用你的提交到最新的 master 上
# 如果有衝突，按照 Git 的指示解決：
#   1. 編輯衝突檔案
#   2. git add <resolved-files>
#   3. git rebase --continue

# 4. 解決所有衝突後，推送到內網
git push origin feature/my-work
```

**預防措施：** 養成每天開始工作前先同步 USB 的習慣。開發週期越短，遇到衝突的機率就越低。或者是跟同事約在廁所後面一較高下，打輸的人負責解決衝突。😆

### 問題三：PowerShell 執行政策錯誤

**症狀：** 執行腳本時出現類似以下的錯誤訊息：

```
.\Sync-From-Remote.ps1 : 無法載入檔案，因為這個系統上已停用指令碼執行。
```

**解決方案：**

```powershell
# 檢查目前的執行政策
Get-ExecutionPolicy -List

# 為當前使用者設定 RemoteSigned 政策
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser

# 如果上述指令權限不足，嘗試以系統管理員身分開啟 PowerShell 後執行
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
```

## 最佳實踐建議

經過實際使用這套工作流程後，我整理了一些能讓流程更順暢的建議。

### 1. 建立清晰的分支命名規範

在氣隙環境中，清晰的分支命名特別重要，因為你可能有一段時間看不到團隊其他人的進度。一般來說建議採用這樣的命名規則：

<mark>類型/簡短描述</mark>

```
feat/user-authentication
fix/login-timeout
hotfix/security-patch
docs/api-documentation
```

不過，使用人名作為分支其實也很棒，可讀性高，大家不會打架：

<mark>誰/幹嘛</mark>

```
jim/implement-oauth
john/add-unit-tests
joe/fix-ui-bug
```

團隊大家講好就行。

### 2. 養成頻繁提交的習慣

在外網環境中，本地提交是你唯一的安全網。建議：

- 每完成一個小功能就提交
- 即使程式碼還不完美，也可以先提交，之後再用 `git commit --amend` 或 `git rebase -i` 整理
- 使用描述性的提交訊息，說明「為什麼」而不只是「做了什麼」

例如：

```bash
# 不好的提交訊息
git commit -m "update code"

# 好的提交訊息
git commit -m "feat: add email validation to prevent invalid user registrations

- Implement regex-based email format checking
- Add unit tests for validation logic
- Update user registration form with inline error messages"
```

### 3. 定期同步，避免長期分歧

理想的同步頻率是：

- **每天開始工作前**：將最新的內網狀態同步到 USB
- **每天結束工作後**：將當天的變更推送到內網（如果功能還沒完成，也可以推送到個人的功能分支）
- **重大功能完成後**：立即同步並建立 Pull Request

長期不同步的風險：

- 合併衝突的複雜度會隨時間增加
- 可能重複實作其他人已經完成的功能
- USB 損壞時的損失會更大

### 4. 妥善管理 USB 隨身碟

- **物理安全**：USB 隨身碟中包含完整的專案歷史，遺失可能造成資訊安全問題，妥善保管。
- **定期檢查健康狀態**：不要等到 USB 完全損壞才發現問題。定期檢查是否有讀寫錯誤。
- **遵守資安政策**：絕對不要將程式碼存放在外部雲端或個人裝置上。程式碼只能存在於：外網開發機、USB、內網中介機、內網 Git 伺服器這四個地方。

### 5. 在內網中介機保持警覺

從 USB 推送到內網之前，養成這些檢查習慣：

```powershell
# 查看即將推送的提交
git log origin/master..feature/my-branch --oneline

# 查看具體的程式碼變更
git diff origin/master..feature/my-branch
```

這個「最後檢查點」是四層架構的核心價值。不要因為習慣而跳過這個步驟。

### 6. 文件化你的工作流程

為你的團隊建立一份專屬的操作文件，包括：

- 你們的內網 Git 儲存庫位址
- 分支命名規範
- 常見問題的解決方案
- 不要忘記把這篇文章連結加入到文件中，讓團隊成員能隨時參考。😎

## 結語

透過 Git 裸儲存庫和 USB 隨身碟建立的四層架構工作流程，為企業氣隙環境下的版本控制提供了一個可靠的解決方案。這套方法的關鍵價值在於：

**符合資安政策**：透過物理隔離來符合企業對內外網嚴格分離的要求。外網開發機無法連接內網，所有程式碼傳輸都透過受控的 USB 隨身碟和內網中介機。

**安全性優先**：中介層的設計提供了緩衝機制，讓團隊能在變更真正推送到內網之前，有機會進行最後的檢視和驗證。

但這套方法也有其限制。它要求開發者保持紀律，定期同步，妥善管理 USB 隨身碟。它不適合需要即時協作的場景，也無法完全取代正常的網路環境。它更像是在嚴格資安政策限制下的「最佳變通方案」。

{% chat(speaker="jim") %}
這套流程看起來複雜，但用過幾次就會成為自然的習慣。最重要的是理解每個步驟背後的「為什麼」，而不是機械式地執行指令。當你真正理解了四層架構的價值，就能在遇到問題時做出正確的判斷。

如果你在實作過程中遇到任何問題，或是有更好的做法想分享，歡迎在下方透過 Mastodon 留言討論。😀
{% end %}
