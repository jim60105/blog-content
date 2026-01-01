+++
title = "介紹 WhisperX: AI 語音識別加上單詞級別的時間戳記和語音分離"
description = "介紹 WhisperX: AI 語音識別加上單詞級別的時間戳記和語音分離"
date = 2023-10-31T14:08:00.013Z
updated = 2024-08-31T18:50:59.822Z
draft = false
aliases = [ "/2023/10/whisperx-ai.html" ]

[taxonomies]
tags = [ "AI" ]
licenses = [ "GFDL 1.3" ]

[extra]
iscn = "iscn://likecoin-chain/i6XMdfLq3nGAU_inrEqxpsDKOlCbHyUkwJZLl-U1aXk/1"
hot = true
banner = "preview.png"

  [extra.comments]
  id = "agzboi29x37i00u8"
+++
## 前言

上個月買遊戲拿到了 FANZA 的折抵金，為了花掉它買了平常沒在聽的 ASMR，然後發現

嗯，我 N87 聽不懂日文...

這種小眾的東西網路上也沒有譯文，就試了試當紅的 AI 語音辨識 + 翻譯，成果還不賴。  
這篇文章想分享我發現的好東西 —— WhisperX。

<!-- more -->

## WhisperX: Automatic Speech Recognition with Word-level Timestamps (Diarization)

WhisperX 提供 **高速** 的自動語音識別，並提供詞級時間戳和講話者分離功能。  
(**高速** 指使用 large-v2 模型時可達到 whisper 的 70 倍速)

> GitHub: m-bain/whisperX
>
> ---
> <https://github.com/m-bain/whisperX>

* 使用 whisper large-v2 進行批次推論，實現 70 倍速的轉錄速度
* 以 faster-whisper 做後端，對於 large-v2 使用 beam\_size=5 只需要 8GB 以下的 GPU 顯存
* 使用 wav2vec2 對齊技術達到單詞級別的時間戳
* 利用 pyannote-audio 中的語者分離功能實現多人語音辨識（包含語者 ID 標籤）
* VAD 預處理，減少幻聽問題並提升批次處理效率而不影響字錯率

一言以蔽之，它會把音檔分割成小片段批次送去 whisper 辨識，再將時間軸對齊到單字的層級上。另外它底下是使用 faster-whisper 模型，號稱能在同樣的辨識率下達到 4 倍快速。

實測是真的很快🚀

> 上方數據是 large-v2，不過它有支援新的 large-v3

### 安裝

請按照 [GitHub 文件](https://github.com/m-bain/whisperX#setup-%EF%B8%8F) 安裝

```bash
conda create --name whisperx python=3.10
conda activate whisperx
conda install pytorch==2.0.0 torchaudio==2.0.0 pytorch-cuda=11.8 -c pytorch -c nvidia
pip install git+https://github.com/m-bain/whisperx.git --upgrade
```

或者是...↓

### Docker 執行

推薦使用我寫的 Docker image，一行指令啟動，省去折騰 Python 環境的時間 💪  
此方法你需要安裝好 [Docker Desktop](https://www.docker.com/products/docker-desktop/), [CUDA Toolkit](https://developer.nvidia.com/cuda-downloads), [NVIDIA Windows Driver](https://www.nvidia.com.tw/Download/index.aspx?lang=tw)，並確保你的 [Docker 運行 WSL2 上](https://docs.docker.com/desktop/wsl/#turn-on-docker-desktop-wsl-2)。

> GitHub: jim60105/docker-whisperX
>
> ---
> <https://github.com/jim60105/docker-whisperX>

```bash
docker run --gpus all -it -v ".:/app" ghcr.io/jim60105/whisperx:base-en     -- --output_format srt audio.mp3
docker run --gpus all -it -v ".:/app" ghcr.io/jim60105/whisperx:large-v2-ja -- --output_format srt audio.mp3
docker run --gpus all -it -v ".:/app" ghcr.io/jim60105/whisperx:no_model    -- --model tiny --language en --output_format srt audio.mp3
```

### 模型列表

WhisperX 背後是使用 fast-whisper 做轉錄，請在 [Hugging Face](https://huggingface.co/Systran) 上查看模型清單

### 使用

印出所有的參數和說明，也可以[在 GitHub 上查看](https://github.com/m-bain/whisperX/blob/942c336b8f2f7cf9e78a25b5af68fbff63f29a62/whisperx/transcribe.py#L20)

```bash
whisperx -h
```

基本的使用方式是直接在程式後帶入音檔檔名

```bash
whisperx audio.mp3
```

其它語言需要帶入參數指令語言。  
除了英語以外的其它語言，**模型建議要使用到 large** 成果才比較能看

```bash
whisperx --model large-v3 --language zh audio_zh.mp3
```

轉錄 **中文** 和 **日文** 時<span class="danger">一定要加上參數 `--chunk_size`</span>，5\~10 都不錯，如果斷句很怪的話在此範圍調調看。具體用法可以查看[這支我提交的 PR](https://github.com/m-bain/whisperX/pull/445)  
(預設值是驚人的 30)

```bash
whisperx --chunk_size 6 audio.mp3
```

如果你很不幸的想要在 CPU 上執行它，請加上 `--device cpu --compute_type int8`

```bash
whisperx --device cpu --compute_type int8 audio.mp3
```

若要啟用講者分類功能，請在 `--hf_token` 參數後面帶入你的 Hugging Face 存取金鑰，可以從[這裡](https://huggingface.co/settings/tokens)取得。  
並且需要接受以下兩個模型的使用者協議：[pyannote/segmentation-3.0](https://huggingface.co/pyannote/segmentation-3.0)、[pyannote/speaker-diarization-3.1](https://huggingface.co/pyannote/speaker-diarization-3.1)  
(登入後在頁面上填寫 Company/Website，然後按 Agree)  
使用起來會像這樣

```bash
whisperx --diarize --hf_token hf_OOOOOOOOOOOOOOOOOOOOOOOOOOOOOOO audio.mp3
```

`--output_format` 可指定輸出格式，可選格式有 "all", "srt", "vtt", "txt", "tsv", "json", "aud"

```bash
whisperx --output_format srt audio.mp3
```
