+++
title = "[個人專案] OBS Weather Widget - 可愛風 OBS 天氣小工具"
description = "一個適用於 OBS Studio 瀏覽器來源的可愛風格天氣小工具，顯示明日天氣預報並支援多個地點輪播。專案使用純靜態網頁技術打造，搭配 Open-Meteo 免費 API 取得天氣資料，具備使用者友善的設定頁面與即時預覽功能。採 AGPL-3.0 開源授權，自由軟體，無廣告。"
date = "2026-01-11T12:18:52.075Z"
updated = "2026-01-14T07:31:02.535Z"

[taxonomies]
tags = [ "Cloudflare", "JavaScript", "Livestream", "VTuber" ]
licenses = [ "GFDL 1.3" ]

[extra]
card = "preview.png"
hot = true

  [extra.comments]
  id = "ahdgzwaup3ek008j"

  [extra.preview]
  withAI = true
  url = "https://gemini.google.com/share/c28d28bc3ffa"
  description = "Made with Nano Banana Pro by Gemini 3"
+++
<iframe src="https://weather.obs.xn--jgy.tw/widget?locations=%5B%7B%22name%22%3A%22%E5%8F%B0%E5%8C%97%22%2C%22lat%22%3A25.0531%2C%22lon%22%3A121.5264%7D%2C%7B%22name%22%3A%22%E5%8F%B0%E4%B8%AD%22%2C%22lat%22%3A24.1469%2C%22lon%22%3A120.6839%7D%2C%7B%22name%22%3A%22%E5%8F%B0%E5%8D%97%22%2C%22lat%22%3A22.9908%2C%22lon%22%3A120.2133%7D%5D&interval=3000&unit=celsius" title="OBS Weather Widget Preview" width="500" height="300" style="border:none; overflow:hidden;" loading="lazy" referrerpolicy="no-referrer-when-downgrade" style="width: 500px" ></iframe>

> OBS Weather Widget - 可愛風 OBS 天氣小工具  
> <https://weather.obs.琳.tw/>

一個適用於 OBS Studio 瀏覽器來源的可愛風格天氣小工具，顯示明日天氣預報並支援多個地點輪播。

## 起源

這個專案的緣由要從上次看 [阿奈奈](https://www.youtube.com/@anainainainai) 的直播說起。

昨天阿奈在直播中提到想要買一個能在 OBS 上顯示天氣的小工具。我聽到之後跟她說，實作這個其實很簡單，於是這小專案就誕生了。核心功能大概花了兩個小時，剩下的時間都花在美術素材和介面調整。

<iframe class="youtube-embed" src="https://www.youtube.com/embed/TUx0XUa4Smw?si=bAbZiG-jSWB3psjh&amp;clip=Ugkxj-LnOl22BIRSc8Eo9BDHUfFZx6VTSOqW&amp;clipt=EKaPkAMYhuSTAw" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

{% chat(speaker="anainai") %} 我想要一個天氣的啊，天氣的很酷誒，你不覺得嗎 {% end %}

{% chat(speaker="jim") %} 天氣的到底誰會需要 {% end %}

{% chat(speaker="anainai") %}
可是...可是你們不覺得很酷嗎！...可是很酷誒！

你想你想你想你想你想，在看主播的時候旁邊還可以有一個天氣預報，而且主播常常都開台到晚上十二點，你就可以看明天的天氣耶，不覺得很酷嗎？
{% end %}

{% chat(speaker="anainai") %} ...不酷嗎？ {% end %}

{% chat(speaker="jim") %} 太酷了吧 {% end %}

<figure>
{{ image(url="usage.png", alt="阿奈奈在直播中使用可愛風 OBS 天氣小工具") }}
<figcaption>阿奈奈在直播中使用可愛風 OBS 天氣小工具</figcaption>
</figure>

### 關於美術素材

這個專案的圖像素材是[與 Nano Banana Pro by Gemini 3 協作完成](https://gemini.google.com/share/0ccfcfe3dce7)。

Gemini 3 幫我產生了可愛風格的天氣圖示和小工具設計。在裁切圖示的時候，由於使用美術軟體做去背圖檔超出我跟 AI 的能力範圍，是請阿奈幫忙處理的。

<figure>
{{ image(url="art-style-reference.png", alt="Gemini 設計的美術風格參考") }}
<figcaption>Gemini 設計的美術風格參考</figcaption>
</figure>

<figure>
{{ image(url="icon-reference.png", alt="Gemini 產生的圖示") }}
<figcaption>Gemini 產生的圖示</figcaption>
</figure>

<!-- more -->

## 功能特色

### 明日天氣預報

小工具會顯示 **「明日」** 的天氣預報，包含天氣狀態圖示、最高溫度、最低溫度與天氣描述。

<figure>
{{ video(url="preview.webm", alt="可愛風 OBS 天氣小工具預覽影片", controls=true, loop=true, muted=true, autoplay=true) }}
<figcaption>可愛風 OBS 天氣小工具預覽影片</figcaption>
</figure>

### 多地點輪播

{{cg(body="支援設定多個地點，自動循環輪播顯示。")}}循環間隔時間可以自訂，預設為 5 秒。

### 透明背景設計

{{cg(body="小工具採用透明背景設計，放入 OBS 瀏覽器來源後不會遮擋到直播畫面。")}}配合可愛的手繪風格，讓天氣資訊成為直播畫面的一部分而非干擾元素。

### 友善的設定介面

{{cg(body="專案提供了完整的設定頁面")}}，使用者可以透過地名搜尋功能找到想要的地點，自訂每個地點的顯示名稱，並在內嵌的 iframe 中即時預覽小工具效果。設定完成後只需一鍵複製產生的 Widget URL，再貼到 OBS 的瀏覽器來源即可。

## 使用方式

### 步驟一：設定地點

1. 開啟設定頁面：<https://weather.obs.琳.tw/>
2. 在「地點搜尋」區塊輸入想要的地名（例如：台北、東京）
3. 從搜尋結果中選擇正確的地點
4. 調整顯示名稱後按「新增到清單」
5. 可重複上述步驟新增多個地點
   {{ image(url="setup-page.png", alt="設定頁面截圖") }}
6. 複製產生後的 Widget URL
   {{ image(url="widget-url.png", alt="設定頁面截圖") }}

### 步驟三：加入 OBS

1. 在 OBS Studio 中新增一個「瀏覽器來源」
2. 將設定頁面產生的 Widget URL 貼入 URL 欄位
3. 設定寬度為 **500**、高度為 **300**
4. 完成，小工具會以透明背景顯示在直播畫面上

{{ image(url="obs-setup.png", alt="OBS 設定截圖", no_hover=true) }}

## 技術細節

### 技術堆疊

這個專案採用純靜態網頁技術開發，前端使用純 **HTML5** 搭配 **Tailwind CSS** 快速打造現代化介面，以 **Vanilla JavaScript (ES6+)** 實作。字型方面使用 [`UoqMunThenKhung`](https://fonts.google.com/specimen/UoqMunThenKhung) 營造可愛氛圍。

### 天氣資料來源

本專案使用以下 API：

| API      | Endpoint                                     | 用途             |
| -------- | -------------------------------------------- | ---------------- |
| 天氣預報 | `https://api.open-meteo.com/v1/forecast`     | 取得每日天氣資料 |
| 地理編碼 | `https://nominatim.openstreetmap.org/search` | 將地名轉換為座標 |

### WMO 天氣代碼

小工具根據 WMO（世界氣象組織）天氣代碼來選擇對應的圖示：

| 代碼       | 狀態說明 |
| ---------- | -------- |
| 0          | 晴朗溫暖 |
| 1, 2, 3    | 多雲時晴 |
| 45, 48     | 霧氣朦朧 |
| 51, 53, 55 | 細雨綿綿 |
| 61, 63, 65 | 輕柔降雨 |
| 71, 73, 75 | 浪漫飄雪 |
| 80, 81, 82 | 陣雨來訪 |
| 85, 86     | 陣雪飄落 |
| 95, 96, 99 | 雷雨活力 |

### URL 參數格式

Widget 頁面透過 URL 參數接收設定：

| 參數      | 類型   | 說明                            |
| --------- | ------ | ------------------------------- |
| locations | JSON   | 地點陣列，包含 name、lat、lon   |
| interval  | number | 輪播間隔（毫秒），預設 5000     |
| unit      | string | 溫度單位：celsius 或 fahrenheit |

OBS 和使用者慣用的瀏覧器是兩個獨立的環境。透過 URL 參數傳遞設定資料，{{ cg(body="使得使用者可以在慣用瀏覧器中搞定所有設定，然後再將產生的 URL 貼到 OBS 中使用") }}。

## 開源授權

本專案採用 **[AGPL-3.0](https://github.com/jim60105/obs-weather-widget/blob/master/LICENSE)** 授權開源。程式碼完整公開在 GitHub 上，歡迎自由使用、修改與分享。

若讀者對專案有任何建議或發現問題，歡迎到 [GitHub](https://github.com/jim60105/obs-weather-widget) 提出 Issue，或由以下 Fediverse 回覆我。

> [!TIP]
>
> 延伸閱讀
>
> - [打造 VTuber 須多夜花官方網站：Zola SSG 與 AI 協作開發實錄](@/SideProject/suda-yoruka-official-website-zola-ssg/index.md)
> - [\[個人專案\] SoundButtons - 聲音按鈕](@/SideProject/soundbuttons/index.md)
> - [\[個人專案\] Unfair Spin Wheel - 不公平轉盤](@/SideProject/unfair-spin-wheel/index.md)
> - [影片下載轉檔筆記 (Youtube、Twitch、TwitCasting、Twitter Spaces 音訊空間、fc2 live、ffmpeg、yt-dlp)](@/Livestream/youtube-download-ytdlp-ffmpeg/index.md)
