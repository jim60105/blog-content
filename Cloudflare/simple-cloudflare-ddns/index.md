+++
title = "用 Cloudflare DDNS 讓家中伺服器永遠連得上：完整設定指南"
description = "家裡伺服器 IP 老是變？本文帶你從頭理解 DDNS 原理，手把手教你設定 Cloudflare API Token、取得 Zone ID 和 Record ID，並用我的開源專案 simple-cloudflare-ddns 輕鬆實現自動化 DNS 更新。"
date = "2025-12-26T08:55:48.741Z"
draft = true
updated = "2025-12-26T09:19:03.915Z"

[taxonomies]
tags = [ "Cloudflare", "Container", "Linux", "System Admin" ]
licenses = [ "GFDL 1.3" ]

[extra]
withAI = "使用 GitHub Copilot 搭配 Claude Opus 4.5 寫作"
+++

{% chat(speaker="user") %}
@jim 我家裡架了一台伺服器，想用自己的網域連回去。但問題是我家的對外 IP 會變動，每次 IP 一換，DNS 設定就失效了，有沒有辦法自動處理這件事？
{% end %}

{% chat(speaker="jim") %}
這是架設 Home Lab 或自架服務常遇到的問題。你需要的是 DDNS (Dynamic DNS) 解決方案。簡單來說，DDNS 會定期檢查你目前的對外 IP，發現變動時就自動更新 DNS 記錄，讓你的網域始終指向正確的 IP 位址。

既然你的網域已經託管在 Cloudflare，我們就可以利用 Cloudflare API 來實現這件事。我做了一個簡單的開源專案叫 [simple-cloudflare-ddns](https://github.com/jim60105/simple-cloudflare-ddns)，跑在容器裡面，設定好環境變數就能使用了喔！
{% end %}

<!-- more -->

## DDNS 運作原理

在開始設定之前，先來理解 DDNS 的運作流程。整個過程可以拆解成兩個核心步驟：首先是取得伺服器目前的對外 IP 位址，接著是透過 Cloudflare API 將這個 IP 寫入對應的 DNS 記錄。

<pre class="mermaid">
sequenceDiagram
    participant Server as 你的伺服器
    participant IPService as IP 偵測服務
    participant Cloudflare as Cloudflare API
    participant DNS as DNS 記錄

    Server->>IPService: 1. 查詢目前對外 IP
    IPService-->>Server: 返回 203.0.113.42

    Server->>Cloudflare: 2. 更新 DNS A Record
    Note over Server,Cloudflare: Authorization: Bearer API_TOKEN<br/>{"type": "A", "content": "203.0.113.42"}
    Cloudflare->>DNS: 寫入新 IP
    Cloudflare-->>Server: 更新成功

    Note over Server,DNS: 現在 home.example.com 指向 203.0.113.42
</pre>

當你家的 IP 位址改變時（例如路由器重新撥號），只要 DDNS 腳本再次執行，就會自動偵測到新的 IP 並更新 DNS 記錄。透過定時執行這個腳本，你的網域就能始終指向正確的位址。

{% chat(speaker="user") %}
原理聽起來不難，那實際設定要準備什麼？
{% end %}

{% chat(speaker="jim") %}
你需要準備三樣東西：Cloudflare API Token、Zone ID、以及 DNS Record ID。

API Token 用來授權存取 Cloudflare API；  
Zone ID 代表你的網域在 Cloudflare 的識別碼；  
Record ID 則是你想更新的那筆 DNS 記錄的識別碼。

接下來我一步步帶你取得這些資訊。
{% end %}

## 前置需求

在開始之前，請先確認以下條件：

1. 你的網域已經託管在 Cloudflare（DNS 由 Cloudflare 管理）
2. 你已經在 Cloudflare 建立了要用於 DDNS 的 DNS 記錄（A 記錄或 AAAA 記錄）
3. 你有辦法執行容器（Podman 或 Docker）

如果還沒建立 DNS 記錄，先到 Cloudflare Dashboard 的 DNS 頁面新增一筆 A 記錄，IP 可以先填任意值，之後 DDNS 會自動更新。

## 建立 Cloudflare API Token

Cloudflare API 使用 Token 進行身份驗證。為了遵循最小權限原則，我們要建立一個只有 DNS 編輯權限的專用 Token。

{% chat(speaker="user") %}
為什麼不直接用 Global API Key？網路上有些教學是用那個。
{% end %}

{% chat(speaker="jim") %}
Global API Key 擁有你 Cloudflare 帳號的完整存取權限，風險太高了！若這個金鑰外洩，攻擊者可以修改你所有的設定、刪除網域，做任何他想做的事。使用 API Token 可以精確控制權限範圍，這筆 Token 只能編輯 DNS 記錄，就算不小心洩漏，影響範圍也有限。
{% end %}

前往 Cloudflare Dashboard 的 [API Tokens 頁面](https://dash.cloudflare.com/profile/api-tokens)，依照以下步驟建立 Token：

1. 點擊 **Create Token**
2. 選擇 **Custom token** 範本
3. 設定 Token 名稱，例如 `DNS Update Script`
4. 在 Permissions 區塊設定：**Zone** → **DNS** → **Edit**
5. 在 Zone Resources 區塊設定：**Include** → **All zones**（或選擇特定網域）
6. 點擊 **Continue to summary**，確認權限後點擊 **Create Token**
7. 複製產生的 Token 並妥善保存

> [!CAUTION]
> API Token 只會顯示一次，離開頁面後就無法再次查看。請立即複製並存放在安全的地方。

## 取得 Zone ID

Zone ID 是 Cloudflare 用來識別你網域的唯一識別碼。取得方式十分簡單：

1. 登入 [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. 選擇你要設定 DDNS 的網域
3. 在 Overview 頁面的右側欄位，找到 **Zone ID**
4. 複製這串 32 字元的十六進位識別碼

## 取得 DNS Record ID

每筆 DNS 記錄都有一個獨立的 Record ID。這個 ID 無法在 Dashboard 上直接看到，需要透過 API 查詢。開啟終端機或 Postman，執行以下指令：

```bash
curl -X GET "https://api.cloudflare.com/client/v4/zones/YOUR_ZONE_ID/dns_records/" \
     -H "Authorization: Bearer YOUR_API_TOKEN"
```

將 `YOUR_ZONE_ID` 和 `YOUR_API_TOKEN` 替換成剛才取得的值。執行後會列出該網域下所有 DNS 記錄的詳細資訊。找到你要用於 DDNS 的記錄，記下它的 `id` 值。

{% chat(speaker="user") %}
如果我想同時更新 IPv4 和 IPv6 的記錄呢？
{% end %}

{% chat(speaker="jim") %}
那你需要分別記下 A 記錄和 AAAA 記錄的 Record ID。A 記錄對應 IPv4，AAAA 記錄對應 IPv6。simple-cloudflare-ddns 支援同時更新這兩種記錄，只要在環境變數裡分別設定就可以囉！
{% end %}

## 選擇 IP 偵測服務

DDNS 的第一步是取得目前的對外 IP。這需要透過一個外部服務來查詢，因為你的伺服器只能看到內網 IP，無法直接得知 NAT 後面的公開 IP 位址。

### 預設服務

simple-cloudflare-ddns 預設使用 [ipify](https://www.ipify.org/) 服務：

- IPv4：`https://api.ipify.org`
- IPv6：`https://api6.ipify.org`

這是一個運作多年的開源服務，直接返回你的 IP 位址純文字。

### 部署自己的 IP 偵測服務

依賴第三方服務有一定風險：服務可能停機、可能有速率限制，甚至有可能未來被惡意控制，回傳了駭客指定的其它內容。若你追求更高的可靠性，我建議部署自己的 IP 偵測服務。

我做了另一個開源專案 [worker-your-ip](https://github.com/jim60105/worker-your-ip)，這是一個跑在 Cloudflare Workers 上的 IP 偵測服務。除了回傳 IP 位址，還能提供地理位置、時區、ASN 等額外資訊。

[![Deploy to Cloudflare Workers](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/jim60105/worker-your-ip)

點擊上面的按鈕可以一鍵部署到你的 Cloudflare 帳號。部署完成後，你的 IP 偵測服務就會有以下端點：

- `/ip`：返回 IPv4 或 IPv6 位址
- `/ipv4`：返回 IPv4 位址（如果沒有 IPv4 則回傳 400 錯誤）
- `/ipv6`：返回 IPv6 位址（如果沒有 IPv6 則回傳 400 錯誤）
- `/json`：以 JSON 格式返回完整資訊

{% chat(speaker="user") %}
自己部署的好處是什麼？
{% end %}

{% chat(speaker="jim") %}
首先是安全，你不會因為第三方服務遭駭而被影響；  
其次是效能，Cloudflare Workers 跑在全球邊緣網路，回應速度超快；  
第三是成本，Cloudflare Workers 的免費額度足夠個人使用；  
最後是隱私，你的 IP 查詢紀錄不會經過不受控的第三方。
{% end %}

## 執行 DDNS 更新

準備好所有資訊後，就可以執行 simple-cloudflare-ddns 了。這個專案打包成容器映像檔，用 Podman 或 Docker 都能跑。

### 僅更新 IPv4

```bash
podman run --rm \
  -e API_TOKEN=your_cloudflare_api_token \
  -e ZONE_ID=your_zone_id \
  -e A_RECORD_ID=your_a_record_id \
  -e A_RECORD_NAME=home.example.com \
  ghcr.io/jim60105/simple-cloudflare-ddns
```

### 僅更新 IPv6

```bash
podman run --rm \
  -e API_TOKEN=your_cloudflare_api_token \
  -e ZONE_ID=your_zone_id \
  -e AAAA_RECORD_ID=your_aaaa_record_id \
  -e AAAA_RECORD_NAME=home.example.com \
  ghcr.io/jim60105/simple-cloudflare-ddns
```

### 同時更新 IPv4 和 IPv6

```bash
podman run --rm \
  -e API_TOKEN=your_cloudflare_api_token \
  -e ZONE_ID=your_zone_id \
  -e A_RECORD_ID=your_a_record_id \
  -e A_RECORD_NAME=home.example.com \
  -e AAAA_RECORD_ID=your_aaaa_record_id \
  -e AAAA_RECORD_NAME=home.example.com \
  ghcr.io/jim60105/simple-cloudflare-ddns
```

### 使用自訂 IP 偵測服務

```bash
podman run --rm \
  -e API_TOKEN=your_cloudflare_api_token \
  -e ZONE_ID=your_zone_id \
  -e A_RECORD_ID=your_a_record_id \
  -e A_RECORD_NAME=home.example.com \
  -e IPV4_API_URL=https://your-worker.workers.dev/ipv4 \
  -e IPV6_API_URL=https://your-worker.workers.dev/ipv6 \
  ghcr.io/jim60105/simple-cloudflare-ddns
```

## 設定定時執行

手動執行一次只能更新當下的 IP，但 DDNS 的重點在於持續監控。你需要設定排程讓腳本定期執行。

### 使用 Cron

在 Linux 系統上，cron 是最簡單的排程方式：

```bash
# 編輯 crontab
crontab -e

# 新增這行，每 30 分鐘執行一次
*/30 * * * * podman run --rm -e API_TOKEN=... -e ZONE_ID=... -e A_RECORD_ID=... -e A_RECORD_NAME=... ghcr.io/jim60105/simple-cloudflare-ddns
```

### 使用 Kubernetes CronJob

如果你的伺服器跑 Kubernetes，可以用 CronJob 來排程。按照最佳實踐，請將敏感資訊存在 Secret 裡：

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-ddns
type: Opaque
stringData:
  API_TOKEN: "your_cloudflare_api_token"
  ZONE_ID: "your_zone_id"
  A_RECORD_ID: "your_a_record_id"
  A_RECORD_NAME: "home.example.com"
```

```yaml
# cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: simple-cloudflare-ddns
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: ddns
              image: ghcr.io/jim60105/simple-cloudflare-ddns:latest
              imagePullPolicy: IfNotPresent
              envFrom:
                - secretRef:
                    name: cloudflare-ddns
```

執行 `kubectl apply -f secret.yaml` 和 `kubectl apply -f cronjob.yaml` 套用設定。

## 環境變數參考

完整的環境變數說明如下：

| 變數名稱 | 必要性 | 說明 | 預設值 |
| --------- | -------- | ------ | -------- |
| `API_TOKEN` | 必要 | Cloudflare API Token | - |
| `ZONE_ID` | 必要 | Cloudflare Zone ID | - |
| `A_RECORD_ID` | 選填 | IPv4 A 記錄的 Record ID | - |
| `A_RECORD_NAME` | 選填 | A 記錄的網域名稱 | - |
| `AAAA_RECORD_ID` | 選填 | IPv6 AAAA 記錄的 Record ID | - |
| `AAAA_RECORD_NAME` | 選填 | AAAA 記錄的網域名稱 | - |
| `IPV4_API_URL` | 選填 | 取得 IPv4 位址的 API 端點 | `https://api.ipify.org` |
| `IPV6_API_URL` | 選填 | 取得 IPv6 位址的 API 端點 | `https://api6.ipify.org` |

`A_RECORD_ID` 和 `AAAA_RECORD_ID` 至少要設定其中一個，否則腳本沒有目標可更新。

{% chat(speaker="user") %}
設定完之後怎麼確認有沒有正常運作？
{% end %}

{% chat(speaker="jim") %}
手動跑一次容器看輸出訊息，成功的話會顯示目前偵測到的 IP 和更新結果。你也可以到 Cloudflare Dashboard 的 DNS 頁面確認記錄內容是否正確。如果使用 cron，可以檢查系統日誌或將 cron 輸出導向檔案來追蹤執行狀況。
{% end %}

## 結語

DDNS 的原理和實作都不複雜，核心就是「偵測 IP」加上「呼叫 API 更新記錄」這兩件事。simple-cloudflare-ddns 把這些邏輯包裝成一個輕量的容器，只需要設定幾個環境變數就能使用。如果你也在經營 Home Lab 或自架服務，希望這個工具能幫上忙。

專案原始碼在 GitHub 上開源，歡迎提出問題或參與貢獻：

- [jim60105/simple-cloudflare-ddns](https://github.com/jim60105/simple-cloudflare-ddns)
- [jim60105/worker-your-ip](https://github.com/jim60105/worker-your-ip)

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true, theme: 'dark' });
</script>
