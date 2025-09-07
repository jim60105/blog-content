+++
title = "微軟倉頡輸入法更換自訂碼表"
description = "微軟倉頡輸入法更換自訂碼表"
date = "2025-08-13T13:14:24.702Z"
updated = "2025-08-13T13:14:24.949Z"
draft = true
aliases = [ ]

[taxonomies]
tags = [ ]
licenses = [ "GFDL 1.3" ]
+++
我個人日常使用大新倉頡碼表，搭配 RIME 中州韻輸入法。最近換新工作，來到一間資安管控較嚴格的大公司，因為資安規範禁止在筆電中自由安裝軟體。不誇張，這是我進公司後遇到的第一個難關！我沒辦法打中文字！

山不轉路轉。由於規範是不能安裝軟體，我轉而研究能不能「改造」一下微軟內建的倉頡輸入法。再不行的話就只能路不轉人轉——練新倉頡了🫠...好在最後是有成功，將微軟倉頡碼表替換為了自訂碼表。

本文會講解如何寫自己的碼表，烤製成 lex 檔案，並替換系統內建的碼表。後面也會提供我自己改好的大新倉頡碼表，以及倉頡三代、五代的碼表，讓讀者直接替換即可使用。
<!-- more -->

我們會依照以下步驟進行:

1. 準備符合格式的碼表
2. 運行碼表轉換程式
3. 替換至系統內

## 準備符合格式的碼表

首先，烤製前碼表的格式如下:

- UTF-8 無 BOM 表頭
- 一行一個字
- 左邊是倉頡編碼、右邊是中文字，以製表符 <kbd>tab</kbd> 或半型空格 <kbd> </kbd> 分開
- 中文字不能是詞組，意即一次只能出一個字
- 倉頡編碼僅能包含 `a-z`，長度六碼之下

這裡有一份範本碼表供參考: [MicrosoftCangjieTool/cj5_sample.txt at master · Arthurmcarthur/MicrosoftCangjieTool](https://github.com/Arthurmcarthur/MicrosoftCangjieTool/blob/master/cj5_sample.txt)
