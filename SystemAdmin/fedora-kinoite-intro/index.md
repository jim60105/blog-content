+++
title = "Fedora Kinoite 開發者快速上手指南"
description = "從傳統 Linux 轉移到 Fedora Kinoite？本指南教你使用 Toolbx 建立獨立開發環境、透過 Flatpak 安裝桌面應用程式、用 rpm-ostree 管理系統套件。包含常見問題解答與快速對照表,讓你快速上手原子化系統。"
date = "2025-12-15T09:42:16.460Z"
updated = "2025-12-15T09:42:16.795Z"
draft = false
aliases = [ ]

[extra]
banner = "preview.png"

[taxonomies]
tags = [ "Linux", "RHEL/Fedora", "System Admin" ]
licenses = [ "All Rights Reserved" ]
+++
{% alert(edit=true) %}
這篇文章是我寫給同事的指南，放在公司電腦的桌面上，目的是讓下一位接手這台電腦的人能夠順利使用 Fedora Kinoite，而不是把它重灌成 Ubuntu 或 Windows 🙏

在這裡也分享給大家參考。
{% end %}

Hi, 我是 Jim。

歡迎使用 Fedora Kinoite！這是一套基於 KDE Plasma 桌面的「原子化」(Atomic) 作業系統。如果你第一次使用它，這份指南會幫助你快速上手。

---

## 為什麼不要把機器重灌成 Ubuntu 或 Windows？

Fedora Kinoite 帶來幾個傳統系統做不到的好處：

{{ cg(body="系統永遠不會壞掉") }}：更新是原子化操作，要嘛整個成功、要嘛整個失敗。不會發生「更新到一半斷電，系統就開不了機」的慘況。每次更新前都會自動保留前一個版本，出問題可以隨時 rollback 回去。

{{ cg(body="開發環境乾淨隔離") }}：透過 Toolbx 容器，你可以為每個專案建立獨立的開發環境。Python 3.10 和 Python 3.12 專案能各自有自己的容器，刪掉容器就能清除所有髒東西，不會像傳統系統那樣越用越亂。

{{ cg(body="SELinux 預設開啟且完整支援") }}：這是企業級的安全機制，可以防止惡意程式取得不該有的權限。很多人在其他發行版會因為「太麻煩」而關掉它，但 Kinoite 的設計讓你幾乎不會遇到 SELinux 造成的問題，因為大部分軟體都跑在容器或 Flatpak 沙盒裡。

{{ cg(body="由 Red Hat 贊助") }}：Fedora 是 RHEL 的上游，這裡學到的東西可以直接套用到企業環境。

{{ cg(body="它是 Linux") }}：Linux 是自由的。我喜歡自由，我希望你也喜歡。😉

---

## 核心概念：三種安裝軟體的方式

在 Kinoite 上，你有三種方式安裝軟體，各有不同用途：

**Flatpak**：用於安裝圖形介面應用程式。這些 App 執行在沙盒中，有權限控管，且跨版本更新不受影響。一般來說，桌面應用程式優先使用 Flatpak 安裝。

**Toolbx**：用於開發工作。在容器內安裝編譯器、程式語言、開發工具，用在有著大量依頼且常常變更的情況。

**rpm-ostree**：用於系統層級的套件，例如驅動程式、虛擬化工具。除非必要，儘量避免使用。在這裡安裝的程式越多，每次變更系統或套件就要越久的時間，並且會使得系統逐漸不穩定。在原子系統的概念中，安裝任一個套件時都會重裝整個系統！

---

## Flatpak：安裝桌面應用程式

Flatpak 應用程式可以透過 Plasma Discover 圖形介面安裝，或者用指令：

```bash
# 加入 Flathub 套件庫（通常已內建）
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# 安裝應用程式
flatpak install flathub org.mozilla.firefox
flatpak install flathub com.spotify.Client
flatpak install flathub com.visualstudio.code

# 執行應用程式
flatpak run com.visualstudio.code

# 更新所有 Flatpak 應用程式
flatpak update

# 列出已安裝的應用程式
flatpak list
```

📖 更多資訊：<https://flatpak.org/setup/Fedora/>

---

## Toolbx：開發者的最佳夥伴

Toolbx 讓你在容器中進行開發，同時保持與主系統的無縫整合。容器內可以直接存取你的 home 目錄、SSH agent、網路、USB 裝置等。

### 基本操作

```bash
# 建立預設容器（會使用與主系統相同版本的 Fedora）
toolbox create

# 進入容器（提示符會出現 ⬢ 符號）
toolbox enter

# 在容器內，你可以自由安裝任何套件
⬢[user@toolbox ~]$ sudo dnf install nodejs npm python3 gcc

# 離開容器
⬢[user@toolbox ~]$ exit

# 列出所有容器
toolbox list

# 刪除容器
toolbox rm fedora-toolbox-42
```

### 為不同專案建立獨立環境

```bash
# 建立指定名稱的容器
toolbox create --container project-a
toolbox create --container project-b

# 使用不同版本的 Fedora
toolbox create --release 41 --container legacy-project

# 使用 Ubuntu 環境
toolbox create --distro ubuntu --release 22.04

# 進入特定容器
toolbox enter --container project-a

# 不進入容器直接執行指令
toolbox run --container project-a python3 script.py
```

### 在容器內使用 GUI 應用程式

Toolbx 容器可以執行圖形介面程式，它們會直接顯示在你的桌面上：

```bash
toolbox enter
⬢[user@toolbox ~]$ sudo dnf install emacs
⬢[user@toolbox ~]$ emacs &
```

### 使用 Containerfile 建立可再現的開發環境

**重要**：手動在容器內一個一個安裝套件雖然方便，但有個致命缺點——無法再現。當你需要在另一台電腦建立相同環境，或是半年後要重建環境時，你很難記得當初裝了什麼。

更好的做法是撰寫 Containerfile，把所有安裝步驟寫成程式碼。這樣做有以下好處：

- **穩定可再現**：任何人都能用同一份 Containerfile 建立一模一樣的環境
- **版本控管**：Containerfile 可以放進 Git，記錄環境變更歷史
- **團隊協作**：整個團隊使用相同的開發環境，減少「在我電腦上可以跑」的問題
- **快速復原**：環境壞了就刪掉重建，幾分鐘內完全復原

我自己維護了一組 Toolbx Containerfiles，涵蓋常見的開發環境配置：

👉 **推薦使用我的 Toolbx**：<https://github.com/jim60105/toolbx>

裡面包含了 .NET、Java、Node.js 等開發工具，以及 VSCode、JetBrains IDE 等完整配置。你可以直接使用預建的映像檔，或是參考 GitHub 上的 Containerfile 來撰寫自己的配置。

基本用法（以 base toolbox 為例）：

```bash
# 建立 base toolbox（包含常用開發工具）
toolbox create -i quay.io/jim60105/toolbx:latest fedora-toolbox-43

# 複製 runner scripts 到主系統
toolbox run sh -c 'cp /copy-to-host/* ~/.local/bin/'

# 進入容器
toolbox enter fedora-toolbox-43
```

📖 更多資訊：<https://containertoolbx.org/>

---

## rpm-ostree：系統層級套件管理

這是與傳統 dnf 最不同的地方。在 Kinoite 上，系統根目錄是唯讀的，你不能直接用 dnf 安裝套件。取而代之的是 rpm-ostree，它會將套件「疊加」(layer) 到系統映像上。

**重要**：儘量少用這個功能，優先使用 Flatpak 和 Toolbx。

```bash
# 檢查系統狀態
rpm-ostree status

# 安裝套件（需要重開機才生效）
rpm-ostree install fish zsh

# 移除疊加的套件
rpm-ostree uninstall fish

# 搜尋套件
rpm-ostree search docker

# 更新系統
rpm-ostree upgrade

# 檢查可用更新
rpm-ostree upgrade --check

# 查看系統安裝的套件
rpm-ostree status

# 固定一個穩定的版本，以防它被洗掉
ostree admin pin <序號>
```

### Rollback：回復到前一版本

如果更新後系統出問題，可以輕鬆回復：

```bash
# 暫時回復：重新開機後，在 GRUB 選單選擇前一個版本

# 永久回復
rpm-ostree rollback
```

📖 更多資訊：<https://docs.fedoraproject.org/en-US/fedora-kinoite/updates-upgrades-rollbacks/>

---

## 常見問題

### VS Code 怎麼安裝？

**強烈建議使用 Toolbx 而非 Flatpak**。原因是：

開發工作需要大量依賴：編譯器、程式語言執行環境（Python、Node.js、Go 等）、除錯工具、版本控制、資料庫客戶端⋯⋯這些依賴：

1. **需要高度客製化** - 每個專案可能需要不同版本的 Node.js 或 Python
2. **需要記錄安裝過程** - 半年後你會忘記當初裝了什麼
3. **Flatpak 的沙盒限制** - 會讓 VSCode 難以存取其它的工具

我的建議是：把 VSCode 裝在 Toolbx 容器內，並用 Containerfile 記錄整個環境。你可以直接使用我的配置：

```bash
# 建立 VSCode toolbox（已包含 VSCode 和常用開發工具）
toolbox create -i quay.io/jim60105/toolbx-vscode:latest vscode

# 複製桌面圖示和 runner scripts 到主系統
toolbox run -c vscode cp /usr/share/icons/vscode.png ~/.local/share/icons/
toolbox run -c vscode cp /usr/share/icons/vscode-insiders.png ~/.local/share/icons/
toolbox run -c vscode cp /usr/share/applications/code.desktop ~/.local/share/applications/
toolbox run -c vscode cp /usr/share/applications/code-url-handler.desktop ~/.local/share/applications/
toolbox run -c vscode sh -c 'cp /copy-to-host/* ~/.local/bin/'

# 之後可以直接從應用程式選單啟動，或用指令啟動
code
# 或進入容器
toolbox enter vscode
```

**重要設定**：VSCode 需要設定 OS keyring 來同步設定，請參考[官方文件](https://code.visualstudio.com/docs/editor/settings-sync#_recommended-configure-the-keyring-to-use-with-vs-code)配置 `gnome-libsecret`。

這樣做的好處是，整個開發環境（包括 VSCode、擴充套件、程式語言）都被 Containerfile 版本控管，任何人都能一鍵重建相同環境。我的 Toolbx 映像檔已經預先建置並發布到 quay.io，你可以直接拉取使用，不需要自己編譯。

### Docker 怎麼用？

Kinoite 預裝 Podman，它與 Docker 完全相容且不需要 daemon：

```bash
podman pull nginx
podman run -d -p 8080:80 nginx
```

如果真的需要 Docker 請照這個方式安裝： <https://discussion.fedoraproject.org/t/installing-docker-on-silverblue/119610/5>

請特別注意，{{ cr(body="我們公司只能安裝 Docker CE，不能安裝 Docker Desktop!") }}

> Docker Desktop 作為 Docker Personal 訂閱的一部分，對個人、非商業開源開發者、學生和教育工作者，以及員工少於 250 人且年收入低於 1,000 萬美元的小型企業免費提供。若公司有超過 250 名員工或年收入超過 1,000 萬美元，則需付費訂閱（Pro、Team 或 Business）才能商業使用 Docker Desktop。

### 系統升級到新版本（例如 Fedora 42 → 43）

```bash
rpm-ostree rebase fedora:fedora/43/x86_64/kinoite
```

### "Kinoite" 怎麼唸？

{{ youtube(id="xlsnb3snVZc", start=67) }}

---

## 快速對照表

| 你想做的事 | 傳統 Linux | Fedora Kinoite |
|-----------|-----------|----------------|
| 安裝桌面應用程式 | apt/dnf install | flatpak install |
| 安裝開發工具 | apt/dnf install | toolbox enter → dnf install |
| 安裝驅動程式 | apt/dnf install | rpm-ostree install |
| 更新系統 | apt/dnf upgrade | rpm-ostree upgrade |
| 修復壞掉的系統 | 重灌 | rpm-ostree rollback |

---

## 延伸閱讀

- Fedora Kinoite 官方文件：<https://docs.fedoraproject.org/en-US/fedora-kinoite/>
- Toolbx 專案網站：<https://containertoolbx.org/>
- Flatpak 文件：<https://docs.flatpak.org/>
- Flathub 應用程式商店：<https://flathub.org/>

有問題歡迎詢問，祝開發愉快！🚀
