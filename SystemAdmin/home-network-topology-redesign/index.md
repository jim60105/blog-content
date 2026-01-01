+++
title = "自家網路拓撲重新設計：FortiGate 60F 與 Mikrotik hEX S 的角色轉換"
description = "記錄將我家網路從 Mikrotik 前端架構轉換為 FortiGate 60F 前端的過程。包含網路拓撲圖、多 PPPoE 撥接設定、IP 規劃、通訊埠轉發、防火牆政策設計，以及 SSL VPN 配置等。"
date = "2025-12-25T16:44:38.693Z"
updated = "2025-12-25T20:10:03.060Z"

[taxonomies]
tags = [ "FortiGate", "System Admin" ]
licenses = [ "All Rights Reserved" ]

[extra]
withAI = "使用 GitHub Copilot 搭配 Claude Opus 4.5 整理筆記"
card = "preview.jpg"

  [extra.comments]
  id = "agzcp8cmx37i00vj"
+++

## 背景

我家原本使用的 FG-50E UTM 防火牆過期了，於是又買了一台（二手的😝）FG-60F 來替換。趁這個時候把家裡的網路拓撲架構也稍微調整了一下。

<!-- more -->

### 舊架構的設計

<figure>
{{ image(url="RB760iGS.webp", alt="Mikrotik hEX S (RB760iGS)", no_hover=true) }}
<figcaption>Mikrotik hEX S (RB760iGS)</figcaption>
</figure>

以前是將 [Mikrotik hEX S (RB760iGS)](https://mikrotik.com/product/hex_s) 放在最外面，負責 PPPoE 撥接、NAT、以及 VPN 伺服器，然後再把所有流量往後丟給 FG-50E 做防火牆和 UTM 功能。FG-50E 做通透模式，沒有做 NAT。Mikrotik RouterOS 的路由功能非常厲害，但我大致上用不太到那些進階功能。當時實際使用到的是撥接多路 PPPoE、通訊埠轉發、最簡易的防火牆通訊埠阻擋、VPN、切了簡單的 VLAN。

### 新架構的改變

<figure>
{{ image(url="FG-60F.png", alt="FortiGate 60F", no_hover=true) }}
<figcaption>FortiGate 60F</figcaption>
</figure>

這次換了 [FortiGate FG-60F](https://www.fortinet.com/resources/data-sheets/fortigate-fortiwifi-60f-series)，它的效能夠強（ARMv8 8-core CPU、2 GB RAM），讓我決定把 FG-60F 放在最外面負責最危險的工作，同時也直接負責多路 PPPoE 撥接、NAT、VPN。將 Mikrotik hEX S 換到內部網路，透過 Bridge-Lan 將所有內部設備橋接在一起，純粹作為高速交換器使用。

之所以要一台內部交換器而不是把所有設備都接在 FG-60F，是因為我有一台 [SeaweedFS](https://github.com/seaweedfs/seaweedfs) 跑在 NAS 上，而另一台 K8s 設備會經常存取這個 S3 站台。如果所有流量都經過 FG-60F，NAS 和 K8s 之間的流量會經過防火牆，導致防火牆的負載增加。把 Mikrotik 放在內部交換器的位置可以讓 NAS 和 K8s 之間的流量直接在交換器內轉發，就不用經過防火牆了。

### Mikrotik 的角色轉換

在新架構中，Mikrotik hEX S 的設定大幅簡化。改為純 Bridge 模式，將 `ether1`（上行至 FortiGate）、`ether2~5` 全部橋接成 `Bridge-Lan`，並設定靜態 IP 方便管理和監控。所有 bridge port 都啟用 L2 HW Offloading 以提升效能。

Mikrotik RouterOS 的設定介面真的是非常複雜，當年花了我很多時間摸索。現在只用到最基本的橋接功能，設定起來倒是簡單不少。

---

## 網路拓撲結構

```
Internet
   |
中華電信光世代 Modem
   |
FG-60F (192.168.10.1)
  - wan1: 
      PPPoE 主介面 (多路撥號中華 Hinet PPPoE)
      ├── ppp2: 固定 IP (Gateway，僅用於 VPN 連線)
      ├── pppoe_default: 浮動 IP (其它預設出站路由)
      ├── pppoe_web: 浮動 IP (給 Talos K8s)
      └── pppoe_dsm: 浮動 IP (給 NAS #1)
  
  - internal (hardware switch，包含 internal1~5)
      |
      ├── internal1 ──> RT-AC66U B1 (192.168.10.3) - Wi-Fi AP
      └── internal2 ──> Mikrotik hEX S (192.168.10.2)
                          └── Bridge-Lan（橋接 ether1~5 + sfp1）
                              |-- ether1: 連接 FG-60F internal2（上行）
                              |-- ether2: DS218J NAS #2 (192.168.10.40) - 內部備份
                              |-- ether3: Linux Workstation (192.168.10.10) - 個人工作站
                              |-- ether4: DS920+ NAS #1 (192.168.10.30) - 對外服務
                              |-- ether5: Talos K8s node (192.168.10.20)
                              └── sfp1: 未使用（停用）
  
  - internal3~5: 未使用（Down）
```

Modem 之下其實還有一台中華電信的 Wi-Fi 6 全屋通 4T4R，這台是給樓下我家人專用，和我自己的網路是完全獨立。不得不說這台機器的硬體規格是真的猛，比我的 AC66U 還好。

---

## IP 規劃

### 公網 IP（中華電信 Hinet PPPoE）

| 介面 | 用途 |
| --- | --- |
| `ppp2` | 主要 Gateway |
| `pppoe_default` | 預設路由（一般上網） |
| `pppoe_web` | Talos K8s 對外服務 |
| `pppoe_dsm` | DSM NAS 對外服務 |

我家是申租中華電信 Hinet 光世代雙向 500M (7 浮 1 固)，我在這裡用了 4 個公網 IP。FG-60F 透過多 PPPoE 撥接取得這些 IP，並使用 Policy-Based Routing 及 Virtual IP 決定不同內部設備的流量走向。

### 內部 IPv4 網段

網段為 `192.168.10.0/24`，子網遮罩 `255.255.255.0`，預設閘道 `192.168.10.1`（FG-60F `internal`），DNS 伺服器 `192.168.10.1`。

### 固定 IP 分配表

| 設備 | IP 位址 | 說明 |
| --- | --- | --- |
| FG-60F `internal` | `192.168.10.1` | Gateway / DNS / DHCP |
| Mikrotik hEX S | `192.168.10.2` | 交換器 |
| RT-AC66U B1 | `192.168.10.3` | Wi-Fi AP |
| Linux Workstation | `192.168.10.10` | 開發工作站 |
| Talos K8s node | `192.168.10.20` | Kubernetes 節點 |
| DS920+ NAS #1 | `192.168.10.30` | 對外服務 + 反向代理 |
| DS218J NAS #2 | `192.168.10.40` | 內部備份 |
| DHCP 範圍 | `192.168.10.100~199` | Wi-Fi 裝置動態分配 |

### VPN 網段

SSL VPN Pool 為 `10.255.0.0/24`。VPN 客戶端連線後會從此網段獲得虛擬 IP，DNS 指向 `192.168.10.1`。

---

## 多 PPPoE 撥接策略

<figure>
{{ image(url="WAN_Interface.png", alt="FortiGate 60F WAN Interface", no_hover=true) }}
<figcaption>FortiGate 60F WAN Interface</figcaption>
</figure>

使用「單一實體介面多條 PPPoE」的方式從中華電信取得 4 個公網 IP。在 FortiGate 上只要使用 `PPPoE Interface` 就能多路撥號。這種做法可以達到服務隔離，不管是外對內還是內對外的流量，兩台設備都有固定的對應對外 IP。

以前我是使用單一公網 IP 接受所有的 HTTPS `443` 連線，再在 DS920+ 上設定 `Reverse Proxy` 導向到內部不同主機。這使得我的 K8s 設備連線也都要經過 NAS，增加不必要的延遲和單點故障風險。現在兩台設備有自己的公網 IP，不需要 `Reverse Proxy` 也不會打架了。

> HiNet 非固定制固定 IP 申請網址  
> <https://123.cht.com.tw/HiNetStaticIp>

值得一提的是，使用 `[HN 號碼]@ip.hinet.net` 作為 PPPoE 帳號會拿到固定 IP；使用 `[HN 號碼]@hinet.net` 則是浮動 IP。浮動 IP 可以同時多線撥號。

### 流量路由策略（Policy-Based Routing）

<figure>
{{ image(url="Policy_Routes.png", alt="FortiGate 60F Policy Routes", no_hover=true) }}
<figcaption>FortiGate 60F Policy Routes</figcaption>
</figure>

FortiGate 使用 `Policy-Based Routing` 來決定流量走向（內部 IP → 公網 IP）。DS920+ (`192.168.10.30`) 強制走 `pppoe_dsm`，Talos K8s (`192.168.10.20`) 強制走 `pppoe_web`，其他設備都走 `pppoe_default`，並開啟完整 UTM 防護。

### 通訊埠轉發（Virtual IP）

<figure>
{{ image(url="VIP.png", alt="FortiGate 60F Virtual IP", no_hover=true) }}
<figcaption>FortiGate 60F Virtual IP</figcaption>
</figure>

使用 FortiGate 的 `VIP` 功能將特定公網 IP 的連接埠轉發到內部伺服器。

| 名稱 | 外部介面 | 外部通訊埠 | 內部 IP | 內部通訊埠 | 用途 |
| --- | --- | --- | --- | --- | --- |
| `vip_talos_443_web` | `pppoe_web` | `443` | `192.168.10.20` | `443` | K8s Ingress (HTTPS) |
| `vip_dsm_443_dsm` | `pppoe_dsm` | `443` | `192.168.10.30` | `443` | DSM 上運行的網路服務 (HTTPS) |
| `vip_dsm_80_dsm` | `pppoe_dsm` | `80` | `192.168.10.30` | `80` | Let's Encrypt 憑證驗證 |
| `vip_dsm_51413_dsm` | `pppoe_dsm` | `51413` | `192.168.10.30` | `51413` | Transmission BT |

---

## 防火牆政策設計

### 內部網路政策

`lan2lan` 政策允許內網設備之間互通（`internal` ↔ `internal`）。`vpn2lan` 政策讓 VPN 使用者可存取內網服務（僅開放特定通訊埠和協定）。

實際上 `internal` 介面是包含 Mikrotik hEX S 的 `Bridge-Lan`，所以內部設備之間的流量會直接在交換器內轉發，不會經過 FG-60F 的 `lan2lan` 政策。

### 對外連線政策

`LAN_to_WAN_pppoe_default` 處理來自 `internal`（除了 DS920+ 和 Talos）到 `pppoe_default` 的流量，啟用完整 UTM（AV、Web Filter、DNS Filter、IPS、File Filter）保護我的日常使用，並啟用 NAT。

`DSM_to_WAN_pppoe_dsm` 處理來自 `192.168.10.30`（DS920+）到 `pppoe_dsm` 的流量，啟用 DNS Filter，啟用 NAT。

`Talos_to_WAN_pppoe_web` 處理來自 `192.168.10.20`（Talos）到 `pppoe_web` 的流量，啟用 DNS Filter，啟用 NAT。

比較特別的是，我設定了一條 Youtube 專用的政策，讓我看影片的流量不用經過 UTM 檢查，減少延遲和卡頓。在另一篇文章有詳細說明: [通過 FortiGate 看 Youtube 很 LAG？ - 琳的備忘手札](@/Livestream/fortigate-passthrough-youtube/index.md)

### 對內轉發政策

`wan2http` 處理來自 `pppoe_dsm`、`pppoe_web` 到 `VIP` 對象的流量，啟用 IPS（入侵防禦），不啟用 SSL 深度檢測。

### VPN 政策

`vpn2lan` 處理來自 `SSL_VPN_POOL` 到內網的流量，允許服務包括 SSH、RDP、HTTP/HTTPS、Samba、DNS、ICMP，啟用 AV 和 IPS。

`vpn2wan` 處理來自 `SSL_VPN_POOL` 到 `wan1` 的流量，允許所有服務，啟用 NAT，讓 VPN 使用者可透過家裡網路上網。

---

## SSL VPN 伺服器設定

### 設定參數

監聽介面為 `wan1`（實際上透過 `ppp2` 的固定 IP）。IP 位址池為 `SSL_VPN_POOL`（`10.255.0.0/24`），DNS 伺服器為 `192.168.10.1`。

### Portal 設定

啟用 `Tunnel Mode`（支援完整網路存取）、啟用 `Web Mode`（支援瀏覽器 Portal）、啟用 `Auto Connect`（自動連線）、停用 `Split Tunneling`（所有流量都走 VPN）。

### 功能說明

從外部連線到 VPN 時，可以存取所有內網設備（`192.168.10.0/24`）、存取 K8s 服務、NAS 管理介面、Workstation，以及使用家裡的網路上網（透過 `wan1` 出去）。這樣我就可以不將 DSM 或其他管理介面直接曝露到網際網路，而是先連接 VPN 進來再存取。

---

## DNS 伺服器設定

<figure>
{{ image(url="DNS.png", alt="FortiGate 60F DNS Server", no_hover=true) }}
<figcaption>FortiGate 60F DNS Server</figcaption>
</figure>

我在 FG-60F 上啟用 DNS 伺服器，主要是為了設定內部網域解析。我將外部網域名稱對應到內部 IP，這樣由內部存取時就不會走到外頭去繞一圈，能直接在 Mikrotik 交換器內完成轉發。它的上游 DNS 我使用 `1.1.1.1`。

---

## Cloudflare DDNS

<figure>
{{ image(url="simple-cloudflare-ddns.webp", alt="Simple Cloudflare DDNS Updater", no_hover=true) }}
<figcaption>Simple Cloudflare DDNS Updater</figcaption>
</figure>

這邊特別提一下，我之所以能夠使用多個浮動 IP 來做為對外服務而不怕 IP 變動，是因為我實作了 Cloudflare DDNS 功能。我在 NAS 和 K8s 上都跑了自己寫的 Cloudflare DDNS Updater 容器，每小時更新一次 DNS A 紀錄，確保我的網域名稱永遠指向正確的公網 IP。

有類似需求的讀者也可以使用我的小專案： [Simple Cloudflare DDNS Updater](https://github.com/jim60105/simple-cloudflare-ddns)

---

## 結語

這次的網路調整花了我好幾天時間研究 FortiGate 的設定和功能。尤其是 `Policy-Based Routing` 和 `Virtual IP` 的部分，我原本不曉得是用這兩個功能來達成我的需求。幸好 2025 年是 AI 的時代 —— 問 ChatGPT 搞定一切！

我現在都笑稱我家大概比一般中小企業的防護還要強！  
拿 FG-60F 來做為家用是相當兇猛了 😀
