+++
title = "微軟倉頡輸入法自訂碼表教學"
description = "在無法安裝第三方輸入法的企業環境中，教你如何將微軟內建倉頡替換為自訂碼表，從碼表格式、轉換工具到系統檔案替換與設定調整的完整步驟。"
date = "2025-09-14T15:23:19.922Z"
updated = "2025-09-14T15:23:17.274Z"
aliases = [ ]

[taxonomies]
tags = [ ]
licenses = [ "GFDL 1.3" ]

[extra]
withAI = "使用 GitHub Copilot 搭配 Claude Sonnet 4 寫作"
+++

我日常使用自由大新倉頡碼表，搭配 RIME 中州韻輸入法。

八月初換新工作，來到一間大公司，嚴格的資安規範禁止在公務筆電中自行安裝軟體。不誇張，這可能是新工作中最痛苦的部份 —— 突然被逼著回頭面對垃圾微軟內建輸入法，那種挫折感真的難以言喻，我打不出中文字！

不過山不轉路轉，路不轉人轉，既然不能安裝軟體，那就想辦法 *調整* 內建工具吧！經過一番研究，真的讓我找到了將內建倉頡輸入法替換碼表的方法。現在就讓我來分享這個解決方案，讓遇到同樣困擾的讀者也能繼續使用熟悉的輸入碼表。

<!-- more -->

## 解決方案概述

核心想法很簡單：既然不能安裝輸入法，那就只換碼表，將微軟倉頡的內建碼表替換成自己習慣的版本。整個過程分為三個主要步驟：

1. **準備碼表檔案** - 將你喜歡的碼表轉換為指定格式
2. **烤製 lex 檔案** - 使用工具將碼表轉換為系統可識別的格式
3. **替換系統檔案** - 將新的碼表檔案放到正確位置

若你想用的是倉頡三代、五代補完計劃，或者是我轉好的自由大新倉頡碼表，也能直接使用社群上傳的 lex 檔案而不需要自己轉換，後面有提供連結。

## 步驟一：準備碼表檔案

第一步是準備一份符合轉換工具要求的碼表檔案。格式規範如下：

- **編碼格式**：UTF-8 無 BOM 表頭
- **檔案結構**：一行一個字，左邊是倉頡編碼、右邊是中文字
- **分隔符號**：使用製表符 <kbd>tab</kbd> 或半型空格 <kbd> </kbd> 分開
- **內容限制**：中文字不能是詞組（一次只能出一個字）
- **編碼限制**：倉頡編碼僅能包含 `a-z`，長度六碼以下

如果不確定格式是否正確，請參考這份範本碼表：[MicrosoftCangjieTool/cj5_sample.txt](https://github.com/Arthurmcarthur/MicrosoftCangjieTool/blob/master/cj5_sample.txt)

## 步驟二：轉換碼表格式

接下來要使用 [MicrosoftCangjieTool](https://github.com/Arthurmcarthur/MicrosoftCangjieTool/releases/latest) 將準備好的碼表轉換為微軟輸入法可以使用的 lex 格式。

{% alert(note=true) %}
若你也面臨資安考量，建議在另一台個人電腦上完成這個步驟，再將產出的 lex 檔案傳到公務電腦使用。轉換後的 lex 檔案只包含碼表資料，可以放心使用。有興趣的讀者請查看[開源程式碼](https://github.com/Arthurmcarthur/MicrosoftCangjieTool/blob/fb4efdc0ac5e01a03c09ba285f780fc2188e565f/src/MSCJTable.cpp#L202)了解轉換過程。
{% end %}

### 下載並使用轉換工具

<figure>
{{ image(url="MSCJTool.png", alt="MSCJTool zip file", no_hover=true) }}
<figcaption>下載 MSCJTool 工具</figcaption>
</figure>

下載最新版本的工具並解壓縮後，你會看到 `MSCJTool.exe` 執行檔。

<figure>
{{ image(url="MSCJTool2.png", alt="MSCJTool.exe", no_hover=true, transparent=true) }}
<figcaption>MSCJTool 轉換介面</figcaption>
</figure>

使用方式很簡單：執行 `MSCJTool.exe`，選擇準備好的碼表檔案，然後按下「烤製碼表」按鈕。轉換完成後會在同一個資料夾中產生 `ChtChangjieExt.lex` 檔案，這就是我們要的目標檔案。

## 步驟三：替換系統檔案

現在要進行最關鍵的步驟——將轉換好的碼表檔案替換到系統中。這個過程需要一些準備工作和細心操作。

### 準備工作

在開始替換之前，先確保微軟倉頡輸入法沒有在運行：

1. 如果目前正在使用微軟倉頡，請先切換到其他輸入法
2. 開啟工作管理員，尋找所有 `Microsoft IME` 處理程序並強制結束

這樣做是為了防止系統鎖定檔案，導致無法順利替換。

### 檔案替換操作

{% alert(important=true) %}
請特別注意檔名的差異：轉換工具產生的檔名是 ChtC{{ cr(body="h") }}angjie，但系統中的檔名是 ChtCangjie。ChtC{{ cr(body="h") }}angjie 是舊系統使用的名稱。
{% end %}

**針對 Windows 10 2004 之後和 Windows 11 使用者：**

1. 進入 `C:\Windows\System32\zh-hk` 資料夾
2. (備份並)移除以下三個檔案：
   - `ChtCangjie.sdc`
   - `ChtCangjie.spd`
   - `ChtCangjieExt.lex`
3. 將轉換工具產生的 `ChtChangjieExt.lex` 重新命名為 `ChtCangjieExt.lex`
4. 將重新命名的檔案複製到該資料夾中

**Windows 10 2004 之前**的使用者請參考原作者在倉頡之友論壇的詳細說明：[替換微軟倉頡碼表教學](https://www.chinesecj.com/forum/forum.php?mod=redirect&goto=findpost&ptid=194346&pid=208501)

### 調整輸入法設定

<figure>
{{ image(url="mssetting.png", alt="Microsoft Settings", no_hover=true, transparent=true) }}
<figcaption>Microsoft Settings</figcaption>
</figure>

檔案替換完成後，還需要調整微軟倉頡的設定：

1. **啟用香港增補字符集**：啟用「包含香港增補字符集字元 (HKSCS)」選項
2. **選擇 Unicode 範圍**：根據需求選擇以下其中一項：
   - 「包括中日韓統一表意文字擴展區 A」：可輸入基本區和擴展 A 區字符
   - 「包括中日韓統一表意文字擴展區 A 和 B」：可輸入基本區和擴展 A-G 區字符

{% alert(important=true) %}
由於替換的是 Ext 格式的 lex 檔案，上述設定調整是必要的。如果維持預設設定，將會無法正常出字。
{% end %}

## 現成的社群碼表資源

如果你不想自己製作碼表，這裡提供三個社群維護的優質碼表選擇。這些專案的 `MSCJData` 資料夾中都有已經轉換好的 lex 格式檔案，可以直接下載使用：

1. **倉頡三代** - [Arthurmcarthur/Cangjie3-Plus](https://github.com/Arthurmcarthur/Cangjie3-Plus)  
   倉頡三代補完計畫實作

2. **倉頡五代** - [Jackchows/Cangjie5](https://github.com/Jackchows/Cangjie5)  
   倉頡五代補完計畫實作

3. **自由大新倉頡** - [jim60105/Newcj_rime](https://github.com/jim60105/Newcj_rime/blob/master/ChtCangjieExt.lex)  
   將自由大新倉頡轉換為微軟倉頡可讀格式:

   - 由於微軟倉頡只支援 a-z
     - 將候選字用的 `;` 轉換為 `x`(難)
     - 將字碼內含有 a-z 之外的條目刪除
       - 捨棄原本用於 `食` 字旁和 `禾` 字旁的 `;`
       - 捨棄其它所有標點 `,` `.` `[` `]` `'` `/`
   - 由於微軟倉頡不支援詞組，將非一單字元的條目刪除

## 總結

透過這個方法，即使在嚴格的企業環境中也能繼續使用習慣的倉頡輸入碼表。雖然步驟看起來複雜，但實際操作並不困難，而且一次設定就能長期使用。

對於同樣面臨安裝輸入法限制困擾的讀者，希望這篇教學能夠幫助大家找回熟悉的打字手感，不要被微軟新倉頡給綁架！😡

---

### 參考資料

- [替換微軟倉頡碼表，盡享倉頡補完計劃的樂趣 - 倉頡論壇](https://www.chinesecj.com/forum/forum.php?mod=viewthread&tid=194346)
- [Arthurmcarthur/MicrosoftCangjieTool - GitHub](https://github.com/Arthurmcarthur/MicrosoftCangjieTool)
