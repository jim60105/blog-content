+++
title = "[Docker 專案] 一行指令運行 Stable Diffusion WebUI 和 Kohya's GUI"
description = "這篇文章介紹如何運用 Docker 容器化啟動 Stable Diffusion WebUI 和 kohya_ss 兩個 AI 繪圖工具。經過最佳化的 Dockerfile 成功地減小了映像尺寸大小，同時，文章中提供了使用指南和建置步驟，並說明如何將現有的設定遷移到 Docker 環境中。"
date = 2024-03-11T15:35:00.026Z
updated = "2026-01-01T15:57:23.219Z"
draft = false
aliases = [ "/2024/03/docker-stable-diffusion-webui-kohyas-gui.html" ]

[taxonomies]
tags = [ "AI", "Container" ]
licenses = [ "GFDL 1.3" ]

[extra]
banner = "preview.jpg"
hot = true

  [extra.preview]
  description = '↑ Stable Diffusion 最有名的圖片是<a href="https://zh.wikipedia.org/zh-tw/File:A%5Fphotograph%5Fof%5Fan%5Fastronaut%5Friding%5Fa%5Fhorse%5F2022-08-28.png" target="_blank">一個太空人騎馬</a>。我們用 Docker，讓他騎個鯨魚🐋！'
  url = "https://civitai.com/images/7792500"
  withAI = true
+++
每次只要遇到 Python 專案，我在運行前一定先看**有沒有提供 Dockerfile**。

Why?

你有沒有遇過安裝了 Python 2.7, 3.8, 3.9, 3.11，然後新程式只能跑在 3.10。  
重點是還不一定能升級，裝好了 Python 3.11 不代表你可以跑 3.8, 3.9, 3.10 的程式！  
Python Developer 裝這麼多版本不累嗎😅  

我知道你會說 conda，玩 Python 的人不可能沒用過 [Anaconda](https://www.anaconda.com/)，它用來解決這惱人的依頼問題，讓你方便地換個程式就切換一套運行環境。  
這發明很棒，真的，「先有需求」才有供給對吧😏

<span class="danger">直到有一天 C 槽被各版本的 Python、各程式專屬的 environment 給佔滿</span>  
我最終決定把它們全都砍了，從此以後打死不在本機裝 Python  
通通給我到容器🐋裡去吧！

> 不方便開發?
>
> ---
> 開發需求推薦 [devcontainer](https://code.visualstudio.com/learn/develop-cloud/containers)  
> 專案做好 `.devcontainer` 設定檔 & 一鍵建立 GitHub CodeSpaces  
> Python 開發體驗一級棒👍  
>
> <span class="spoiler">......前提是你懂 Docker</span>

說回今天的主題，**Stable Diffusion WebUI** 和 **Kohya's GUI**，這兩套程式都有前人做了容器化。  
我過往看過 Dockerfile 後認為「**有最佳化的餘地，但沒什麼大問題**」並直接使用了一段時間。

前兩週比較有空閒，於是來貢獻貢獻所學，重寫了更好的 Dockerfile。

[bmaltais/kohya\_ss](https://github.com/bmaltais/kohya%5Fss) 已經 PR 回原專案，現在 [master branch 上面的 Dockerfile](https://github.com/bmaltais/kohya%5Fss/blob/master/Dockerfile) 是我重寫的  
[AUTOMATIC1111/stable-diffusion-webui](https://github.com/AUTOMATIC1111/stable-diffusion-webui) 不接受 Dockerfile，我是放在[自己的 GitHub，並做了整套的 CI](https://github.com/jim60105/docker-stable-diffusion-webui)

<!-- more -->

> 更新 2024/04/26:
>
> ---
> 我最終將 kohya-ss-gui 縮減到了 <span class="success">10.3GB</span>，並且幫 `bmaltais` 建了 [docker build CI](https://github.com/bmaltais/kohya%5Fss/blob/master/.github/workflows/docker%5Fpublish.yml)  
> 它現在會自動建置映像並推送至 [ghcr.io/bmaltais/kohya-ss-gui](https://github.com/bmaltais/kohya%5Fss/pkgs/container/kohya-ss-gui)

Stable Diffusion WebUI 的對照組[是它](https://github.com/AbdBarho/stable-diffusion-webui-docker)。  
前人寫得很不錯，是很棒的參考🙏

想了解技術的人推薦把我的 Dockerfile 拿來讀一次，相信能讓你有所收獲。  
也歡迎在底下留言找我，我樂意向你解釋每一行為何這樣子設計。

## 🏁 簡介 Stable Diffusion WebUI 和 Kohya's GUI

### Stable Diffusion WebUI

[Stable Diffusion](https://zh.wikipedia.org/zh-tw/Stable%5FDiffusion) 是一個強大的 AI 影像生成模型，由 Stability AI 開發。它能根據文字描述生成高品質、多樣化的圖像。而 [AUTOMATIC1111/stable-diffusion-webui](https://github.com/AUTOMATIC1111/stable-diffusion-webui) 開源專案為這個模型創造了一個簡單易用的介面，強大的擴充框架讓其生態系蓬勃發展。

![stable-diffusion-webui](stable-diffusion-webui.png)

### Kohya's GUI

[bmaltais/kohya\_ss](https://github.com/bmaltais/kohya%5Fss) 是將 [kohya-ss/sd-scripts](https://github.com/kohya-ss/sd-scripts) 打包起來的 Gradio GUI 專案，能讓使用者用簡單易用的網頁介面去訓練適用於 Stable Diffusion 的繪圖模型。

> 如果你不認識它們，推薦先看這篇影片
>
> ---
> {{ youtube(id="3pdtwip-wNQ") }}

## 🚀 準備好讓你的 Docker 支援 GPU

### Windows

只要完成安裝 [**Docker Desktop**](https://www.docker.com/products/docker-desktop/)、[**CUDA Toolkit**](https://developer.nvidia.com/cuda-downloads)、[**NVIDIA Windows Driver**](https://www.nvidia.com.tw/Download/index.aspx)，並確保 Docker 是使用 [**WSL2**](https://docs.docker.com/desktop/wsl/#turn-on-docker-desktop-wsl-2) 運行，那麼你就準備好了。

以下官方文件提供參考  
<https://docs.nvidia.com/cuda/wsl-user-guide/index.html#nvidia-compute-software-support-on-wsl-2>  
<https://docs.docker.com/desktop/wsl/use-wsl/#gpu-support>

> 延伸閱讀
>
> ---
> Will 保哥有一篇專文介紹  
> [如何在 Windows 的 Docker Desktop 中啟用 NVIDIA CUDA 支援 (GPU)](https://blog.miniasp.com/post/2024/02/28/Enable-GPU-NVIDIA-CUDA-Support-for-Docker-Desktop-on-Windows)

### Linux, OSX

請安裝 NVIDIA GPU 驅動程式  
<https://docs.nvidia.com/datacenter/tesla/tesla-installation-notes/index.html>

並按照此指南安裝 NVIDIA Container Toolkit  
<https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html>

## 🖥️ 如何使用這個專案

1. 將儲存庫 clone 到本地並導航到該目錄。**你也可以點擊下方連結下載程式碼壓縮檔**，是一樣的  
   * [Stable Diffusion WebUI](https://github.com/jim60105/docker-stable-diffusion-webui/archive/refs/heads/master.zip):  

   ```bash
   git clone https://github.com/jim60105/docker-stable-diffusion-webui.git
   cd docker-stable-diffusion-webui
    ```

   * [kohya\_ss](https://github.com/bmaltais/kohya%5Fss/archive/refs/heads/master.zip):  

   ```bash
   git clone https://github.com/bmaltais/kohya_ss.git
   cd kohya_ss
   ```

2. 在資料夾中 「Shift+右鍵」 → 「在這裡開啟 Powershell 視窗」，輸入指令啟動服務並等待數分鐘載入。  
標題說的<span class="success">一行指令</span>就是下面這行啦😉  

   ```bash
   docker compose up -d
   ```

3. 在 <http://localhost:7860> 訪問 Web UI。  
   <span class="danger">(瀏覽器不會自動啟動！)</span>

> 模型和設置將被儲存在目錄 `./data`  
> 在預設情況下，輸出的圖片將儲存在`./data/output`

## 🛠️ 建置指南

> <span class="danger">Git clone 儲存庫時使用 `--recursive` 來包含子模組</span>
>
> ```bash
> git clone --recursive https://github.com/jim60105/docker-stable-diffusion-webui.git
> ```

使用以下指令建置映像。

```bash
docker compose up -d --build
```

> 若你使用舊版的 Docker 客戶端，請在建置時[啟用 BuildKit 模式](https://docs.docker.com/build/buildkit/#getting-started)。這是因為我使用了 `COPY --link` 功能，該功能在 Buildx v0.8 中被導入並可增強建置效能。隨著 Docker Engine 23.0 和 Docker Desktop 4.19 的推出，Buildx 已成為預設的建置客戶端。  
>
> 因此，在使用最新版本時不必擔心這個問題喔！

## 🔄 從現有的設定檔遷移

1. 編輯你現有的 `config.json`，將所有路徑修改為在 `/data` 之下，舉例來說：  

   ```json
   {
       "outdir_samples": "",
       "outdir_txt2img_samples": "/data/output/txt2img-images",
       "outdir_img2img_samples": "/data/output/img2img-images",
       "outdir_extras_samples": "/data/output/extras-images",
       "outdir_grids": "",
       "outdir_txt2img_grids": "/data/output/txt2img-grids",
       "outdir_img2img_grids": "/data/output/img2img-grids",
       "outdir_save": "/data/log/images",
       "outdir_init_images": "/data/output/init-images",
   }
   ```

2. 將 `config.json` 放置在 `data` 目錄下。
3. 將模型和其他現有資料放入 `data` 目錄下相應的資料夾中。
4. **若這些文件來自於 Linux 系統 (你之前使用 WSL 或 Linux)，請修正`data` 資料夾中所有文件的權限。**  

   ```bash
   docker run -v ".:/app" -it busybox sh -c "chown -R 1001:0 /app/data && chmod -R 775 /app/data"
   ```

   > 這個指令修改了 `data` 目錄的擁有者群組為 **0(root group)**，並授予了 **群組寫入權限**。這符合 OpenShift 最佳實踐的<span class="success">支援以任意 uid 運行</span>。
   >
   > 這個映像遵循最佳實踐，<span class="success">使用非 root 用戶運行</span> 並 <span class="success">限制非必要路徑的寫入權限</span>。除非進行了適當的修改，否則你可能無法將文件儲存在 `/data` 路徑之外。

> 延伸閱讀
>
> ---
>
> * [\[Docker\] Linux主機之Docker安裝和 ReverseProxy 建置](/Container/linux-docker-setup-reverse-proxy)
> * [琳的備忘手札: AI翻譯解鎖——日文網路小說無障礙閱讀體驗](/AI/unlocking-ai-translation-barrier-free-reading-experience-of-japanese-web-novels)
