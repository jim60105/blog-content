+++
title = "GitHub Coding Agent 搭配 Playwright 的截圖正確顯示網頁字型指南"
description = "當 GitHub Coding Agent 使用 Playwright MCP 截圖時，中文字會變成方塊？這篇文章解析 Playwright 網路限制和防火牆設定的差異，提供兩種解決方案：修改 MCP 設定或預裝本地字型，讓你的 AI 代理正確渲染多語言文字。"
date = "2025-12-03T08:18:08.711Z"
updated = "2025-12-03T08:18:09.103Z"

[taxonomies]
tags = [ "DevOps", "GitHub", "AI" ]
licenses = [ "GFDL 1.3" ]

[extra]
banner = "cover.png"
withAI = "使用 GitHub Copilot 搭配 Claude Opus 4.5 寫作"

  [extra.preview]
  withAI = true
  url = "https://gemini.google.com/share/d08bf91bb1b5"
+++

最近我頻繁使用 GitHub Copilot Coding Agent 進行 Side project 開發工作。在處理前端專案時，我注意到 Playwright 截圖中的中文字全部變成方塊。直覺反應是網路被擋住了，於是我試著先關閉 Coding Agent 的防火牆設定，結果問題依舊存在。

翻閱官方文件後才發現，防火牆和 Playwright MCP Server 的網路限制是兩個獨立的機制。即使防火牆放行，Playwright 預設仍只能存取本機資源。為了驗證這個理解，我建立了 [test-runner-fonts](https://github.com/jim60105/test-runner-fonts) 儲存庫進行系統性測試。這篇文章記錄我的發現，並提供兩種可行的解決方案。

<!-- more -->

## 問題背景：防火牆與 MCP 網路限制

在討論解決方案之前，得先釐清 GitHub Coding Agent 的兩層網路限制機制。

### Coding Agent 防火牆

根據 GitHub 官方文件，Coding Agent 的防火牆預設開啟，用於限制代理存取外部網路的能力。

防火牆有一個重要的限制條件：

> The firewall only applies to processes started by the agent via its Bash tool. It does not apply to Model Context Protocol (MCP) servers or processes started in configured Copilot setup steps.[^1]

防火牆**只適用於**透過 Bash 工具啟動的程式。{% cr() %}MCP 伺服器和 Copilot setup steps 中啟動的程式不受防火牆管控{% end %}。這代表即使關閉防火牆，MCP 伺服器仍可能有自己的網路限制。

### Playwright MCP Server 的預設限制

根據 GitHub 官方文件：

> By default, the Playwright MCP server is only able to access web resources hosted within Copilot's own environment, accessible on `localhost` or `127.0.0.1`.[^2]

{% cr() %}Playwright MCP Server 預設只能存取本機資源，無法存取外部網路{% end %}。這是獨立於防火牆的安全設計。

當網頁使用 Google Fonts 或其他 CDN 提供的網頁字型時，Playwright 無法從外部伺服器下載這些字型檔案。結果就是渲染時找不到對應的字形，中文字變成方塊。

## 測試驗證

為了確認上述理論，我在 [test-runner-fonts](https://github.com/jim60105/test-runner-fonts) 儲存庫中進行了四組測試，驗證不同配置下的字型載入情況。

| 測試 | 防火牆設定 | Playwright 設定 | 字型安裝 | 結果 |
|------|------------|-----------------|----------|------|
| [PR #1](https://github.com/jim60105/test-runner-fonts/pull/1) | 開啟（網路受限） | 預設值 | 無 | ❌ 字體未載入 |
| [PR #2](https://github.com/jim60105/test-runner-fonts/pull/2) | 關閉（網路暢通） | `--allowed-hosts *` | 無 | ✅ 字體成功載入 |
| [PR #3](https://github.com/jim60105/test-runner-fonts/pull/3) | 關閉（網路暢通） | 預設值 | 無 | ❌ 字體未載入 |
| [PR #4](https://github.com/jim60105/test-runner-fonts/pull/4) | 開啟（網路受限） | 預設值 | action-install-google-fonts | ✅ 字體成功載入 |

### 關鍵發現

- **PR #3 的結果最具說明性**：即使防火牆設定為關閉（網路暢通），Playwright 的預設設定仍會阻擋外部資源存取。這證實了 Playwright 的網路限制是獨立運作的機制。

- **PR #2 和 PR #4 則展示了兩種可行的解決路徑**：透過 `--allowed-hosts *` 參數開放 Playwright 的網路存取，或透過預先安裝本地字型來繞過網路需求。

## 解決方案一：修改 Playwright MCP Server 設定

這個方案透過設定 `--allowed-hosts *` 參數，讓 Playwright 可以存取外部網路資源。

### MCP 設定步驟

前往儲存庫設定：Settings → Copilot → Coding agent，在 MCP configuration 區塊加入以下配置：

```json
{
  "mcpServers": {
    "playwright": {
      "type": "local",
      "command": "npx",
      "args": ["@playwright/mcp@latest", "--allowed-hosts", "*"],
      "tools": ["*"]
    }
  }
}
```

根據 Playwright MCP 官方文件[^3]，`--allowed-hosts` 參數接受逗號分隔的主機列表。傳入 `*` 可停用主機檢查，允許存取所有外部資源。

> [!TIP]  
> 我有另一篇 AI 部落格文章專門介紹 [GitHub Copilot Coding Agent Playwright MCP 外部主機存取設定教學](https://xn--uy0a.tw/AI/github-copilot-coding-agent-playwright-mcp-external-hosts-setup/)，歡迎有興趣的讀者參考。

### 方案一適用情境

若專案需要存取多種外部資源，或無法事先確定所需字型名稱，這個方案較為靈活。測試環境或開發階段使用這個設定能減少設定負擔。

### 安全考量

{% cr() %}開放所有外部存取會降低安全隔離程度{% end %}。若專案有嚴格的安全需求，建議評估方案二。

## 解決方案二：使用 action-install-google-fonts 安裝本地字型

> [!NOTE]  
> 我寫的 😉

這個方案透過 Copilot setup steps 預先安裝字型到 runner 本地，讓 Playwright 直接使用系統字型渲染，不需要網路存取。

### Workflow 設定步驟

在儲存庫中建立 `.github/workflows/copilot-setup-steps.yml` 檔案：

```yaml
name: "Copilot Setup Steps"

on:
  workflow_dispatch:
  push:
    paths:
      - .github/workflows/copilot-setup-steps.yml
  pull_request:
    paths:
      - .github/workflows/copilot-setup-steps.yml

jobs:
  copilot-setup-steps:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: Install fonts
        uses: jim60105/action-install-google-fonts@v2
        with:
          fonts: |
            Noto Sans TC
            Noto Color Emoji
```

我為這個需求特地寫了一個 GitHub Action [action-install-google-fonts](https://github.com/jim60105/action-install-google-fonts)，從 Google Fonts 下載並安裝指定字型到系統中。

### 參數說明

| 參數 | 必填 | 預設值 | 說明 |
|------|------|--------|------|
| fonts | 是 | - | 要安裝的字型列表（逗號或換行分隔） |
| weights | 否 | '400,700' | 要安裝的字型粗細（100-900） |

### 常用字型清單

中文字型方面，`Noto Sans TC` 適用於繁體中文，`Noto Sans SC` 適用於簡體中文，`Noto Sans HK` 適用於香港地區用字。日文可使用 `Noto Sans JP`，韓文可使用 `Noto Sans KR`。

Emoji 顯示需要 `Noto Color Emoji`。另外強烈推薦我愛用的正體中文字體 `Iansui`（芫荽體）！

其它字型請參考 [Google Fonts](https://fonts.google.com/)。

### 方案二適用情境

這個方案{% cg() %}在安全性上較佳{% end %}，因為 Playwright 維持預設的網路限制，僅在 setup steps 階段下載字型。字型安裝在 runner 本地後，{% cg() %}後續的截圖作業不需要網路存取{% end %}。

若專案已知需要的字型種類，這個方案提供更穩定的渲染結果。

## 兩種方案的比較

| 面向 | 方案一：修改 MCP 設定 | 方案二：安裝本地字型 |
|------|---------------------|---------------------|
| 安全性 | {% cr() %}較低（開放外部存取）{% end %} | {% cg() %}較高（保持預設限制）{% end %} |
| 設定複雜度 | 簡單 | 中等 |
| 網路依賴 | 每次渲染都需要 | 僅安裝時需要 |
| 適用場景 | 開發測試、需要完整外部存取 | 生產環境、安全優先 |
| Runner 相容性 | {% cg() %}所有{% end %} | {% cr() %}僅 Ubuntu{% end %} |
| 彈性 | {% cg() %}高（支援任意外部資源）{% end %} | {% cr() %}中（需預先指定字型）{% end %} |

## 實際應用建議

**雖然方案二在安全性上較為嚴謹，我個人更偏好使用方案一，但將 `--allowed-hosts` 參數限定為 Google Fonts 或其他 CDN 的特定網域，而非直接使用萬用字元 `*`。** 這種折衷做法在安全性與便利性之間取得平衡，不需要預先列出所有外部資源，設定流程也相對簡潔。

若專案對安全隔離有極高要求，方案二仍是首選。在正式環境中，{% cg() %}預先安裝已知字型能確保渲染結果的一致性，同時維持 Playwright 的安全限制{% end %}。

## 技術細節補充

### Model Context Protocol 簡介

Model Context Protocol (MCP)[^5] 是一個開放標準，定義了應用程式如何與大型語言模型共享上下文。GitHub Coding Agent 透過 MCP 連接到各種工具和服務，擴展其能力。

Coding Agent 預設配置了兩個 MCP 伺服器：GitHub MCP Server 提供對 GitHub 資料的存取；Playwright MCP Server 提供瀏覽器自動化能力。

### Copilot Setup Steps 運作原理

`copilot-setup-steps.yml` 是一個特殊的 GitHub Actions workflow，在 Coding Agent 開始工作之前執行。這讓我們能預先安裝工具、設定環境變數，或下載必要的資源。

根據官方文件[^4]，這個 workflow 必須：

- 檔案路徑為 `.github/workflows/copilot-setup-steps.yml`
- 包含名為 `copilot-setup-steps` 的 job
- 存在於儲存庫的預設分支

若 setup step 失敗（回傳非零 exit code），Coding Agent 會跳過剩餘的 setup steps，以目前的環境狀態開始工作。

> [!TIP]
> 值得一提的是，{% cg() %}Copilot setup steps 執行時期不受防火牆設定限制{% end %}。

## 參考資源

- [GitHub Copilot Coding Agent 文件](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent)
- [Extending GitHub Copilot coding agent with the Model Context Protocol (MCP)](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/extend-coding-agent-with-mcp)
- [action-install-google-fonts](https://github.com/jim60105/action-install-google-fonts) - 在 GitHub Workflows 中安裝 Google Fonts 的 Action
- [test-runner-fonts](https://github.com/jim60105/test-runner-fonts) - 本文測試用的儲存庫

[^1]: [Customizing or disabling the firewall for GitHub Copilot coding agent - GitHub Docs](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/customize-the-agent-firewall#limitations)
[^2]: [Model Context Protocol (MCP) and GitHub Copilot coding agent - GitHub Docs](https://docs.github.com/en/copilot/concepts/agents/coding-agent/mcp-and-coding-agent#default-mcp-servers)
[^3]: [Playwright MCP Server - GitHub](https://github.com/microsoft/playwright-mcp)
[^4]: [Customizing the development environment for GitHub Copilot coding agent - GitHub Docs](https://docs.github.com/en/copilot/customizing-copilot/customizing-the-development-environment-for-copilot-coding-agent)
[^5]: [Model Context Protocol - Introduction](https://modelcontextprotocol.io/introduction)
