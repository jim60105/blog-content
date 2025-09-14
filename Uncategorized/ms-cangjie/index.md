+++
title = "微軟倉頡輸入法更換自訂碼表"
description = "微軟倉頡輸入法更換自訂碼表"
date = "2025-09-14T15:23:19.922Z"
updated = "2025-09-14T15:23:17.274Z"
draft = true
aliases = [ ]

[taxonomies]
tags = [ ]
licenses = [ "GFDL 1.3" ]
+++
我個人日常使用大新倉頡碼表，搭配 RIME 中州韻輸入法。最近換新工作，來到一間資安管控較嚴格的大公司，因為資安規範禁止在公務筆電中自行安裝軟體。不誇張，這是我進公司後遇到的第一個難關！我沒辦法打中文字！

山不轉路轉。由於規範是不能安裝軟體，我轉而研究能不能「改造」一下微軟內建的倉頡輸入法。再不行的話就只能路不轉人轉——練習微軟新倉頡了🫠...好家在最後有成功，將微軟倉頡碼表替換為了自訂碼表。

本文涵蓋講解如何寫自己的碼表，烤製成 lex 檔案，並替換系統內建的碼表。後面也會提供我自己改好的大新倉頡碼表，以及社群製作的倉頡三代、五代的碼表，讓讀者直接替換即可使用。
<!-- more -->

我們會依照以下步驟進行:

1. 準備符合格式的碼表
2. 運行碼表轉換程式
3. 替換至系統內

## 準備符合格式的碼表

首先，準備一份用來烤製的碼表，其格式如下:

- UTF-8 無 BOM 表頭
- 一行一個字
- 左邊是倉頡編碼、右邊是中文字，以製表符 <kbd>tab</kbd> 或半型空格 <kbd> </kbd> 分開
- 中文字不能是詞組，意即一次只能出一個字
- 倉頡編碼僅能包含 `a-z`，長度六碼之下

這裡有一份範本碼表供參考: [MicrosoftCangjieTool/cj5_sample.txt at master · Arthurmcarthur/MicrosoftCangjieTool](https://github.com/Arthurmcarthur/MicrosoftCangjieTool/blob/master/cj5_sample.txt)

又或者，你也可以下載我改好的大新倉頡碼表: [Newcj_rime/newcj.txt at master · jim60105/Newcj_rime](https://github.com/jim60105/Newcj_rime/blob/master/newcj.txt)

## 運行碼表轉換程式 Arthurmcarthur/MicrosoftCangjieTool

{% alert(note=true) %}
由於公司資安規範不該運行這類程式，所以我是在家裡的電腦上完成這個步驟，然後再把產出的 lex 檔案傳到公務電腦替換。轉出的 lex 檔案中真的就只是碼表資料，[開源程式碼在此](https://github.com/Arthurmcarthur/MicrosoftCangjieTool/blob/fb4efdc0ac5e01a03c09ba285f780fc2188e565f/src/MSCJTable.cpp#L202)。
{% end %}

<figure>
{{ image(url="MSCJTool.png", alt="MSCJTool zip file", no_hover=true) }}
<figcaption>MSCJTool 的壓縮檔案</figcaption>
</figure>

在 [Arthurmcarthur/MicrosoftCangjieTool](https://github.com/Arthurmcarthur/MicrosoftCangjieTool/releases/latest) 下載最新版本的工具，並解壓縮。

<figure>
{{ image(url="MSCJTool2.png", alt="MSCJTool.exe", no_hover=true, transparent=true) }}
<figcaption>MSCJTool.exe</figcaption>
</figure>

運行 `MSCJTool.exe`，選擇剛剛準備好的碼表檔案，並按下 `烤製碼表`。產出的檔案會在同一個資料夾中，檔名為 `ChtChangjieExt.lex`。

## 替換至系統內

若你現在正在使用微軟倉頡，或開機後啟用過微軟倉頡，請你切換到其他輸入法。接著在工作管理員找到你看到的所有的 `Microsoft IME` 處理程序，並把它們強制結束。這是為了防止系統鎖定檔案，導致無法移除。

{% alert(important=true) %}
請注意下面有 ChtC{{ cr(body="h") }}angjie 和 ChtCangjie 兩種檔名，注意分辨。
{% end %}

Windows 10 2004 之後和 Windows 11 使用者，請到 `C:\Windows\System32\zh-hk` 資料夾內，移除 `ChtCangjie.sdc`、`ChtCangjie.spd` 和 `ChtCangjieExt.lex` 文件。再將取得的 `ChtChangjieExt.lex` 重命名為 `ChtCangjieExt.lex` 後複製到資料夾中。

對於 Windows 10 2004 之前的使用者，請參見作者在倉頡之友論壇中的說明: [替換微軟倉頡碼表，盡享倉頡補完計劃的樂趣 - 倉頡論壇 - 康熙字典与倉頡之友](https://www.chinesecj.com/forum/forum.php?mod=redirect&goto=findpost&ptid=194346&pid=208501)

接下來進入微軟倉頡的設置，打開「包含香港增補字符集字元 (HKSCS)」的開關。

上方的「選取要包含的 Unicode 字元集」選項，請務必根據自己的需求改為「包括中日韓統一表意文字擴展區 A」或「包括中日韓統一表意文字擴展區 A 和 B」。若是改為「包括中日韓統一表意文字擴展區 A」，則僅能輸入基本區和擴展 A 區字符，若改為「包括中日韓統一表意文字擴展區 A 和 B」，則能輸入基本區和擴展 ABCDEFG 區的字符。

由於小工具解開的是 Ext 的 lex 文件，所以修改這些選項是必需的，若維持預設設置將無法出字。

## 社群製作的碼表

以下提供三個社群製作的碼表，裡面的 `MSCJData` 為已轉換好的 lex 格式，讀者可以直接下載並替換使用:

1. 倉頡三代 [Arthurmcarthur/Cangjie3-Plus](https://github.com/Arthurmcarthur/Cangjie3-Plus)
2. 倉頡五代 [Jackchows/Cangjie5](https://github.com/Jackchows/Cangjie5)
3. 自由大新倉頡 [jim60105/Newcj_rime](https://github.com/jim60105/Newcj_rime/blob/master/ChtCangjieExt.lex)

## 參考資料

- [替換微軟倉頡碼表，盡享倉頡補完計劃的樂趣 - 倉頡論壇 - 康熙字典与倉頡之友](https://www.chinesecj.com/forum/forum.php?mod=viewthread&tid=194346)
- [Arthurmcarthur/MicrosoftCangjieTool: This tool helps users to use Microsoft Cangjie in a more efficient way.](https://github.com/Arthurmcarthur/MicrosoftCangjieTool)
