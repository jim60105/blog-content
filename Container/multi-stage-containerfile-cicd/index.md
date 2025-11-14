+++
title = "擺脫 CI/CD 廠商鎖定 (Vendor Lock-in) —— 以 Containerfile 實現跨平台流程"
description = "用 Dockerfile 多階段建構封裝測試邏輯，讓 Azure DevOps、GitLab CI、GitHub Actions 共用同一套流程。詳解 test、report、final 三階段設計，實現本地與雲端環境完全一致，解決平台遷移痛點。"
date = "2025-11-12T08:52:20.157Z"
updated = "2025-11-14T03:07:50.376Z"

[taxonomies]
tags = [ "Container", "DevOps", "Docker" ]
licenses = [ "GFDL 1.3" ]

[extra]
featured = true
withAI = "使用 GitHub Copilot 搭配 Claude Sonnet 4.5 寫作"
banner = "banner.png"

  [extra.preview]
  withAI = true
  description = "Made with Gemini 2.5 Flash Image (Nano Banana) and rework with Flux.1-dev<br>🤖 AI 參考文章簡介做的圖，我也不曉得它想表達什麼 🙃"

  [extra.comments]
  id = "115545837892371362"
+++

在軟體開發的日常中，團隊常常會遇到這樣的狀況：專案 A 用 Azure DevOps 建好了 CI/CD Pipeline，到了專案 B 客戶指定要用 GitLab CI，結果發現測試腳本、環境設定、報告產生邏輯幾乎要全部重寫。每個平台的 YAML 語法不同，維護多套類似功能的腳本既費時又容易出錯。更棘手的是，本機測試通過的程式碼，推上 Pipeline 卻可能因為環境差異而失敗，除錯時很難精確重現雲端的建構環境。

這篇文章要分享一個根本性的解決方案：把 CI/CD 的建構邏輯封裝在 Containerfile 的多階段建構中。透過 `test`、`report`、`final` 三階段設計，讓 Pipeline 變成薄薄一層，只需要一行建構指令就能完成測試和報告產生。更重要的是，本機和雲端使用完全相同的 Containerfile，環境百分之百一致，真正做到跨平台可移植、可重現的建構流程。

<!-- more -->

{% chat(speaker="user") %}
等等，把測試寫在 Containerfile 裡不會讓映像檔變超大嗎？
{% end %}

{% chat(speaker="jim") %}
這就是多階段建構的神奇之處啦！test 階段執行測試，report 階段提取結果，最後的 final 階段只打包生產環境需要的程式碼。測試工具完全不會進到最終映像檔裡。
{% end %}

## 問題場景：平台特定的 CI/CD 困境

<figure>
{{ image(url="works-on-my-machine-v2-2025-jon-galloway-2-1.png", alt="Works on My Machine badge", no_hover=true, end=true, href="https://blog.codinghorror.com/the-works-on-my-machine-certification-program/") }}
</figure>

在現代軟體開發中，我們經常面臨這些挑戰：

**平台鎖定的痛點**：每個 CI/CD 平台都有自己的語法和生態系統。測試腳本、環境變數設定、報告產生邏輯，若全都綁在平台特定的 YAML 檔案裡，團隊得為每個客戶、每個平台維護一套類似但不同的配置，既費時又容易出錯。

**本地與雲端環境不一致**：開發者在本機執行測試時，使用的是本機 Python 環境。但 CI/CD 上可能用的是不同版本的 Python、不同的系統套件，導致「Works on My Machine」的經典問題。除錯時，你無法精確重現 Pipeline 的建構環境。

**Pipeline 配置越來越複雜**：隨著專案成長，Pipeline YAML 檔案動輒數百行。裡面包含套件安裝、測試執行、覆蓋率產生、報告上傳等邏輯。維護這些平台特定的腳本需要學習曲線，新人接手時常常不知從何改起。

## 解決方案：Containerfile 多階段建構

**核心概念**：將 CI/CD 的建構邏輯封裝在 Containerfile 中，利用多階段建構 (Multi-stage Build) 分離不同的建構目標。這樣一來，無論是本機開發還是雲端 Pipeline，都使用同一份 Containerfile，讓環境保持完全一致。

多階段建構允許我們在單一 Containerfile 中定義多個 `FROM` 指令，每個階段有獨立的建構邏輯。後續階段可以從前面階段複製檔案，但不會繼承整個檔案系統，達到映像檔瘦身的效果。

> [!Note]
> 本文不詳細介紹多階段建構的基本概念，建議參考以下官方資源：
>
> - Multi-stage builds | Docker Docs  
>   <https://docs.docker.com/get-started/docker-concepts/building-images/multi-stage-builds/>
> - Multi-stage | Docker Docs  
>   <https://docs.docker.com/build/building/multi-stage/>
> - Best practices | Docker Docs  
>   <https://docs.docker.com/develop/develop-images/dockerfile_best-practices/>

### 架構設計

我們將建構流程分為幾個階段：

<pre class="mermaid">
graph LR
    A[base<br/>基礎環境] --> B[build<br/>安裝相依套件或編譯]
    B --> C[test<br/>執行測試]
    C --> D[report<br/>提取測試結果]
    B --> E[final<br/>生產環境映像檔]
    A --> E
</pre>

**各階段職責**：

- **base**：建立基礎環境，安裝系統套件
- **build**：安裝專案相依套件或編譯，準備應用程式
- **test**：在 build 基礎上安裝開發相依套件並執行測試
- **report**：從 test 階段複製測試結果到空映像檔，供 Pipeline 提取
- **final**：從 base 重新開始，再從 build 提取執行檔產出，僅包含生產環境需要的檔案

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

`--mount=type=bind` 將主機檔案系統掛載到建構容器中，而不是複製檔案。這帶來最主要的好處是能減少映像檔大小，bind mount 不會將檔案複製到映像檔層級，避免不必要的膨脹。

#### 2. BuildKit cache mounts

```dockerfile
RUN --mount=type=cache,id=uv-$TARGETARCH$TARGETVARIANT,sharing=locked,target=/root/.cache/uv \
    uv sync --frozen --no-dev
```

`--mount=type=cache` 在多次建構之間保留快取目錄，讓套件管理器不用每次都重新下載套件，大幅加速建構過程。`id` 參數包含架構資訊，避免跨架構建構時混用快取。這個技巧對於需要下載大量相依套件的專案特別有用。[^2]

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
好問題！主要原因是配合 `--output` flag 使用。

當你用 `--output type=local` 提取檔案時，它會輸出整個 stage 的所有檔案。test stage 包含完整的測試環境、套件、原始碼等，包含很多東西。但 report stage 從 `scratch` 開始，只有我們明確 `COPY` 的測試結果檔案，輸出就只會包含這些報告檔案。

這就是為什麼我特別設計了這個階段。
{% end %}

### 本機執行流程

現在讓我們示範如何在本機使用相同的 Containerfile 執行測試。

#### 步驟 1：建構並執行測試階段

```bash
podman build --target test --tag my-app:test --jobs 0 .
```

這個指令會：

1. 建構 base → build → test 階段
2. 執行所有測試和程式碼品質檢查
3. 產生測試結果和覆蓋率報告（儲存在映像檔內）

如果測試失敗，建構會中止並顯示錯誤訊息。這時你就知道程式碼有問題，需要修正後重新建構。

#### 步驟 2：建構 report stage 並提取測試結果

現代容器建構工具都支援 `--output` flag，可以直接提取測試結果：

```bash
podman build --target report --output type=local,dest=. --jobs 0 .
```

這個指令會直接將 report stage 的所有檔案提取到本機目錄。因為 report stage 從 `scratch` 開始，只包含測試結果檔案，所以輸出內容非常乾淨。

現在你的本機就有 `test-results.xml` 和 `coverage.xml` 了，可以用任何支援 JUnit XML 和 Cobertura XML 的工具檢視結果。

> [!NOTE]
> 關於提取測試結果的其他方法和詳細說明，請參考後面的[提取測試結果](#提取測試結果)章節。

#### 步驟 3：建構生產環境映像檔

```bash
podman build --target final --tag my-app:latest --jobs 0 .
```

final stage 建構出的映像檔不包含任何測試工具或測試結果，只有執行應用程式所需的最小化內容。這個映像檔可以直接部署到生產環境。

### 使用 Buildah 的進階技巧

Buildah 是 Podman 的底層建構工具，提供更細緻的控制。在 CI/CD Pipeline 中，我們也可以直接使用 buildah 指令。

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

`--layers` 參數讓 buildah 保留中間層快取。當你建構 report stage 時，它發現 test stage 的所有層級都已經快取，直接重用即可，不需要重新執行測試。這在 CI/CD Pipeline 中特別重要，可以避免不必要的重複建構。

### 提取測試結果 {#提取測試結果}

這個章節詳細說明如何從 report stage 提取測試結果檔案。根據你的環境和需求，可以選擇以下任一方法。

#### 方法 A：使用 `--output` flag

現代容器建構工具都支援 `--output` flag：

```bash
# 使用 Podman
podman build --target report --output type=local,dest=. --jobs 0 .

# 使用 Docker buildx
docker build --target report --output type=local,dest=. .

# 使用 Buildah
buildah bud --target report --output type=local,dest=. --jobs 0 .
```

`--output` 功能會將指定階段的所有檔案直接輸出到主機檔案系統。因為我們的 report stage 從 `scratch` 開始，只包含從 test stage 複製的測試結果檔案，所以輸出的內容非常乾淨，不會有任何多餘檔案。這也是為什麼要特別設計一個獨立的 report stage 的原因，它讓我們能夠精確控制要提取哪些檔案。

> [!TIP]
> `--jobs 0` 參數在 Podman 和 Buildah 中啟用多執行緒平行建構，加速建構過程。  
> Docker buildx 預設已經支援平行建構，不需要額外參數。

#### 方法 B：使用容器實例複製檔案

透過建立容器實例，使用 `cp` 指令提取檔案：

```bash
# 建構 report stage
podman build --target report --tag my-app:report --jobs 0 .

# 建立容器實例（不啟動）
CONTAINER_ID=$(podman create my-app:report)

# 從容器複製測試結果到本機
podman cp $CONTAINER_ID:/test-results.xml ./test-results.xml
podman cp $CONTAINER_ID:/coverage.xml ./coverage.xml

# 清理容器
podman rm $CONTAINER_ID
```

這個方法適用於所有容器工具（podman、docker），也可以精確指定要複製的檔案路徑。

#### 方法 C：使用 Buildah 掛載檔案系統

Buildah 提供了 `unshare` 功能，可以直接掛載容器檔案系統：

```bash
# 建構 report stage
buildah bud --layers --target report --tag my-app:report --jobs 0 .

# 建立容器實例
CONTAINER_ID=$(buildah from my-app:report)

# 在 rootless 模式下掛載並複製檔案
buildah unshare --mount mnt=$CONTAINER_ID sh -c \
  'cp ${mnt}/test-results.xml ./test-results.xml && \
   cp ${mnt}/coverage.xml ./coverage.xml'

# 清理
buildah rm $CONTAINER_ID
```

`buildah unshare` 在 rootless 模式下建立使用者命名空間，允許掛載容器檔案系統。

`sh` 中可以做 `cp` 之外的事，以滿足更複雜的需求。

## 簡化 Pipeline 配置

有了 Containerfile 封裝的邏輯，Pipeline 變得極其簡潔。

### 傳統 Pipeline（平台特定）

<details>
<summary>展開檢視傳統 Azure DevOps Pipeline 範例</summary>

```yaml,name=傳統做法：所有邏輯都在 Pipeline
# 僅節錄部分
steps:
  - task: UsePythonVersion@0
    inputs:
      versionSpec: '3.13'

  - script: |
      # 安裝 uv - 現代 Python 套件安裝工具
      curl -LsSf https://astral.sh/uv/install.sh | sh
      export PATH="$HOME/.cargo/bin:$PATH"
    displayName: 'Install uv'

  - script: |
      # 建立虛擬環境
      uv venv --system-site-packages /tmp/venv
      source /tmp/venv/bin/activate
      
      # 安裝開發相依套件
      uv sync --frozen --no-install-project
    displayName: 'Install dev dependencies'

  - script: |
      source /tmp/venv/bin/activate
      export PYTHONPATH="$(pwd)/src:/tmp/venv/lib/python3.13/site-packages"
      
      # 執行程式碼格式檢查
      black --check --line-length=100 --skip-string-normalization src/ tests/

      # 執行程式碼風格檢查
      flake8 src/ tests/

      # 執行型別檢查
      mypy src/ --no-incremental
    displayName: 'Run black, flake8, mypy'

  - script: |
      source /tmp/venv/bin/activate
      export PYTHONPATH="$(pwd)/src:/tmp/venv/lib/python3.13/site-packages"

      # 執行測試並產生報告
      pytest \
        --junit-xml=test-results.xml \
        --cov=my_app \
        --cov-report=xml:coverage.xml \
        --cov-fail-under=68 \
        --verbose
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

```yaml,name=新做法：邏輯在 Containerfile，Pipeline 只是薄薄一層
# 僅節錄部分
steps:
  - bash: |
      # 建構並執行測試
      podman build --layers --target test --tag my-app:test --jobs 0 .
      
      # 提取測試結果
      podman build --layers --target report --output type=local,dest=. --jobs 0 .
    displayName: 'Run tests in container'
  
  - task: PublishTestResults@2
    inputs:
      testResultsFormat: 'JUnit'
      testResultsFiles: '**/test-results.xml'
  
  - task: PublishCodeCoverageResults@2
    inputs:
      summaryFileLocation: 'coverage.xml'
```

程式碼指令只有兩行，而且這個 Pipeline 可以輕易移植到任何支援容器的 CI/CD 平台。

{% chat(speaker="jim") %}
有沒有發現？這個 Pipeline 完全不用裝 Python、不用 pip install、不用設定測試框架，不會依賴各個 Agent/Runner 的環境。

所有複雜的邏輯都包在 Containerfile 裡了，Pipeline 只要做兩件事：「建構容器」跟「發布結果」。
{% end %}

### GitHub Actions 移植範例

```yaml,name=GitHub Actions 版本
# 僅節錄部分
steps:
  - name: Run tests in container
    run: |
      # 建構並執行測試
      podman build --layers --target test --tag my-app:test --jobs 0 .
      
      # 提取測試結果
      podman build --layers --target report --output type=local,dest=. --jobs 0 .

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

{% chat(speaker="user") %}
等等，你是說這個流程在哪都跑的完全一樣？
{% end %}

{% chat(speaker="jim") %}
沒錯，一模一樣！這就是整個方案最厲害的地方。不管你在哪裡執行，Containerfile 都定義了統一的建構流程。本機跟雲端用的邏輯、Python 版本、套件版本全部相同。
{% end %}

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

這不影響建構結果，只是失去了一點快取最佳化效果。本機開發環境如果使用 Podman 5.6.0+，可以保留 `--link` 享受更好的效能。[^3]

### 問題 2：層級快取失效

**現象**：`buildah bud` 每次建構 report stage 都重新執行了測試，沒有重用 test stage 快取

**原因**：沒有啟用 `--layers` 參數

**解決方案**：讓兩個階段都使用 `--layers`

```bash
buildah bud --layers --target test --tag my-app:test .
buildah bud --layers --target report --tag my-app:report .
```

## 總結與優勢

將 CI/CD 邏輯封裝在 Containerfile 的多階段建構中，帶來以下核心優勢：

**平台無關性**：Containerfile 是 OCI 標準的一部分，任何支援 OCI 的平台都能執行。這個專案用 Azure DevOps、下個專案用 GitLab CI、客戶自建的 Jenkins，全都可以用同一套 Containerfile，不需要為每個客戶、每個平台維護不同的腳本。團隊的知識和經驗可以完整累積，不會因為換平台就得從頭再來。

**本地與雲端一致性**：開發者在本機執行 `podman build --target test` 得到的結果，和 Pipeline 上執行的完全一致。相同的 Python 版本、相同的系統套件、相同的測試框架配置。這消除了環境差異導致的問題，讓除錯變得簡單直觀。

**簡化 Pipeline 配置**：Pipeline YAML 不再需要處理複雜的環境設定和測試腳本，只需要呼叫建構指令和發布結果。這降低了維護成本，工程師不需要花時間鑽研各平台的細節，專注在應用程式上。

**映像檔大小最佳化**：透過多階段建構，測試工具、測試結果、開發套件都不會進入最終的 final stage。生產環境映像檔保持輕量化，只包含執行應用程式所需的最小內容。

**可追溯性與可重現性**：每次建構都基於同一份 Containerfile，版本控制讓你可以追溯任何歷史建構的確切環境。需要重現三個月前的某次建構？只要 checkout 對應的 git commit，執行相同的 build 指令即可。

{% chat(speaker="jim") %}
這個方案並非萬靈丹，它需要團隊對容器技術有一定的理解。但投入學習成本後，你會發現它帶來的一致性遠超預期。從此你的 CI/CD 不再綁定特定平台，而是真正可移植、可重現的建構流程。
{% end %}

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true, theme: 'dark' });
</script>

[^2]: [RUN --mount | Dockerfile reference | Docker Docs](https://docs.docker.com/reference/dockerfile/#run---mount)
[^3]: [\[RFE\] Add support for `--link` in `COPY_ADD` · Issue #4325 · containers/buildah](https://github.com/containers/buildah/issues/4325#issuecomment-3251167324)
