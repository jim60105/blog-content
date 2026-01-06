+++
title = "使用 Fedora CoreOS 在雲端部署 Misskey 聯邦宇宙節點"
description = "用 Fedora CoreOS 和 Ignition 在 DigitalOcean 部署 Misskey，一份 YAML 定義整台伺服器。涵蓋 Rootless Podman、Traefik SSL、S3 備份還原，實現災難復原一鍵部署。"
updated = "2026-01-06T06:23:25.269Z"
date = "2026-01-05T10:29:11.875Z"

[taxonomies]
licenses = [ "GFDL 1.3" ]
tags = [
  "Cloudflare",
  "Container",
  "Digital Ocean",
  "DNS",
  "Misskey",
  "RHEL/Fedora",
  "S3",
  "System Admin"
]

[extra]
featured = true
withAI = "使用 GitHub Copilot 搭配 Claude Opus 4.5 寫作"
card = "preview.png"

  [extra.preview]
  withAI = true
  description = "Made with Nano Banana Pro by Gemini 3"
  url = "https://gemini.google.com/share/276fce756a05"

  [extra.comments]
  id = "ah4uiy1zgwnf00ef"
+++

這篇文章要分享的是我架設 [Misskey](https://misskey-hub.net/tw/) 伺服器的經驗。我選擇了一條跟大多數人不太一樣的路：用 [Fedora CoreOS](https://docs.fedoraproject.org/en-US/fedora-coreos/) 搭配 [Ignition](https://coreos.github.io/ignition/) 設定檔來部署，而不是常見的 Docker Compose 加上 Ubuntu。

為什麼要這麼做？那是因為我想實現一個理想：{{ cg(body="透過設定檔一鍵重建整台伺服器") }}。

<!-- more -->

## 從搬家說起

2022 年，我加入了 [liker.social](https://liker.social) 這個 Mastodon 伺服器，開始了我的聯邦宇宙之旅。這幾年來，我在上面認識了不少有趣的人，也累積了一些追隨者。然而，liker.social 將在 2026 年 1 月 26 日關站。

我很感謝 liker.social 陪伴的這段時光。同時，我也面臨抉擇：**我要搬去哪裡？**

{% chat(speaker="Jim") %}
與其再找一個伺服器寄居，不如自己開一台吧。弄一個屬於自己的小天地。
{% end %}

於是我開始以建置維運的角度研究各種聯邦宇宙的實作。

最知名的 [Mastodon](https://joinmastodon.org/zh-TW)；  
網友們針對單人實例需求推薦的 [GoToSocial](https://gotosocial.org/)；  
研究了極輕量的 [Hollo](https://docs.hollo.social/)；  
但最後我被 [Misskey](https://misskey-hub.net/tw/) 的特色吸引了。

---

## 什麼是 Misskey？

[Misskey](https://misskey-hub.net/tw/) 是一個來自日本的開源去中心化社交平台，實作了 [ActivityPub](https://www.w3.org/TR/activitypub/) 協定，可以和 Mastodon、Pleroma 等其他聯邦宇宙平台互通。

它有幾個讓我特別喜歡的特點：

**{{ cg(body="介面精美") }}**。Misskey 的 Web 介面設計感很強，支援多欄佈局、佈景主題自訂，暗黑模式當然也沒問題。整體視覺體驗比 Mastodon 更活潑。

**{{ cg(body="功能豐富") }}**。除了基本的發文、轉發、追蹤，Misskey 還有頻道、天線（自訂過濾時間軸）、雲端硬碟（管理上傳檔案）、自訂表情符號、MFM（專屬的標記語言，可以讓文字動起來）等功能。

**{{ cg(body="可玩性高") }}**。對於喜歡折騰的人來說，Misskey 有很多可以自訂的地方：角色系統、徽章、小工具、外掛，甚至還有成就系統。這些小設計讓經營伺服器變得有趣。

**{{ cg(body="技術架構現代") }}**。後端用 Node.js，資料庫用 PostgreSQL，搜尋引擎支援 Meilisearch。官方提供了 [Docker Compose 安裝指南](https://misskey-hub.net/tw/docs/for-admin/install/guides/docker/)，部署起來不會太困難。

在讀過完整文件之後，我確信 Misskey 可以配合我設計的這一套 Infrastructure as Code 流程。得益於聯邦宇宙的可遷移設計，我可以把追隨者從舊家無痛轉移過來。

{% chat(speaker="user") %}
所以你的 Misskey 伺服器在哪裡？
{% end %}

{% chat(speaker="Jim") %}
[友.tw](https://xn--ior.tw) ！

沒錯，又是我很喜歡的一字中文網域。這是我的私人伺服器，不開放註冊，但歡迎來追蹤我：<https://友.tw/@Jim>
{% end %}

---

## 從一個痛點說起

{% chat(speaker="user") %}
用 Docker Compose 部署不是很簡單嗎？官方都有[現成的範例檔案](https://github.com/misskey-dev/misskey/blob/develop/compose_example.yml)，為什麼要搞這麼複雜？
{% end %}

{% chat(speaker="Jim") %}
確實，[照著這份官方文件](https://misskey-hub.net/tw/docs/for-admin/install/guides/docker/) 部署 Misskey 非常直覺。Compose file 已經是很不錯的設定檔管理方式，但我想再更進一步。
{% end %}

主機掛了要重建、想在另一台機器測試，才發現就連怎麼設定 OS、設定這些容器、怎麼建置備份還原系統都是一堆手動步驟。

文件可以寫，但{{ cr(body="文件會過時") }}，而且{{ cr(body="人會偷懶 😅") }}。我想要的是一種方式，讓 **設定檔本身就是文件**，而且**整台伺服器就是照著它跑起來的。**

這就是「Infrastructure as Code」的核心理念。而 [Fedora CoreOS](https://docs.fedoraproject.org/en-US/fedora-coreos/) 的設計剛好完美契合這個需求。

---

## 為什麼選擇 Fedora CoreOS？

[Fedora CoreOS](https://docs.fedoraproject.org/en-US/fedora-coreos/) (FCOS) 是 為容器工作負載設計的精簡原子化作業系統。它有幾個特性讓我非常喜歡：

第一，它的 **根檔案系統是唯讀的**。這聽起來很麻煩，但其實是個優點：{{ cg(body="你不能手動 SSH 進去亂改東西，所有變更都必須透過正式的設定流程。") }}這讓系統狀態變得可預測、可重現。

第二，它有 **自動更新機制**。系統會在背景下載安全更新，重新開機後切換到新版本。{{ cg(body="如果更新出問題，還能自動回滾。") }}身為一個不想天天盯著伺服器的人，這功能太棒了。

第三，它是 **原子化系統**。系統更新是以整個映像檔的方式進行，而不是單一套件。{{ cg(body="你用的系統和其它使用者一致。") }}這讓系統更可靠且不容易出錯。

最後也是最關鍵的，它使用 **[Ignition](https://coreos.github.io/ignition/)** 做初始化。

{% chat(speaker="user") %}
Ignition 是什麼？跟 cloud-init 有什麼不同？
{% end %}

{% chat(speaker="Jim") %}
[Ignition](https://coreos.github.io/ignition/) 是 FCOS 在 **第一次開機時** 執行的初始化程式。你提供一份 JSON 格式的設定檔，它會幫你完成磁碟分割、建立檔案系統、新增使用者、寫入設定檔、啟用 systemd 服務——所有你能想到的初始化工作。

跟 cloud-init 較大的不同是，Ignition 在非常早期的 initramfs 階段就執行，而且一直包辦到系統啟動完成。Ignition 更偏宣告式。
{% end %}

> 既 Fedora Kinoite 之後我又多了一個沒人在用的系統 😂  
> 附上這張同事傳給我的圖  
> ![Oh, Ubuntu, you are my favorite Linux-based operating system.](./Scene%20from%20Big%20Bang%20Theory.png#no-hover)

## Butane：讓設定檔變得人類可讀

Ignition 設定檔是 JSON 格式，機器讀得很快，但{{ cr(body="人類寫起來會抓狂") }}。官方提供了 **Butane** 這個工具，讓我們可以用 YAML 格式寫設定，再轉譯成 JSON。

```bash
podman run --interactive --rm --security-opt label=disable \
  --volume "${PWD}":/pwd --workdir /pwd \
  quay.io/coreos/butane:release --pretty --strict \
  misskey-fcos.bu > misskey-fcos.ign
```

`.bu` 檔案是人類寫的 YAML，`.ign` 檔案是給 FCOS 讀的 JSON。轉譯過程中 Butane 還會幫你檢查語法錯誤，算是多一層保障。

有了這個理解，接下來我要帶你看我怎麼設計這份設定檔。

---

## 第一步：選擇主機

<aside><a href="https://www.digitalocean.com/?refcode=ac52b09a7f63&amp;utm_campaign=Referral_Invite&amp;utm_medium=Referral_Program&amp;utm_source=badge">{{ image(url="https://web-platforms.sfo2.cdn.digitaloceanspaces.com/WWW/Badge%201.svg", alt="DigitalOcean Referral Badge") }}</a></aside>

我選擇 <a href="https://m.do.co/c/ac52b09a7f63" target="_blank"><b><span style="-webkit-text-stroke: 0.05em rgb(0,105,255); color: white;">DigitalOcean</span></b></a> 來託管。它是老牌的雲端主機商，口碑不錯，亞洲區有新加坡機房，對台灣連線速度佳。透過上面的推廣連結註冊你可以拿到 **60 天內 200 美元試用額度**，足夠把各種功能玩透。DigitalOcean 的註冊優惠都是這個價，用我的推廣就當成是對我小額贊助吧！😉

規格方面，我建議選擇 CP 值最高的 **每月 6 美元的方案** (1GB RAM / 1 vCPU)。{{ cr(body="4 美元的方案記憶體只有 512MB，無法執行 Misskey。") }}

{% chat(speaker="user") %}
1GB 記憶體夠用嗎？Misskey 加上 PostgreSQL 和 Redis，感覺會很緊繃。
{% end %}

{% chat(speaker="Jim") %}
確實緊繃。但有個省錢的小技巧：**用 Volume 做 Swap**。
{% end %}

DigitalOcean 的 Volume 是每 GB 每月 0.10 美元。我掛一個 8GB 的 Volume 當 Swap，{{ cg(body="每月只要 0.80 美元") }}。相較之下，升級到 2GB RAM 的方案要 {{ cr(body="每月 12 美元") }}，價差超過 10 倍。

在 Butane 設定檔中，我讓系統開機時自動把這顆 Volume 格式化成 swap 並掛載：

```yaml
storage:
  disks:
    - device: /dev/disk/by-path/pci-0000:00:05.0-scsi-0:0:0:1
      wipe_table: true
      partitions:
        - number: 1
          label: swap
          size_mib: 0  # 整顆磁碟都給 swap
  filesystems:
    - device: /dev/disk/by-partlabel/swap
      format: swap
      wipe_filesystem: true
      with_mount_unit: true
```

`with_mount_unit: true` 會讓 Butane 自動產生 systemd mount unit，開機就會掛載。

---

## 第二步：安全性設計

部署在公網上的伺服器，安全性是第一優先。我的設定有幾個重點：

**SSH 只允許金鑰認證**。設定檔裡會寫入我的公鑰，並且停用密碼登入：

```yaml
passwd:
  users:
    - name: core
      ssh_authorized_keys:
        - "ssh-ed25519 AAAA... your-key-here"
      groups: [wheel, sudo]

storage:
  files:
    - path: /etc/ssh/sshd_config.d/20-disable-passwords.conf
      contents:
        inline: |
          PasswordAuthentication no
```

**SSH Port 改成非標準的 22222**。這不是真正的安全措施，但能大幅減少自動化掃描和暴力破解嘗試：

```yaml
- path: /etc/ssh/sshd_config.d/10-custom-port.conf
  contents:
    inline: |
      Port 22222
```

{% chat(speaker="user") %}
改 SSH Port 不是 security through obscurity 嗎？這真的有用？
{% end %}

{% chat(speaker="Jim") %}
嚴格來說是的。但實務上，把 Port 從 22 改掉之後，log 裡的垃圾訊息會少掉大半。這不能取代真正的安全措施（如金鑰認證、防火牆），但我總是會這麼做。
{% end %}

另外，因為 FCOS 預設啟用 SELinux，改了 SSH Port 之後要告訴 SELinux 這是合法的。設定檔裡有一個 systemd service 會處理這件事：

```yaml
- name: selinux-ssh-port.service
  enabled: true
  contents: |
    [Service]
    Type=oneshot
    ExecStart=/usr/sbin/semanage port -a -t ssh_port_t -p tcp 22222
```

當然也不要忘了，在 DigitalOcean 控制台的防火牆設定裡開放這個 Port。我在系統進入穩定，不需要常常連線時也會手動關閉這條防火牆規則，下次要連線時再打開。

---

## 第三步：容器架構設計

Misskey 需要以下幾個服務一起運作：

- **Misskey** 本身（Node.js 應用程式）
- **PostgreSQL** 資料庫
- **Redis** 快取
- **Meilisearch** 全文搜尋（可選但推薦）
- **Traefik** 反向代理（處理 SSL 和路由）

傳統做法是用 Docker Compose 把這些服務編排起來。但在 FCOS 上，我選擇用 **Podman Quadlet**。

{% chat(speaker="user") %}
Podman 我知道，但 Quadlet 是什麼？
{% end %}

{% chat(speaker="Jim") %}
Quadlet 是 Podman 的一個功能，讓你可以 **用類似 systemd unit 檔案的格式定義容器**。把 `.container` 檔案放在特定目錄下，systemd 就會自動管理容器的啟動、停止、重啟。
{% end %}

這帶來幾個好處：{{ cg(body="容器變成 systemd 服務") }}，可以用 `systemctl` 指令管理；{{ cg(body="可以設定服務之間的相依性") }}（比如 Misskey 要等 PostgreSQL 起來才啟動）；{{ cg(body="系統重開機後服務會自動恢復") }}。

更重要的是，所有容器都以 {{ cg(body="rootless 模式") }} 執行。它們跑在普通使用者 `core` 的權限下，{{ cg(body="大幅降低被攻破後的風險") }}。

Quadlet 檔案放在 `/var/home/core/.config/containers/systemd/` 目錄下。以 PostgreSQL 為例：

```ini
[Container]
ContainerName=db
Image=docker.io/postgres:18-alpine
Network=misskey.network
EnvironmentFile=/var/misskey/config/docker.env
Volume=/var/misskey/db:/var/lib/postgresql:Z
HealthCmd=pg_isready -U misskey -d misskey
HealthInterval=5s
HealthRetries=20
```

`:Z` 是 SELinux 的 context 設定，讓 Podman 自動處理權限問題。`HealthCmd` 則是健康檢查，systemd 會根據這個判斷服務是否正常。

所有容器共用一個內部網路 `misskey.network`，讓它們可以透過容器名稱互相溝通。Misskey 的設定檔裡寫 `host: db` 就能連到 PostgreSQL，不需要知道實際 IP，而 db 也不會暴露在外網。

---

## 第四步：SSL 憑證自動化

網站要上 HTTPS，需要 SSL 憑證。我用 [Traefik](https://traefik.io/traefik) 當反向代理，它能自動向 Let's Encrypt 申請憑證。

{% chat(speaker="user") %}
等等，Traefik 要監聽 443 port，但你使用的 rootless 容器預設不能綁 1024 以下的 Port？
{% end %}

{% chat(speaker="Jim") %}
好眼力！確實需要處理這個問題。我們也要修改系統設定，允許非特權使用者綁定低 Port：
{% end %}

```yaml
- path: /etc/sysctl.d/99-unprivileged-ports.conf
  contents:
    inline: |
      net.ipv4.ip_unprivileged_port_start=0
```

這樣 Traefik 就能以 rootless 模式監聽 443 port 了。

至於 SSL 憑證的申請，我使用 [DNS Challenge](https://doc.traefik.io/traefik/reference/install-configuration/tls/certificate-resolvers/acme/#dnschallenge) 而不是 HTTP Challenge。DNS Challenge 的原理是在你的 DNS 記錄裡加一條 TXT 記錄來證明你擁有這個網域，{{ cg(body="不需要開 Port 80") }}，而且{{ cg(body="支援萬用字元憑證") }}。

我的網域放在 Cloudflare，Traefik 內建支援 Cloudflare 的 API，設定起來很簡單：

```yaml
certificatesResolvers:
  cloudflare:
    acme:
      email: your-email@example.com
      storage: /acme/acme.json
      dnsChallenge:
        provider: cloudflare
```

只需要提供 Cloudflare 的 API Token（有 Zone:Read 和 DNS:Edit 權限），Traefik 就會自動處理憑證申請和續約。

---

## 第五步：備份與災難復原

這是整個設計中我最得意的部分 😤

我用 [Restic](https://restic.net/) 把 PostgreSQL 資料庫備份到 [Cloudflare R2](https://www.cloudflare.com/zh-tw/developer-platform/products/r2/)（S3 相容的物件儲存）。R2 提供 {{ cg(body="每月 10GB 免費空間") }}，小型伺服器綽綽有餘。

備份腳本的設計按照最佳實踐使用 `pg_dumpall` 串流輸出，並直接 pipe 給 Restic。

```bash
podman exec db pg_dumpall -U misskey | \
  podman run --rm -i \
    --env-file /var/misskey/backup/restic.env \
    docker.io/restic/restic:latest \
    backup --stdin --stdin-filename "misskey-db.sql"
```

systemd timer 會每天自動執行，並且套用保留策略，舊的備份會自動清除。

{% chat(speaker="user") %}
備份有了，但要怎麼還原？
{% end %}

{% chat(speaker="Jim") %}
這就是精華所在！👀✨

我在設定檔裡寫了一個 `restic-init.service`，它會在 **初次開機時** 檢查 S3 上有沒有既有的備份。如果有，它會 {{ cg(body="自動還原最新的備份") }}。
{% end %}

資料庫還原後還有個問題：{{ cr(body="Meilisearch 的搜尋索引不會自動重建") }}。這代表還原前發布的文章會消失在搜尋結果中。

為了解決這個問題，我寫了 `rebuild-meilisearch-index.sh` 腳本。它會在資料庫還原後、Misskey 啟動前執行一次，{{ cg(body="自動從 PostgreSQL 提取所有文章並重建 Meilisearch 索引") }}。

這代表什麼？代表 **災難復原變成一鍵部署**。

假設哪天主機掛了，我只需要：

1. 直接砍了舊主機。沒錯，刪除 VM！
2. 用同一份 Ignition 設定檔開一台新的 VM
3. 把 DNS 指向新 IP
4. 完成

系統初始化時會自動從 S3 拉取最新備份並還原，幾分鐘後服務就上線了。

VM 再也不是寵物，而是牲畜。

<figure>
{{ image(url="./pets vs cattle.png", alt="Pets vs Cattle", no_hover=true) }}
<figcaption>Made with Nano Banana Pro by Gemini 3</figcaption>
</figure>

---

## 需要準備什麼？

整理一下，在使用這份設定檔之前，你需要先準備：

**DigitalOcean 相關**：帳號、SSH 金鑰、上傳 FCOS 映像檔（從 [Fedora CoreOS 下載頁面](https://fedoraproject.org/coreos/download/?stream=stable#cloud_images) 取得 DigitalOcean 版本，參考 [官方 DigitalOcean 部署文件](https://docs.fedoraproject.org/en-US/fedora-coreos/provisioning-digitalocean/)）

**Cloudflare 相關**：帳號、網域、API Token（需要 Zone:Read 和 DNS:Edit 權限）

**Cloudflare R2**：建立兩個 bucket，一個給備份用，一個給 Misskey 的媒體檔案用

**各種密碼和金鑰**（以 `openssl rand -base64 32` 產生）：PostgreSQL 密碼、Meilisearch Master Key、Restic 備份密碼

---

## 部署流程

1. [下載我的 Butane 模板](https://gist.github.com/jim60105/7753efd7379726b2851b4928e9c87773)，填入所有 `<PLACEHOLDER>` 標記的值
2. 用 Butane 轉譯成 Ignition 設定檔

   ```bash
   podman run --interactive --rm --security-opt label=disable \
      --volume "${PWD}":/pwd --workdir /pwd \
      quay.io/coreos/butane:release --pretty --strict \
      misskey-fcos-template.bu > misskey-fcos.ign
   ```

3. 在 DigitalOcean 建立 Droplet，選擇你上傳的 FCOS 映像檔，新增 Volume，在 User Data 貼上 Ignition 設定檔

   <details><summary>DigitalOcean Droplet 建立參考截圖 (它超長，點我展開)</summary>

   [![DigitalOcean Droplet 建立參考截圖](droplet.png#full-bleed)](droplet.png)

   </details>

4. 等待五分鐘 ☕ （系統會重開機幾次來安裝套件）
5. SSH 連入檢查狀態，記得要把 DigitalOcean 的防火牆規則設定打通！

   ```bash
   ssh -p 22222 core@your-server-ip
   systemctl --user status misskey-web
   ```

---

## 這種做法適合誰？

老實說，用 Fedora CoreOS 部署 Misskey 的難度比一般方式高了不少。你需要先搞懂 Ignition、Butane、Podman Quadlet 這些工具，還要習慣「不要 SSH 進去手動改東改西」的維護理念。

但如果你跟我一樣{{ ch(body="是個怪咖") }}，受夠了「這台伺服器到底改過什麼」的困擾，想要一份能完整描述系統狀態的設定檔，那這種玩法包你滿意。

{{ cg(body="一份 YAML 檔案定義整台伺服器") }}。想重建？轉譯、部署、等五分鐘。資料庫自動從雲端還原，SSL 憑證自動申請，服務自動啟動。

這就是我追求的 Infrastructure as Code。

![Nice meme](./nice.gif#no-hover)

---

## 延伸閱讀

- [Fedora CoreOS 官方文件](https://docs.fedoraproject.org/en-US/fedora-coreos/)
- [Ignition 官方文件](https://coreos.github.io/ignition/)
- [Misskey Hub 繁體中文](https://misskey-hub.net/tw/)
- [Misskey Docker 安裝指南](https://misskey-hub.net/tw/docs/for-admin/install/guides/docker/)
- [Misskey Docker Compose 範例檔案](https://github.com/misskey-dev/misskey/blob/develop/compose_example.yml)
- [在 DigitalOcean 部署 Fedora CoreOS](https://docs.fedoraproject.org/en-US/fedora-coreos/provisioning-digitalocean/)

---

{% chat(speaker="user") %}
為什麼不直接部屬在 AKS/EKS/DOKS?
{% end %}

{% chat(speaker="Jim") %}
沒錢。
{% end %}
