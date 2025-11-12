+++
title = "將 CI/CD 邏輯寫進 Containerfile：多階段建構的平台無關實踐"
description = "探討如何運用 Containerfile 多階段建構封裝測試與報告產生邏輯，實現平台無關的 CI/CD 流程。透過 test、report、final 三階段設計，讓本地開發環境與雲端建構環境完全一致，擺脫 CI/CD 平台鎖定。"
date = "2025-11-12T08:52:20.157Z"
updated = "2025-11-12T10:07:47.520Z"
draft = true

[taxonomies]
tags = [ "Container", "DevOps", "Docker" ]
licenses = [ "GFDL 1.3" ]

[extra]
featured = true
withAI = "使用 GitHub Copilot 搭配 Claude Sonnet 4.5 寫作"
+++

你的團隊上個專案在 Azure DevOps 寫好 CI/CD Pipeline，這個專案換了客戶要用 GitLab CI，結果發現測試腳本、環境設定、報告產生邏輯全都要重寫一遍。更糟的是，每個平台的 YAML 語法都不一樣，維護多套幾乎相同功能的腳本讓人頭痛。而且本機測試明明通過，推上 Pipeline 卻因為環境差異而失敗，除錯時你根本無法精確重現雲端的建構環境。

這篇文章要介紹一個根本性的解決方案：把 CI/CD 的建構邏輯封裝在 Containerfile 的多階段建構中。透過 `test`、`report`、`final` 三階段設計，讓 Pipeline 變成薄薄一層，只需要一行建構指令就能完成測試和報告產生。更重要的是，本機和雲端使用完全相同的 Containerfile，環境百分之百一致。從此你的 CI/CD 不再被特定平台綁架，真正做到可移植、可重現。

{% chat(speaker="user") %}
等等，把測試寫在 Containerfile 裡不會讓映像檔變超大嗎？
{% end %}

{% chat(speaker="jim") %}
這就是多階段建構的神奇之處啦！test 階段執行測試，report 階段提取結果，最後的 final 階段只打包生產環境需要的程式碼。測試工具完全不會進到最終映像檔裡。
{% end %}

<!-- more -->

## 問題場景：平台特定的 CI/CD 困境

在現代軟體開發中，我們經常面臨這些挑戰：

**平台鎖定的痛點**：每個 CI/CD 平台都有自己的語法和生態系統。上個專案用 Azure DevOps 寫好的 Pipeline，到了新專案客戶要求用 GitLab CI，結果幾乎要全部重寫。測試腳本、環境變數設定、報告產生邏輯，全都綁在平台特定的 YAML 檔案裡。團隊得為每個客戶、每個平台維護一套類似的配置，既費時又容易出錯。

**本地與雲端環境不一致**：開發者在本機執行測試時，使用的是本機 Python 環境。但 CI/CD 上可能用的是不同版本的 Python、不同的系統套件，導致「在我電腦上可以跑」的經典問題。除錯時，你無法精確重現 Pipeline 的建構環境。

**Pipeline 配置越來越複雜**：隨著專案成長，Pipeline YAML 檔案動輒數百行。裡面包含套件安裝、測試執行、覆蓋率產生、報告上傳等邏輯。維護這些平台特定的腳本需要學習曲線，新人接手時常常不知從何改起。

## 解決方案：Containerfile 多階段建構

**核心概念**：將 CI/CD 的建構邏輯封裝在 Containerfile 中，利用多階段建構 (Multi-stage Build) 分離不同的建構目標。這樣一來，無論是本機開發還是雲端 Pipeline，都使用同一份 Containerfile，讓環境保持完全一致。

多階段建構允許我們在單一 Containerfile 中定義多個 `FROM` 指令，每個階段有獨立的建構邏輯。後續階段可以從前面階段複製檔案，但不會繼承整個檔案系統，達到映像檔瘦身的效果。這個功能自 Docker 17.05 版本引入，現在已經是容器建構的標準實踐。

> [!NOTE]
> **延伸閱讀**：
>
> - [Docker 官方文件：Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
> - [Buildah 指令參考](https://buildah.io/blogs/)
> - [Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

### 架構設計

我們將建構流程分為四個階段：

<pre class="mermaid">
graph LR
    A[base<br/>基礎環境] --> B[build<br/>安裝相依套件]
    B --> C[test<br/>執行測試]
    C --> D[report<br/>提取測試結果]
    B --> E[final<br/>生產環境映像檔]

    style A fill:#e1f5ff
    style B fill:#fff4e1
    style C fill:#ffe1e1
    style D fill:#f0e1ff
    style E fill:#e1ffe1
</pre>

**各階段職責**：

- **base**：建立基礎環境，安裝系統套件
- **build**：安裝專案相依套件，準備應用程式
- **test**：在 build 基礎上安裝開發相依套件並執行測試
- **report**：從 test 階段複製測試結果到空映像檔，供 Pipeline 提取
- **final**：從 base 重新開始，僅包含生產環境需要的檔案

{% chat(speaker="jim") %}
重點來了！test 和 final 雖然都從前面的階段延伸出來，但彼此完全獨立。test 階段會有完整的測試工具和結果檔案，final 階段則乾乾淨淨，只留下執行程式需要的東西。
{% end %}

## 實作：Python 專案範例

讓我們以一個 Python FastAPI 專案為例，展示完整的實作細節。

### 完整 Containerfile

<details>
<summary>點擊展開完整 Containerfile</summary>

```dockerfile
# syntax=docker/dockerfile:1
ARG UID=1001
ARG VERSION=latest
ARG RELEASE=0

########################################
# Base stage
########################################
FROM python:3.13-alpine AS base

WORKDIR /tmp

########################################
# Build stage
########################################
FROM base AS build

# RUN mount cache for multi-arch
ARG TARGETARCH
ARG TARGETVARIANT

WORKDIR /app

# Install uv - modern Python package installer
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# UV configuration
ENV UV_PROJECT_ENVIRONMENT=/venv
ENV VIRTUAL_ENV=/venv
ENV UV_LINK_MODE=copy
ENV UV_PYTHON_DOWNLOADS=0

# Create virtual environment
RUN --mount=type=cache,id=uv-$TARGETARCH$TARGETVARIANT,sharing=locked,target=/root/.cache/uv \
    uv venv --system-site-packages /venv

# Install dependencies (production only)
RUN --mount=type=cache,id=uv-$TARGETARCH$TARGETVARIANT,sharing=locked,target=/root/.cache/uv \
    --mount=type=bind,source=pyproject.toml,target=/app/pyproject.toml \
    --mount=type=bind,source=uv.lock,target=/app/uv.lock \
    uv sync --frozen --no-dev --no-install-project

# Install project
RUN --mount=type=cache,id=uv-$TARGETARCH$TARGETVARIANT,sharing=locked,target=/root/.cache/uv \
    --mount=type=bind,source=.,target=.,rw \
    uv sync --frozen --no-dev --no-editable

########################################
# Test stage
########################################
FROM build AS test

ARG TARGETARCH
ARG TARGETVARIANT

# Test environment
ENV PATH="/venv/bin${PATH:+:${PATH}}"
ENV PYTHONPATH="/app/src:/venv/lib/python3.13/site-packages"

# Install dev dependencies (including pytest, coverage, linters)
WORKDIR /app
RUN --mount=type=cache,id=uv-test-$TARGETARCH$TARGETVARIANT,sharing=locked,target=/root/.cache/uv \
    --mount=type=bind,source=pyproject.toml,target=/app/pyproject.toml \
    --mount=type=bind,source=uv.lock,target=/app/uv.lock \
    uv sync --frozen --no-install-project

# Run tests with bind-mounted source code
# This generates JUnit XML and Cobertura XML reports
RUN --mount=type=bind,source=src,target=/app/src \
    --mount=type=bind,source=tests,target=/app/tests \
    --mount=type=bind,source=.flake8,target=/app/.flake8 \
    --mount=type=bind,source=pyproject.toml,target=/app/pyproject.toml \
    black --check --line-length=100 --skip-string-normalization src/ tests/ && \
    flake8 src/ tests/ && \
    mypy src/ --no-incremental && \
    pytest \
        --junit-xml=/app/test-results.xml \
        --cov=my_app \
        --cov-report=xml:/app/coverage.xml \
        --cov-fail-under=68 \
        --verbose

########################################
# Report stage
########################################
FROM scratch AS report

ARG UID=1001
COPY --chown=$UID:0 --chmod=775 --from=test /app/test-results.xml /
COPY --chown=$UID:0 --chmod=775 --from=test /app/coverage.xml /

########################################
# Final stage
########################################
FROM base AS final

# Clean up pip
RUN pip uninstall -y pip wheel && \
    rm -rf /root/.cache/pip

ARG TARGETARCH
ARG TARGETVARIANT

# Install runtime dependencies
RUN --mount=type=cache,id=apk-$TARGETARCH$TARGETVARIANT,sharing=locked,target=/var/cache/apk \
    apk update && apk add -u dumb-init

# Create user
ARG UID
RUN adduser -g "" -D $UID -u $UID -G root

# Create directories
RUN install -d -m 775 -o $UID -g 0 /app && \
    install -d -m 775 -o $UID -g 0 /licenses && \
    install -d -m 775 -o $UID -g 0 /venv

# Copy licenses
COPY --link --chown=$UID:0 --chmod=775 LICENSE /licenses/LICENSE

# Copy dependencies from build stage
COPY --link --chown=$UID:0 --chmod=775 --from=build /venv /venv

# Environment setup
ENV PATH="/venv/bin${PATH:+:${PATH}}"
ENV PYTHONPATH="/venv/lib/python3.13/site-packages"

WORKDIR /app

VOLUME [ "/app" ]

USER $UID

STOPSIGNAL SIGINT

EXPOSE 8001

ENTRYPOINT ["dumb-init", "--", "uvicorn", "my_app.api.main:app"]
CMD ["--host", "0.0.0.0", "--port", "8001"]

# Labels
ARG VERSION
ARG RELEASE
LABEL name="my-app" \
      vendor="Your Organization" \
      maintainer="Your Team" \
      version=${VERSION} \
      release=${RELEASE} \
      summary="My Application" \
      description="Application description"
```

</details>

### Containerfile 關鍵技術解析

#### 1. 使用 bind mounts 加速建構

```dockerfile
RUN --mount=type=bind,source=src,target=/app/src \
    --mount=type=bind,source=tests,target=/app/tests \
    pytest --junit-xml=/app/test-results.xml
```

`--mount=type=bind` 將主機檔案系統掛載到建構容器中，而不是複製檔案。這帶來兩個好處：

- **加速建構**：避免大量檔案複製，特別是程式碼檔案頻繁變更時
- **減少層級大小**：bind mount 不會建立新的映像檔層級

#### 2. BuildKit cache mounts

```dockerfile
RUN --mount=type=cache,id=uv-$TARGETARCH$TARGETVARIANT,sharing=locked,target=/root/.cache/uv \
    uv sync --frozen --no-dev
```

`--mount=type=cache` 在多次建構之間保留快取目錄，讓套件管理器不用每次都重新下載套件，大幅加速建構過程。`id` 參數包含架構資訊，避免跨架構建構時混用快取。這個技巧對於 Python、Node.js、Rust 等需要下載大量相依套件的專案特別有用。

#### 3. 從 scratch 建立 report stage

```dockerfile
FROM scratch AS report
COPY --from=test /app/test-results.xml /
COPY --from=test /app/coverage.xml /
```

`scratch` 是一個空映像檔，不包含任何檔案系統。這樣建立的 report stage 只有我們明確複製的測試結果檔案，映像檔大小僅幾 KB。

{% chat(speaker="user") %}
欸等一下，為什麼要特別弄一個 report stage？直接從 test stage 拿結果不就好了？
{% end %}

{% chat(speaker="jim") %}
技術上當然可以啊，但 report stage 有兩個好處：第一是映像檔超小只有幾 KB，推送拉取都超快；第二是它的用途一目了然——「這個階段就是給你測試結果的」，架構會清楚很多。
{% end %}

### 本機執行流程

現在讓我們示範如何在本機使用相同的 Containerfile 執行測試。

#### 步驟 1：建構並執行測試階段

```bash
# 使用 Podman (或 docker 指令完全相同)
podman build --target test --tag my-app:test .
```

這個指令會：

1. 建構 base → build → test 階段
2. 執行所有測試和程式碼品質檢查
3. 產生測試結果和覆蓋率報告（儲存在映像檔內）

如果測試失敗，建構會中止並顯示錯誤訊息。這時你就知道程式碼有問題，需要修正後重新建構。

#### 步驟 2：建構 report stage 並提取測試結果

```bash
# 建構 report stage
podman build --target report --tag my-app:report .

# 建立容器實例（不啟動）
CONTAINER_ID=$(podman create my-app:report)

# 從容器複製測試結果到本機
podman cp $CONTAINER_ID:/test-results.xml ./test-results.xml
podman cp $CONTAINER_ID:/coverage.xml ./coverage.xml

# 清理容器
podman rm $CONTAINER_ID
```

現在你的本機就有 `test-results.xml` 和 `coverage.xml` 了，可以用任何支援 JUnit XML 和 Cobertura XML 的工具檢視結果。

{% chat(speaker="user") %}
等等，你是說這個流程跟 CI/CD Pipeline 上跑的完全一樣？
{% end %}

{% chat(speaker="jim") %}
沒錯，一模一樣！這就是整個方案最厲害的地方。不管你在哪裡執行，Containerfile 都定義了統一的建構流程。本機跟雲端用的邏輯、Python 版本、套件版本全部相同。
{% end %}

#### 步驟 3：建構生產環境映像檔

```bash
podman build --target final --tag my-app:latest .
```

final stage 建構出的映像檔不包含任何測試工具或測試結果，只有執行應用程式所需的最小化內容。這個映像檔可以直接部署到生產環境。

### 使用 Buildah 的進階技巧

Buildah 是 Podman 的底層建構工具，提供更細緻的控制。在 CI/CD Pipeline 中，我們通常直接使用 buildah 指令。

#### Buildah 關鍵參數

| 參數 | 說明 | 範例 |
|------|------|------|
| `--target <stage>` | 建構至指定階段 | `--target test` |
| `--jobs=N` | 平行建構任務數量，0 表示使用所有核心 | `--jobs=0` |
| `--layers` | 啟用層級快取，加速重複建構 | `--layers` |
| `--build-arg KEY=VALUE` | 傳遞建構參數 | `--build-arg VERSION=v1.0` |
| `--tag NAME:TAG` | 標記建構的映像檔 | `--tag my-app:test` |

#### 層級快取的威力

```bash
# 第一次建構 test stage（執行所有測試）
buildah bud --layers --target test --tag my-app:test .

# 建構 report stage（重用 test 快取，幾乎瞬間完成）
buildah bud --layers --target report --tag my-app:report .
```

`--layers` 參數讓 buildah 保留中間層快取。當你建構 report stage 時，它發現 test stage 的所有層級都已經快取，直接重用即可，不需要重新執行測試。這在 CI/CD Pipeline 中特別重要，可以大幅縮短建構時間。

#### 提取測試結果（Rootless 模式）

```bash
# 建構 report stage
buildah bud --layers --target report --tag my-app:report .

# 建立容器實例
CONTAINER_ID=$(buildah from my-app:report)

# 在 rootless 模式下掛載並複製檔案
buildah unshare --mount mnt=$CONTAINER_ID sh -c \
  'cp ${mnt}/test-results.xml ./test-results.xml && \
   cp ${mnt}/coverage.xml ./coverage.xml'

# 清理
buildah rm $CONTAINER_ID
```

`buildah unshare` 在 rootless 模式下建立使用者命名空間，允許掛載容器檔案系統。這是 Podman/Buildah 無需 root 權限執行的關鍵技術。

## 簡化 Pipeline 配置

有了 Containerfile 封裝的邏輯，Pipeline 變得極其簡潔。

### 傳統 Pipeline（平台特定）

<details>
<summary>展開檢視傳統 Azure DevOps Pipeline 範例（約 80 行）</summary>

```yaml
# 傳統做法：所有邏輯都在 Pipeline
steps:
  - task: UsePythonVersion@0
    inputs:
      versionSpec: '3.13'
  
  - script: |
      python -m pip install --upgrade pip
      pip install -r requirements.txt
      pip install -r requirements-dev.txt
    displayName: 'Install dependencies'
  
  - script: |
      pip install black flake8 mypy
      black --check --line-length=100 src/ tests/
      flake8 src/ tests/
      mypy src/
    displayName: 'Run linters'
  
  - script: |
      pip install pytest pytest-cov
      pytest \
        --junit-xml=test-results.xml \
        --cov=my_app \
        --cov-report=xml:coverage.xml \
        --cov-fail-under=68
    displayName: 'Run tests'
  
  - task: PublishTestResults@2
    inputs:
      testResultsFormat: 'JUnit'
      testResultsFiles: '**/test-results.xml'
  
  - task: PublishCodeCoverageResults@2
    inputs:
      summaryFileLocation: 'coverage.xml'
```

</details>

這個 Pipeline 有幾個問題：

- Python 版本、套件安裝、測試指令全都寫在 YAML 中
- 本機要重現這個環境，需要手動設定相同的 Python 版本和套件
- 下個專案換到 GitLab CI 或 GitHub Actions，這些腳本幾乎全要重寫

### 新方案：極簡 Pipeline

```yaml
# 新做法：邏輯在 Containerfile，Pipeline 只是薄薄一層
steps:
  - bash: |
      # 建構並執行測試
      buildah bud --layers --target test --tag my-app:test .
      
      # 提取測試結果
      buildah bud --layers --target report --tag my-app:report .
      CONTAINER_ID=$(buildah from my-app:report)
      buildah unshare --mount mnt=$CONTAINER_ID sh -c \
        'cp ${mnt}/test-results.xml ./test-results.xml && \
         cp ${mnt}/coverage.xml ./coverage.xml'
      buildah rm $CONTAINER_ID
    displayName: 'Run tests in container'
  
  - task: PublishTestResults@2
    inputs:
      testResultsFormat: 'JUnit'
      testResultsFiles: '**/test-results.xml'
  
  - task: PublishCodeCoverageResults@2
    inputs:
      summaryFileLocation: 'coverage.xml'
```

程式碼行數減少了一半以上，而且這個 Pipeline 可以輕易移植到任何支援容器的 CI/CD 平台。只需要把 `buildah` 指令換成 `docker` 或 `podman`，其他完全不變。

{% chat(speaker="jim") %}
有沒有發現？這個 Pipeline 完全不用裝 Python、不用 pip install、不用設定測試框架。所有複雜的邏輯都包在 Containerfile 裡了，Pipeline 只要做兩件事：「建構容器」跟「發布結果」。
{% end %}

### GitHub Actions 移植範例

```yaml
# GitHub Actions 版本（幾乎一模一樣）
steps:
  - uses: actions/checkout@v4
  
  - name: Run tests in container
    run: |
      docker build --target test --tag my-app:test .
      docker build --target report --tag my-app:report .
      CONTAINER_ID=$(docker create my-app:report)
      docker cp $CONTAINER_ID:/test-results.xml ./test-results.xml
      docker cp $CONTAINER_ID:/coverage.xml ./coverage.xml
      docker rm $CONTAINER_ID
  
  - name: Publish test results
    uses: EnricoMi/publish-unit-test-result-action@v2
    with:
      files: test-results.xml
  
  - name: Publish coverage
    uses: codecov/codecov-action@v3
    with:
      files: coverage.xml
```

下個專案如果改用 GitHub Actions，只需要換掉發布測試結果的 action，建構邏輯完全不用改。同樣的 Containerfile 可以直接沿用。

## 故障排除

在實際應用中，你可能會遇到一些常見問題。

### 問題 1：`--link` flag 相容性

**現象**：Pipeline 建構失敗，錯誤訊息提示 `unknown flag: --link`

```dockerfile
COPY --link src/ /app/src/
```

**原因**：`--link` 是 BuildKit 的進階功能，用於最佳化層級快取，但需要 buildah 1.41.0+ 才支援。部分 CI/CD 平台的 buildah 版本較舊。

**解決方案**：在 Pipeline 中動態移除 `--link` flag

```bash
sed -i 's/COPY --link /COPY /g' Containerfile
buildah bud --target test .
```

這不影響建構結果，只是失去了一點快取最佳化效果。本機開發環境如果使用 Podman 5.6.0+，可以保留 `--link` 享受更好的效能。

### 問題 2：層級快取失效

**現象**：每次建構 report stage 都重新執行了測試，沒有重用 test stage 快取

**原因**：沒有啟用 `--layers` 參數

**解決方案**：讓兩個階段都使用 `--layers`

```bash
buildah bud --layers --target test --tag my-app:test .
buildah bud --layers --target report --tag my-app:report .
```

### 問題 3：測試結果無法提取

**現象**：`buildah unshare` 執行後找不到測試結果檔案

**原因**：測試結果可能寫在錯誤的路徑，或者 pytest 配置有誤

**解決方案**：在 test stage 加上驗證步驟

```dockerfile
# 在 RUN pytest 之後加上
RUN ls -la /app/test-results.xml /app/coverage.xml
```

確認檔案確實產生在正確位置。另外檢查 `pytest.ini` 或 `pyproject.toml` 中的輸出路徑配置。

## 總結與優勢

將 CI/CD 邏輯封裝在 Containerfile 的多階段建構中，帶來以下核心優勢：

**平台無關性**：Containerfile 是 OCI 標準的一部分，任何支援 OCI 的平台都能執行。這個專案用 Azure DevOps、下個專案用 GitLab CI、客戶自建的 Jenkins，全都可以用同一套 Containerfile，不需要為每個客戶、每個平台維護不同的腳本。團隊的知識和經驗可以完整累積，不會因為換平台就得重新學習。

**本地與雲端一致性**：開發者在本機執行 `podman build --target test` 得到的結果，和 Pipeline 上執行的完全一致。相同的 Python 版本、相同的系統套件、相同的測試框架配置。這消除了環境差異導致的問題，讓除錯變得簡單直觀。

**簡化 Pipeline 配置**：Pipeline YAML 不再需要處理複雜的環境設定和測試腳本，只需要呼叫建構指令和發布結果。這降低了維護成本，新人可以快速理解 Pipeline 的運作方式。

**映像檔大小最佳化**：透過多階段建構，測試工具、測試結果、開發套件都不會進入最終的 final stage。生產環境映像檔保持輕量化，只包含執行應用程式所需的最小內容。

**可追溯性與可重現性**：每次建構都基於同一份 Containerfile，版本控制讓你可以追溯任何歷史建構的確切環境。需要重現三個月前的某次建構？只要 checkout 對應的 git commit，執行相同的 build 指令即可。

{% chat(speaker="user") %}
聽起來蠻實用的耶！那我們要從哪裡開始試比較好？
{% end %}

{% chat(speaker="jim") %}
建議先從新專案或比較小的專案開始玩玩看。把現有的測試腳本搬到 Containerfile 的 test stage，確認本機跑得通之後再去改 Pipeline。記得留著原本的 Pipeline 當備份，等測試穩定了再完全切過去。
{% end %}

這個方案不是銀彈，它需要團隊對容器技術有一定的理解。但投入學習成本後，你會發現它帶來的靈活性和一致性遠超預期。從此你的 CI/CD 不再綁定特定平台，而是真正可移植、可重現的建構流程。

---

> [!TIP]
> 完整的範例專案和 Containerfile 可以在 [這個 GitHub Gist](https://gist.github.com/) 找到（連結待補充）。

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true });
</script>
